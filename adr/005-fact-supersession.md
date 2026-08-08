# ADR 005: Fact Supersession — Automatic Invalidation of Conflicting Facts

**Status:** Accepted (2026-08-02)  
**Supersedes:** Three-way-inconsistent conflict handling (extraction worker silent drop, API 409 rejection, cross-episode coexistence)

---

## Context

OpenZync stores extracted knowledge as subject-predicate-object facts with a
temporal validity range (`valid_from`..`valid_to`) and a hard-retraction
marker (`invalid_at`).  Facts can arrive from three independent write paths:

1. The **extraction worker** (`extract_facts`) — LLM extraction from
   conversation episodes, persisted via `batch_create_or_skip`.
2. The **API ingest** endpoint (`POST /v1/projects/{project_id}/facts`) —
   business data, persisted via `batch_create` (409 on exclusion-conflict).
3. **Cross-episode extraction** — the same triple extracted from two
   different episodes coexists because the GiST exclusion constraint
   (`uq_facts_temporal_excl`) is scoped per `source_episode_id`.

When a new fact contradicts an existing active fact, today's behaviour is
three-way inconsistent:

| Write path            | Conflict behaviour                       |
|-----------------------|------------------------------------------|
| Extraction worker     | Silently drops the new fact              |
| API ingest            | Rejects the whole batch with 409         |
| Cross-episode         | Both facts silently coexist              |

None of these preserve the intended semantics of a temporal fact store:
the old fact should become history (`valid_to = now`) and the new fact
should take its place (`valid_from = now`), with the full history retained
for as-of queries.

---

## Decision

**All three write paths converge on a single supersession state machine**:
when an incoming fact conflicts with an active fact of the same identity,
close the old fact's validity range and insert the new one in the same
transaction.  Nothing is dropped, nothing 409s, nothing coexists.

The fact lifecycle:

```{mermaid}
stateDiagram-v2
    [*] --> ingested : extraction worker / API ingest / cross-episode
    note right of ingested
        Three independent write paths converge on one state machine
    end note
    ingested --> active
    active --> superseded : successor contradicts it (valid_to closed)
    active --> retracted : no successor (invalid_at set)
    active --> expired : valid_to passes
    superseded --> [*]
    retracted --> [*]
    expired --> [*]
```

### D1 — Deterministic conflict identity

A fact's conflict identity is:

- `(subject_entity_id, predicate, object_entity_id)` when both entity IDs
  resolve, **or**
- the normalized `(subject, predicate, object)` strings otherwise.

Normalization reuses the aggressive canonicalization already used for
entity resolution in `workers/tasks/extract_facts.py::_match_entity`
(lowercase, punctuation stripped, whitespace trimmed) — deliberately no
new normalization is invented, so the extraction pipeline and every write
path agree on what "same triple" means.

The SQL conflict scan matches case-insensitively (`LOWER(...)`) for string
identities and exactly for entity IDs, backed by a new partial btree index
(`ix_facts_active_spo` on `(project_id, subject, predicate, object) WHERE
invalid_at IS NULL`).  The final identity comparison runs in Python with
full normalization so punctuation/whitespace drift is caught.

Matching is **form-flexible** across write paths: an incoming assertion
supersedes every candidate where (a) both sides carry BOTH entity UUIDs
and they are equal — the *entity match*, which preserves disambiguation
of distinct same-named entities — or (b) the normalized SPO strings are
equal and at least one side lacks the fully-resolved entity form — the
*name match*, the cross-form path that lets string (API) and entity
(extraction) writers of the same triple supersede each other.  Advisory
lock keys and in-batch dedup key on the NAME form (never entity UUIDs)
so cross-form writers of the same triple serialize and collapse.

### D2 — Supersession replaces skip / 409 / coexist

| Write path            | Before                     | After                              |
|-----------------------|----------------------------|------------------------------------|
| Extraction worker     | `ON CONFLICT DO NOTHING`   | Supersession (identical content → skip) |
| API ingest            | 409 `IntegrityError`       | 202 with `superseded_count`        |
| Cross-episode         | silent coexistence         | Supersession (one active fact)     |

`FactBatchResponse` gains `superseded_count` so callers can observe the
replacement.  A residual DB-level constraint violation on a genuinely
concurrent duplicate insert still raises (loud, not silent) — see D6.

### D5 — Single-transaction idempotency; supersession is not episode state

- The truncate (`set_valid_to(old, now)`) and the insert
  (`valid_from = now`) always happen in **one transaction**.  The
  invalidation service never commits mid-operation; the caller's existing
  commit point (worker session or request-scoped session) is the single
  atomic commit, so the truncate and insert succeed or fail together.
- A conflicting active fact with **identical content** causes the insert
  to be skipped entirely — no truncation.  This makes ARQ retries
  idempotent: re-running a failed `extract_facts` job finds its own
  previously-inserted facts and skips rather than superseding them.
- The source episode's `enrichment_status` bit is **never** touched by
  supersession.  Supersession is a side effect of ingestion, not an
  episode-state change; `reconcile_enrichment` therefore does not
  re-extract superseded episodes.

### D6 — `fallback_to_postgres` remains out of scope

The knowledge-graph layer supports a `fallback_to_postgres` mode.  This
ADR intentionally does not change graph-backend behaviour: supersession
is decided at the relational `facts` layer.  Phase 2 shipped without
graph-edge sync (the original deferred item 3); Phase 3 closed that gap —
see **ADR 006** for edge invalidation synchronisation.

---

## State machine (single transaction)

```
for each incoming fact:
    identity = resolve_identity(fact)            # D1
  scan: active facts (effective-at now) in (org, project)
        matching identity, SELECT ... FOR UPDATE   # one query per batch
    if any conflict has identical content:
        skip insert entirely                       # idempotent retry
    else:
        for each conflict: set_valid_to(fact, now) # close old range
        insert new fact with valid_from = now      # open new range
after the batch (same transaction, before caller commit):
    emit webhook FACT_SUPERSEDED
      payload {old_fact_id, new_fact_id, triple, project_id, org_id}
    purge context-cache prefix  ctx:{org}:{proj}:*
increment counter  openzync_facts_superseded_total
```

Embed-enqueueing for the new rows remains with each caller (they already
own per-path `job_id`/`trace_id` semantics), so the new fact is embedded
exactly once on every path.

## Effective-at predicate

Supersession closes a fact's range with `valid_to = now`, so every fact
read query must stop treating `valid_to` as "infinite until told
otherwise".  All fact reads apply the effective-at predicate with
`:t = now` (or the requested instant for as-of queries):

```
(invalid_at IS NULL OR invalid_at > :t)
AND (valid_from IS NULL OR valid_from <= :t)
AND (valid_to IS NULL OR valid_to > :t)
```

Applied to: `get_all_active_for_project`, `search_by_vector`,
`search_by_bm25`, `list_by_session`, `get_facts_at_time`
(`:t = timestamp`), `get_facts_in_range` (invalid-at bound uses
`:t = start` so range queries still surface facts superseded mid-range).

## Consequences

### Positive

- History is preserved: as-of queries can reconstruct the fact timeline.
- All three write paths agree; no silent drops, no 409s on conflicts.
- Retries are idempotent by construction.
- Superseded facts no longer leak into search or context assembly.

### Negative

- Conflict detection runs per batch at insert time (one extra indexed
  query with row locks), a small cost on the write path.
- `CREATE INDEX CONCURRENTLY` in the migration requires breaking Alembic's
  implicit transaction (documented in the migration file).

## Deferred items

1. **Semantic LLM contradiction detection** — supersession is currently
   deterministic (same SPO identity).  Detecting *semantic* contradictions
   ("Alice works at Acme" vs "Alice works at Globex") needs an LLM
   judgment step and stays behind a feature flag.
2. **DB-level cross-episode exclusion** — a true exclusion constraint over
   (org, project, triple, range) would make concurrent first-inserts of the
   same triple from different episodes impossible; today that residual race
   is mitigated by `FOR UPDATE` + effective-at re-check, and a deferred
   constraint would remove it entirely.
3. ~~**Graph-edge sync on supersession**~~ — **SHIPPED in Phase 3.**  When a
   superseded fact was materialized as a graph relationship, the edge is
   now invalidated (or kept) per the supersession-event linkage rule, and
   the read side filters effective-at.  See **ADR 006 — Graph-Edge
   Invalidation Sync**.

## References

- `services/fact_invalidation_service.py` — supersession state machine
- `services/graph_edge_sync_service.py` — edge expiry-set computation and sync (Phase 3)
- `repositories/fact_repository.py` — `find_conflicting_active_for_update`,
  `set_valid_to`, effective-at predicate
- `migrations/versions/0041_add_facts_active_spo_partial_index.py`
- `core/events.py` — `EventType.FACT_SUPERSEDED`

# ADR 006: Graph-Edge Invalidation Sync

**Status:** Accepted (2026-08-02)  
**Extends:** ADR 005 — Fact Supersession (deferred item 3, now shipped)

---

## Context

ADR 005 closes the superseded fact's validity range (`valid_to = now`) at
the relational `facts` layer, but the knowledge graph is a separate,
pluggable layer: Postgres, SurrealDB, and FalkorDB backends each
materialize facts as directed edges between entity nodes.  When a fact is
superseded, edges derived from it must be invalidated too, or traversal
returns temporally stale relationships ("Alice works at Acme" remains
traversable after "Alice works at Globex" supersedes it).

Edges are triple-keyed `(source_id, target_id, relationship_type)`,
merged across facts: `create_relationship` upserts on that key with a
partial unique index `WHERE invalid_at IS NULL`, so at most **one active
edge per triple** exists at any instant.  A fact yields an edge only when
BOTH subject and object entity IDs resolve — literal-subject facts never
create edges.

---

## Decision

Synchronise supersession-driven edge invalidation into all three graph
backends at write time, with a periodic reconciliation cron as the safety
net and backfill.

### D1 — Linkage: derive the expire set from supersession events

Expiry is computed at write time from the supersession events recorded
during ingest (`compute_expiry_commands` in the sync service), applying
the D1 rule per old-fact → successor transition:

- **No successor** (retraction) — expire the edge.
- **Successor with a DIFFERENT edge key** (entity flip-flop: same triple,
  different resolved entities) — expire the old edge.
- **Successor with the SAME edge key** — keep the edge; the successor
  re-asserts it.

A superseded fact whose key never had an edge (literal subject/object,
`old_edge_key is None`) expires nothing.  Retraction always expires —
`new_fact_id` is `None`, so no successor can re-assert.

### D2 — API: expire-and-retrieve contracts on the GraphBackend ABC

- `expire_relationships_matching(org_id, project_id, *, source_id,
  target_id, relationship_type, at_time) -> int` on the `GraphBackend`
  ABC (`packages/graph_backend/interface.py`) — sets
  `invalid_at = at_time` on every **active** (`invalid_at IS NULL`) edge
  matching the triple, and only that field: `valid_to` is never touched.
  The count return makes replay idempotent (replay → 0).
- `retrieve_graph(..., as_of: datetime | None = None)` — effective-at
  edge filtering on the read side; `None` = now.

`at_time` is the deterministic fact-supersession instant, never a fresh
clock read, so edge and fact invalidation agree.

### D3 — Consistency: in-transaction vs after-commit routing

| Backend        | Mechanism                                                              |
|----------------|------------------------------------------------------------------------|
| Postgres       | `expire_relationships_matching` runs **in-transaction** on the caller's session — atomic with the fact truncate+insert. |
| Surreal/Falkor | Post-commit ARQ job `expire_graph_edges` on the low-priority queue; resolves the org's backend inside the worker. |

The supersession-driven edge invalidation flow:

```{mermaid}
sequenceDiagram
    participant Ingest as Ingest (compute_expiry_commands)
    participant Sync as GraphEdgeSyncService
    participant PG as PostgreSQL
    participant ARQ as ARQ (expire_graph_edges, low-priority)
    participant Backend as SurrealDB / FalkorDB
    participant Cron as Reconcile cron (5 min, batch 200)

    Ingest->>Sync: FACT_SUPERSEDED (old_fact_id, new_fact_id, triple)
    Sync->>PG: expire_relationships_matching (in-transaction, atomic with fact commit)
    Note over Sync,Backend: Post-commit routing for external backends
    Sync->>ARQ: enqueue expire_graph_edges
    loop retry up to 3
        ARQ->>Backend: expire_relationships_matching (invalid_at = at_time)
    end
    alt final failure
        ARQ->>ARQ: openzync_graph_edge_sync_failures_total + loud log
    end
    Cron->>PG: anti-join (active edges with no active fact)
    Cron->>ARQ: enqueue expire_graph_edges per stale edge
```

The `expire_graph_edges` task retries ×3 (`@with_retry(max_retries=3)`),
increments `openzync_graph_edge_sync_failures_total` and re-raises on
final failure — facts are the source of truth and are never rolled back;
a loud log names the failure for operator attention.

The `reconcile_graph_edges` cron runs every 5 minutes on the
low-priority worker: a Postgres anti-join finds active edges
(`invalid_at IS NULL`) with **no** active fact asserting the same edge
key (`subject_entity_id = source_id`, `predicate =
relationship_type`, `object_entity_id = target_id`, `valid_to IS NULL`)
and enqueues one `expire_graph_edges` job per stale edge
(batch-limited to 200 per tick).  This is the safety net AND the
backfill: it self-heals any drift the post-commit sync missed (worker
crash, enqueue loss, pre-existing rows from before Phase 3).

### D4 — Read side: effective-at edge filtering

`retrieve_graph` and `traverse` apply the effective-at edge predicate
with `:as_of` (defaults to now when `None`):

```
(invalid_at IS NULL OR invalid_at > :as_of)
AND (valid_from IS NULL OR valid_from <= :as_of)
AND (valid_to IS NULL OR valid_to >= :as_of)
```

SurrealDB `traverse` previously ignored edge validity entirely (arrow
syntax with no `WHERE` on the relation) — fixed in Phase 3 to apply the
same predicate.  `list_entity_edges` filters `invalid_at IS NULL` on
all three backends.

### D5 — Backfill: skipped by design

No explicit backfill migration ships: the `reconcile_graph_edges` cron
anti-join self-heals drift in bounded batches without replaying the fact
commit.

## Cross-form note

String-identity facts supersede entity-resolved facts via the ADR 005
name-match path (normalized SPO names; entity-match precedence only when
both sides carry both UUIDs; advisory locks keyed on the name form).
This is what makes supersession-driven edge expiry reachable from the
API path: a `POST /facts` string write supersedes an extraction-derived
entity fact, so the same supersession state machine — and therefore the
same edge-expiry derivation — applies to business-data ingest.

## Consequences

### Positive

- Traversal is temporally correct across all three backends.
- Postgres sync is atomic with the fact transaction; external backends
  are eventually consistent with a bounded 5-minute self-heal window.
- The cron doubles as the backfill, so no migration-time backfill
  required.

### Negative

- External backends carry a small eventual-consistency window between
  the fact commit and the ARQ job's expiry.
- The cron scans the `graph_relationships` anti-join every 5 minutes
  (indexed, batch-limited, but a recurring cost on large graphs).

## Deferred items

1. **Semantic LLM contradiction detection** — deterministic SPO
   supersession only; semantic contradictions stay behind a feature flag
   (as in ADR 005).
2. **DB-level cross-episode exclusion** — residual concurrent
   first-insert race mitigated by advisory locks + effective-at re-check;
   a deferred exclusion constraint would remove it entirely.
3. **`merge_duplicate_entities` edge-key drift** — entity merging rewires
   edges; the rewired edge keys may drift from the facts that assert
   them (the reconcile cron catches the fact-side drift, but merge-driven
   rewiring is not yet supersession-driven).
4. **API/business-data ingest never creates edges** — pre-existing gap,
   tracked separately; when it lands, it must also emit supersession
   events for edge sync (the cross-form note above already routes through
   the same state machine).

## References

- `services/graph_edge_sync_service.py` — `compute_expiry_commands` (D1),
  `GraphEdgeSyncService.sync_supersessions` (D3 routing)
- `services/fact_invalidation_service.py` — supersession + post-commit
  edge-sync effect
- `packages/graph_backend/interface.py` — `expire_relationships_matching`,
  `retrieve_graph(as_of=...)`
- `packages/graph_backend/{postgres,surrealdb,falkordb}.py` — per-backend
  expiry + effective-at read filter (D4)
- `workers/tasks/expire_graph_edges.py` — ARQ task (retry ×3, metric,
  loud log)
- `workers/tasks/reconcile_graph_edges.py` — 5-minute cron anti-join
- `services/worker/worker.py` — cron registration

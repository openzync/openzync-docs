.. _frontend:

===============
OpenZync Frontend
===============

The OpenZync frontend is a `Next.js 16 <https://nextjs.org/>`_ application that
provides the administrative dashboard and knowledge graph explorer for the
OpenZync agent memory infrastructure platform.

It is a **client-side rendered** (``"use client"``) single-page application
styled with `Tailwind CSS v4 <https://tailwindcss.com/>`_ and `Radix UI
<https://www.radix-ui.com/>`_ primitives, deployed behind an nginx reverse
proxy that also routes API calls to the backend.

.. contents:: On this page
   :local:
   :depth: 3
   :class: this-will-duplicate-information-and-it-is-still-useful-here

--------------
Tech Stack
--------------

.. list-table::
   :header-rows: 1

   * - Layer
     - Technology
   * - Framework
     - `Next.js 16.2.9 <https://nextjs.org/>`_ (App Router, Standalone output)
   * - Language
     - TypeScript 5.x (strict mode)
   * - Styling
     - `Tailwind CSS v4 <https://tailwindcss.com/>`_ + ``tailwind-merge`` + ``clsx``
   * - UI Primitives
     - `Radix UI <https://www.radix-ui.com/>`_ (Dialog, DropdownMenu, Tabs, Toast, Tooltip, etc.)
   * - Icons
     - `Lucide React <https://lucide.dev/>`_ v1.18
   * - Graph Visualization
     - `D3.js v7 <https://d3js.org/>`_ (force simulation, convex hulls)
   * - Notifications
     - `Sonner <https://sonner.emilkowal.ski/>`_ v2
   * - Fonts
     - Inter (sans-serif), JetBrains Mono (monospace)
   * - Font Loading
     - ``next/font`` (Google Fonts via Next.js)
   * - Package Manager
     - npm (``package-lock.json``)
   * - Container
     - Docker (multi-stage Node.js 20 Alpine build)
   * - CI/CD
     - GitHub Actions (``.github/workflows/deploy.yml``)

-----------------------
Application Architecture
-----------------------

Route Pattern
==============

The application uses Next.js **App Router** with the following top-level route
groups:

.. code-block:: text

   src/app/
   ├── page.tsx                        # / → redirects to /overview
   ├── layout.tsx                      # Root layout (fonts, ThemeProvider)
   ├── globals.css                     # Tailwind v4 + design tokens
   │
   ├── login/                          # Public: password login
   ├── login/otp/                      # Public: OTP (magic code) login
   ├── login/mfa/                      # Public: MFA challenge
   ├── signup/                         # Public: registration
   ├── verify-email/                   # Public: email verification
   ├── forgot-password/                # Public: password reset request
   ├── reset-password/                 # Public: password reset (OTP + new pw)
   ├── onboarding/                     # Post-registration setup wizard
   │
   └── (dashboard)/                    # Authenticated layout group
       ├── layout.tsx                  # Dashboard shell (sidebar, topbar)
       ├── require-auth.tsx            # Auth guard component
       ├── overview/                   # Org overview / home
       ├── analytics/                  # Usage analytics + charts
       ├── monitoring/                 # Prometheus-based system monitoring
       ├── monitoring/query/           # Ad-hoc PromQL query tool
       ├── projects/                   # Project list
       ├── projects/[id]/              # Project-scoped routes
       ├── users/                      # User management list
       ├── users/[id]/                 # User detail (summary, instructions)
       ├── audit/                      # Audit log (paginated, filterable)
       └── settings/                   # System settings pages
           ├── schemas/                # Extraction schemas
           ├── classifications/        # Classification schemas
           ├── extractions/            # Extraction viewer
           ├── extraction-instructions/# Extraction instructions
           ├── webhooks/               # Webhook endpoints
           ├── prompts/                # Prompt template manager
           └── org-config/             # Org configuration (LLM, graph, etc.)

Layout Hierarchy
=================

.. code-block:: text

   RootLayout (html, fonts, ThemeProvider)
   ├── Public pages (login, signup, etc.) — no sidebar
   └── DashboardLayout (RequireAuth guard)
       ├── Sidebar (collapsible, role-aware)
       ├── TopBar (page title, search, theme toggle, user menu)
       ├── Breadcrumb
       ├── ProjectLayout (optional, wraps [id] routes)
       │   └── ProjectProvider (fetches project metadata)
       └── Page content

- **RootLayout** (``src/app/layout.tsx``): Loads Inter + JetBrains Mono via
  ``next/font``, wraps children in a ``ThemeProvider`` (``next-themes``,
  forced dark mode), and adds skip-navigation links for accessibility.

- **DashboardLayout** (``src/app/(dashboard)/layout.tsx``): All authenticated
  pages live inside this layout group. It renders the collapsible sidebar,
  top bar with theme/user menu, breadcrumb trail, and the ``main`` content
  area. The layout is guarded by ``<RequireAuth>``.

- **ProjectLayout** (``src/app/(dashboard)/projects/[id]/layout.tsx``):
  Wraps project-scoped pages with ``<ProjectProvider>`` which fetches
  project metadata from the API and makes it available via
  ``useProject()``.

- **OrgConfigLayout** (``src/app/(dashboard)/settings/org-config/layout.tsx``):
  Adds a tabbed sub-navigation (LLM / Embeddings / Graph / Behaviour) for the
  organization configuration section.

--------------
Design System
--------------

The design system is defined in ``src/app/globals.css`` using Tailwind CSS v4's
``@theme`` directive.

Theme Tokens
=============

.. list-table::
   :header-rows: 1

   * - Token Family
     - Example Values
   * - ``--color-brand-*``
     - ``500: #14488C`` (deep blue — primary action colour)
   * - ``--color-accent-*``
     - ``300: #8FAFD9`` (light blue — secondary accent)
   * - ``--color-surface-*``
     - ``900: #161B22`` / ``950: #0D1117`` (dark backgrounds)
   * - ``--color-success``
     - ``#66BB6A``
   * - ``--color-warning``
     - ``#FFA726``
   * - ``--color-error``
     - ``#EF5350``
   * - ``--color-text-primary``
     - ``#F2F2F2``
   * - ``--font-sans``
     - Inter
   * - ``--font-mono``
     - JetBrains Mono

Default Theme
==============

The application defaults to **dark mode** (``defaultTheme="dark"``,
``enableSystem={false}``). Users can toggle between dark and light via the
sun/moon icon in the top bar.

Reusable CSS Patterns
======================

The ``globals.css`` file defines several utility classes used throughout the
application:

- ``.card-base`` — Standard card container (rounded border, surface-900 bg)
- ``.card-interactive`` — Clickable card with hover border/shadow transition
- ``.stat-card`` — KPI stat card with hover elevation
- ``.input-base`` — Unified input/select/textarea styling
- ``.animate-fade-in``, ``.animate-slide-up`` — Common animations

Component Library
==================

UI Primitives (``src/components/ui/``)
-----------------------------------------

.. list-table::
   :header-rows: 1

   * - Component
     - Description
   * - ``Button``
     - Variants: ``primary``, ``secondary``, ``ghost``, ``danger``; sizes: ``sm``, ``md``, ``lg``; loading state with ``Spinner``
   * - ``Badge`` / ``StatusBadge`` / ``ActorTypeBadge``
     - Status badges with colour variants (success, warning, error, info, brand) + HTTP status code and actor type helpers
   * - ``Dialog``
     - Radix-based modal dialog with close button
   * - ``Spinner``
     - Inline SVG animated spinner
   * - ``Switch``
     - Toggle switch (Radix-based)
   * - ``SecretInput``
     - Password-style input with eye toggle for API key fields

Shared Components (``src/components/shared/``)
-------------------------------------------------

.. list-table::
   :header-rows: 1

   * - Component
     - Description
   * - ``PageHeader``
     - Standardised page title + optional description + action buttons
   * - ``StatCard``
     - KPI card with icon, value, loading skeleton, and trend indicator
   * - ``EmptyState``
     - Centered icon + title + description + optional action
   * - ``ErrorState``
     - Inline error banner with optional retry button
   * - ``ConfirmDialog``
     - Modal confirmation dialog (Escape to close, loading state)
   * - ``AuthLoadingScreen``
     - Full-page branded loading overlay during auth check
   * - ``TableSkeleton`` / ``Skeleton``
     - Animated placeholder rows for loading tables

-----------------------------------
API Client & Authentication
-----------------------------------

Centralized API Client (``src/lib/api-client.ts``)
=====================================================

Every API call in the frontend routes through a single module located at
``src/lib/api-client.ts``. It provides:

- **Base URL**: Reads ``NEXT_PUBLIC_API_URL`` from environment, defaults to
  ``http://localhost:8000``.
- **Auth Headers**: Automatically injects ``Authorization: Bearer <token>``
  from ``sessionStorage`` (key ``mg_access_token``).
- **Token Refresh**: On a 401 response, the client attempts a transparent
  token refresh by calling ``POST /v1/auth/refresh`` with the stored refresh
  token (key ``mg_refresh_token``). If refresh succeeds, the original request
  is retried once. If it fails, tokens are cleared and the user is redirected
  to ``/login?reason=not-signed-in``.
- **Typed Helpers**: Exports ``get<T>()``, ``post<T>()``, ``put<T>()``,
  ``patch<T>()``, ``del<T>()`` — all generic typed wrappers around the
  internal ``request()`` function.
- **Pagination**: ``CursorPageParams`` / ``OffsetPageParams`` types and an
  ``extractList()`` helper that normalises responses that use ``data``,
  ``items``, or bare arrays.
- **Error Handling**: ``ApiError`` class with convenience getters
  (``isUnauthorized``, ``isNotFound``, ``isRateLimited``, ``isServerError``).

.. code-block:: typescript

   // Example: fetching paginated sessions
   import { get, post, extractList } from "@/lib/api-client";

   const sessions = await get<SessionsResponse>(
     `/v1/projects/${projectId}/sessions?limit=50`,
   );
   const items = extractList<Session>(sessions);

Authentication Flow
====================

The frontend supports **four authentication methods**:

1. **Password Login** (``/login``)
   - POST credentials to ``/v1/auth/login``
   - On success, stores ``access_token`` and ``refresh_token`` in
     ``sessionStorage``
   - If the backend returns ``requires_mfa: true``, redirects to
     ``/login/mfa`` with an ``mfa_session_token``

2. **OTP / Magic Code** (``/login/otp``)
   - Two-step flow: email → OTP receipt → code verification
   - ``POST /v1/auth/login/otp/send`` then ``POST /v1/auth/login/otp/verify``
   - Resend with 60-second cooldown timer

3. **MFA Challenge** (``/login/mfa``)
   - After password authentication when MFA is enabled
   - Requires email + 6-digit code + ``mfa_session_token``
   - Verifies via ``POST /v1/auth/mfa/verify``

4. **Social Login** (button stubs, currently disabled)
   - GitHub and Google buttons rendered but disabled

Session Token Storage
======================

All tokens are stored in **``sessionStorage``** (not ``localStorage``) using
the keys:

- ``mg_access_token`` — JWT access token
- ``mg_refresh_token`` — JWT refresh token

Tokens are cleared on logout, on 401-after-refresh failure, and when the
session expires.

Auth Guard (``require-auth.tsx``)
===================================

The ``<RequireAuth>`` component wraps the entire dashboard layout. On mount it
checks for a valid (non-expired) JWT in ``sessionStorage``. If the token is
missing or expired (checked by decoding the JWT payload and comparing ``exp``
to ``Date.now()``), it clears stale tokens and redirects to
``/login?reason=not-signed-in`` after a brief 500ms display of the loading
screen (for a polished UX).

---------------
User Flows
---------------

Registration & Onboarding
===========================

1. User visits ``/signup``, fills out organization name, email, and password
   (with a client-side password strength meter)
2. ``POST /v1/auth/signup`` → redirects to ``/verify-email?email=...``
3. User enters 6-digit OTP received by email → ``POST /v1/auth/verify-email``
   → tokens stored → redirected to ``/onboarding``
4. **Onboarding wizard** (``/onboarding``): A single-page setup form
   covering:
   - **LLM Configuration**: Backend (OpenAI, Anthropic, OpenRouter, Azure,
     Ollama, OpenAI-compatible), model name, temperature, max tokens
   - **API Keys**: Secret inputs for OpenAI, Anthropic, OpenRouter, Azure
     keys, plus Azure endpoint and Ollama base URL
   - **Embeddings**: Backend provider, model, dimensions, API key
   - **Knowledge Graph**: Backend (PostgreSQL/pgvector, SurrealDB, or none),
     search type (hybrid, BM25, vector), traversal depth, SurrealDB
     connection fields
   - **Behaviour**: Context cache TTL, audit log response body toggle
5. On save, ``PUT /admin/org/config`` stores the configuration and the user
   is redirected to ``/overview``

Login Flow
===========

1. User enters email/password at ``/login``
2. If MFA is enabled, redirected to ``/login/mfa`` for 6-digit code
3. On success, tokens stored in ``sessionStorage``, redirected to
   ``/overview``
4. Subsequent visits check for valid token via ``RequireAuth``
5. **Token refresh**: If the API returns 401, the client transparently
   attempts ``POST /v1/auth/refresh`` before redirecting to login

Password Reset
===============

1. User enters email at ``/forgot-password`` → ``POST /v1/auth/forgot-password``
2. Email received with reset code → user clicks through to
   ``/reset-password?email=...``
3. User enters 6-digit OTP + new password + confirm password
4. ``POST /v1/auth/reset-password`` → success → auto-redirect to login

Dashboard Overview (``/overview``)
====================================

The landing page after login shows:

- **Stat Cards**: Total Users, Total Sessions, Total Messages, API Keys
  (fetched from ``GET /v1/admin/stats/org``)
- **Quick Actions**: Buttons to ingest memory, create user, new session
- **Recent Activity**: Last 5 audit log entries (``GET /v1/admin/audit-logs?limit=5``)
  with human-readable action labels (e.g. "Session created", "Memory ingested")
- **Daily Usage Summary**: 7-day totals for messages, sessions, episodes

Projects
=========

List (``/projects``)
---------------------

- Fetches all projects via ``GET /v1/projects``
- Displays project cards with name, description, member count, creation date
- **Pin to sidebar**: Up to 3 projects can be pinned via ``localStorage``
  (``usePinnedProjects`` hook). Pinned projects appear in the sidebar for
  quick access.
- **Create Project**: Modal dialog with name + description fields
- Empty, loading, and error states handled with shared components

Project Dashboard (``/projects/[id]/``)
-----------------------------------------

Project-scoped routes are available once inside a project:

- **Sessions** (``/projects/[id]/sessions``): List sessions with pagination,
  create/delete dialogs, status badges (active/closed), message/fact counts
- **Session Detail** (``/projects/[id]/sessions/[sessionId]``): Metadata card
  (ID, creator, timestamps, message/fact counts) + tab navigation for
  Messages, Facts, Graph, Classifications, Extractions
- **Memory** (``/projects/[id]/memory``): Three-tab interface:
  - *Ingest*: Paste messages (format ``role: content`` per line), submit to
    ``POST /v1/projects/{id}/memory``
  - *Context*: Query semantic context via
    ``GET /v1/projects/{id}/context?query=...`` with configurable result limit
  - *Search*: Hybrid search across episodes, facts, and entities with
    type-filter checkboxes and relevance scores
- **Graph Explorer** (``/projects/[id]/graph``): Interactive D3 force graph
  (see :ref:`graph-visualization`)
- **Project Settings** (``/projects/[id]/settings``): Edit name/description,
  archive project
- **API Keys** (``/projects/[id]/settings/api-keys``): Create, view prefix,
  copy raw key (shown once), revoke

Administration
===============

Users (``/users``)
-------------------

- Paginated user list with cursor-based pagination
- Create/Edit/Delete dialogs
- Copy user ID to clipboard
- Navigate to user detail page

User Detail (``/users/[id]``)
-------------------------------

- Profile metadata card (ID, external ID, name, email, role, created date)
- Stats cards (messages, facts, sessions)
- **User Summary**: LLM-generated profile of the user based on conversation
  history. Trigger generation via ``POST /v1/users/{id}/summary`` with
  polling (5s interval, 2min timeout) until ``updated_at`` changes.
- **Summary Instructions**: CRUD for custom instructions that guide summary
  generation. Instructions synced via ``PUT /v1/users/{id}/summary-instructions``.

Settings
=========

System Settings (``/settings``)
--------------------------------

- **Profile**: Edit name and email via ``PATCH /v1/auth/me``
- **Change Password**: Current + new password with client-side validation
- **MFA Toggle**: Enable/disable email-based MFA with password confirmation
  (and OTP for disable)

Administration Pages
---------------------

All under ``/settings/*``:

- **Extraction Schemas** (``/settings/schemas``): CRUD for JSON schema-based
  extraction definitions
- **Classifications** (``/settings/classifications``): View classification schemas
- **Extractions** (``/settings/extractions``): View extracted data by schema
- **Extraction Instructions** (``/settings/extraction-instructions``): Manage
  extraction instructions
- **Webhooks** (``/settings/webhooks``): Create webhook endpoints with event
  type selection, view signing secret once, test delivery
- **Prompts** (``/settings/prompts``): Full prompt template manager with
  version history, edit/customize, set default prompt for type
- **Org Config** (``/settings/org-config``): Tabbed configuration for LLM,
  Embeddings, Graph, and Behaviour (same sections as onboarding)

Monitoring & Analytics
=======================

Analytics (``/analytics``)
----------------------------

- Stat cards for total messages, sessions, facts
- **Daily Usage Chart**: Custom SVG bar chart with 7/30/90-day ranges.
  Renders stacked bars for messages and sessions, with hover tooltip showing
  exact values. Y-axis auto-scales with ``niceMax`` heuristic.

Monitoring (``/monitoring``)
------------------------------

- Real-time platform metrics auto-refreshing every 30 seconds
- **KPI Cards**: Episodes added (24h), enrichment progress bar with
  colour-coded thresholds, error rate, queue depth
- **Latency Panel**: p50/p95/p99 latency cards for overall API, context
  assembly, and graph search (colour-coded: green <100ms, yellow <500ms, red
  500ms+)
- **Scrape Targets Table**: Prometheus scrape targets with health status,
  last scrape time, and error column
- **Status Bar**: Overall system status, active requests, user count, request
  rate (2xx/5xx)
- **Prometheus Query** (``/monitoring/query``): Ad-hoc PromQL query interface

Audit Log (``/audit``)
-------------------------

- Server-side paginated (25 per page) audit log with:
  - Filter by action name, actor type (user/API key/system), status code group
  - Auto-refresh toggle (10s interval)
  - Columns: time, action (monospace), actor ID, actor type badge, status
    badge, HTTP method, path, IP address

.. _graph-visualization:

-----------------------------
Graph Visualization
-----------------------------

The ``ForceGraph`` component (``src/components/force-graph.tsx``) is the
centerpiece of the Graph Explorer page. It is a 1044-line interactive D3.js
force-directed graph.

Types
======

.. code-block:: typescript

   interface GraphNodeData {
     id: string;
     name: string;
     type: string;       // "person", "organization", "location", "event", "concept", "community"
     summary: string | null;
     created_at: string;
   }

   interface GraphEdgeData {
     id: string;
     source_id: string;
     target_id: string;
     type: string;       // e.g. "member_of", "works_at", "located_in"
   }

   interface ForceGraphProps {
     nodes: GraphNodeData[];
     edges: GraphEdgeData[];
     loading?: boolean;
     error?: string | null;
     onRetry?: () => void;
     apiConfig: ApiConfig;           // baseUrl, projectId, auth headers
     userName?: string;
     showFilter?: boolean;
     showControls?: boolean;
     showLegend?: boolean;
     height?: number;
     emptyMessage?: string;
     emptyAction?: React.ReactNode;
   }

Features
=========

1. **D3 Force Simulation**
   - ``forceSimulation`` with ``forceLink`` (distance 120), ``forceManyBody``
     (strength -250), ``forceCenter``, ``forceCollide`` (radius + 8px padding)
   - Node radius proportional to degree (``5 + sqrt(deg) * 4``)
   - Drag-and-drop with fixed-position support (``fx``/``fy``)
   - Zoom/pan with ``d3.zoom`` (scale extent 0.15–5x)
   - Auto zoom-to-fit on initial render

2. **Community Convex Hulls**
   - Entities connected via ``member_of`` edges are grouped into coloured
     convex hull polygons using ``d3.polygonHull``
   - Hulls are expanded by 12px padding for visual breathing room
   - Hovering a hull highlights member nodes and dims non-members
   - Only communities with ≥3 visible members render hulls

3. **Search & Filter**
   - Text filter searches by node name and type
   - Two modes: **Exact** (only matching nodes) and **Related** (matching
     nodes + 1-hop neighbours)
   - Real-time node/edge count display (filtered/total)

4. **Node Selection & Detail Panel**
   - Clicking a node opens a floating info panel
   - Panel fetches node detail from the API
   (``GET /v1/projects/{id}/graph/nodes/{nodeId}``) with metadata and
   relationships
   - Falls back to local data on API error
   - Clicking a neighbour name navigates to that node

5. **Hover Interactions**
   - Hovering a node highlights its connected edges (increased opacity/width)
   - Connected neighbours remain opaque while others dim to 20%
   - Edge labels (relationship type) shown on hover
   - Community hulls highlight on hover

6. **Controls**
   - Zoom in/out/reset buttons
   - Fullscreen toggle (Esc to exit)
   - Entity type legend (Person, Organization, Location, Event, Concept,
     Community) with colour swatches

7. **Empty/Error States**
   - Loading spinner overlay
   - Error state with retry button
   - Empty state with contextual message and optional action

Node Colours
=============

.. list-table::
   :header-rows: 1

   * - Entity Type
     - Colour
     - Hex
   * - person
     - Deep blue
     - ``#14488C``
   * - organization
     - Medium blue
     - ``#1453A6``
   * - location
     - Light blue
     - ``#8FAFD9``
   * - event
     - Navy
     - ``#1747A6``
   * - concept
     - Steel blue
     - ``#6A8DB8``
   * - community
     - Purple
     - ``#7C3AED``

Graph Data Loading (``/projects/[id]/graph/page.tsx``)
=========================================================

The ``GraphExplorerPage`` component:

1. Fetches up to 100 nodes via
   ``GET /v1/projects/{projectId}/graph/nodes?limit=100``
2. Fetches edges for those node IDs via
   ``GET /v1/projects/{projectId}/graph/edges?subject_ids=...&limit=50``
3. Filters out self-loops and edges whose endpoints are not in the node set
4. Passes the data (nodes + edges) to the ``ForceGraph`` component

--------------------
State Management
--------------------

The frontend uses a lightweight state management approach without a global
store library.

React Context
==============

``ProjectContext`` (``src/stores/project-context.tsx``)
---------------------------------------------------------

- Provides ``{ project, loading, error, refetch }`` to all pages nested inside
  ``/projects/[id]/``
- Automatically reads the ``id`` route parameter via ``useParams()``
- Fetches project metadata via ``GET /v1/projects/{id}``
- Handles 404 (sets ``error = "Project not found"``) and other API errors

Custom Hooks
=============

``usePinnedProjects`` (``src/hooks/use-pinned-projects.ts``)
---------------------------------------------------------------

- Manages up to 3 pinned projects stored in ``localStorage``
- Cross-component sync via custom ``StorageEvent``
- Exposes ``{ pinned, togglePin, isPinned, isMaxPinned }``
- Used by the sidebar to render pinned project shortcuts

Local Component State
======================

All other state (form data, dialogs, loading flags, pagination cursors) is
managed with React ``useState`` and ``useReducer`` locally within each page
component. There is no Redux, Zustand, or other global state library.

------------------
Utility Functions
------------------

``src/lib/utils.ts`` provides shared helpers:

- ``cn()`` — Merges Tailwind classes with ``tailwind-merge`` + ``clsx``
- ``timeAgo()`` / ``formatDate()`` / ``smartTimestamp()`` — Date/time
  formatting
- ``truncateId()`` — UUID truncation for display
- ``copyToClipboard()`` — Async clipboard API wrapper
- ``actionLabel()`` — Maps machine action names to human labels (e.g.
  ``"session.create"`` → ``"Session created"``)
- ``formatNumber()`` — Locale-aware number formatting

---------------
Deployment
---------------

Docker
=======

The frontend ships with a **multi-stage Docker build**:

.. code-block:: dockerfile

   # Stage 1: Builder (node:20-alpine)
   COPY package.json package-lock.json ./
   RUN npm ci --legacy-peer-deps
   COPY . .
   RUN npm run build

   # Stage 2: Runner (node:20-alpine)
   COPY --from=builder /app/.next/standalone ./
   COPY --from=builder /app/.next/static ./.next/static
   COPY --from=builder /app/public ./public
   EXPOSE 3000
   HEALTHCHECK --interval=30s --timeout=5s --start-period=10s \
       CMD wget --no-verbose --tries=1 --spider http://127.0.0.1:3000/ || exit 1
   CMD ["node", "server.js"]

Key points:

- Uses ``output: "standalone"`` in ``next.config.ts`` to produce a
  self-contained ``server.js`` with only required ``node_modules``
- Static assets (``.next/static``, ``public/``) copied separately
- The ``NEXT_PUBLIC_API_URL`` build arg defaults to ``""`` (empty string),
  meaning the frontend uses **relative URLs** — API calls like
  ``/v1/sessions`` are proxied through the same nginx origin

Docker Compose
===============

.. code-block:: yaml

   # deploy/docker-compose.yml
   networks:
     default:
       name: openzync_network

   services:
     frontend:
       build:
         context: .
         dockerfile: Dockerfile
       image: ghcr.io/rohnsha0/openzync/frontend:latest
       ports:
         - "3000:3000"
       env_file: .env
       restart: unless-stopped

The compose file is standalone — it does not reference backend services. In
production, the frontend and API are served through a shared nginx reverse
proxy on the same VPS.

CI/CD
======

A GitHub Actions workflow (``.github/workflows/deploy.yml``) builds the Docker
image and pushes it to GitHub Container Registry (``ghcr.io``).

-----------------------
Environment Variables
-----------------------

.. code-block:: bash

   # .env.example
   NEXT_PUBLIC_API_URL=http://localhost:8000

- ``NEXT_PUBLIC_API_URL``: The base URL of the OpenZync API. In development
  this defaults to ``http://localhost:8000``. In production it is typically
  ``""`` (empty), relying on nginx to proxy ``/v1/*`` requests to the API
  backend.

---------------
Project Structure
---------------

.. code-block:: text

   openzync-frontend/
   ├── .env.example              # Environment variable template
   ├── .github/workflows/        # GitHub Actions CI/CD
   ├── deploy/
   │   └── docker-compose.yml    # Standalone deployment
   ├── Dockerfile                # Multi-stage production build
   ├── next.config.ts            # Next.js config (standalone output)
   ├── package.json              # Dependencies & scripts
   ├── postcss.config.mjs        # PostCSS + Tailwind config
   ├── tsconfig.json             # TypeScript config (strict, @/ alias)
   └── src/
       ├── app/                  # Next.js App Router pages
       │   ├── globals.css       # Design tokens, base styles, utility classes
       │   ├── layout.tsx        # Root layout (fonts, theme)
       │   ├── (dashboard)/      # Authenticated layout group
       │   │   ├── layout.tsx    # Sidebar + topbar + breadcrumb shell
       │   │   ├── require-auth.tsx
       │   │   ├── overview/
       │   │   ├── analytics/
       │   │   ├── monitoring/
       │   │   ├── projects/
       │   │   ├── users/
       │   │   ├── audit/
       │   │   └── settings/
       │   ├── login/
       │   ├── signup/
       │   ├── verify-email/
       │   ├── forgot-password/
       │   ├── reset-password/
       │   └── onboarding/
       ├── components/
       │   ├── ui/               # Primitives: Button, Badge, Dialog, etc.
       │   ├── shared/           # PageHeader, StatCard, EmptyState, etc.
       │   ├── force-graph.tsx   # D3 force-directed graph
       │   ├── breadcrumb.tsx    # Breadcrumb trail component
       │   └── theme-provider.tsx
       ├── hooks/
       │   └── use-pinned-projects.ts
       ├── lib/
       │   ├── api-client.ts     # Centralized API client
       │   └── utils.ts          # Shared utilities
       └── stores/
           └── project-context.tsx  # Project state provider

------------------
Cross-References
------------------

- :ref:`api_layer` — OpenZync REST API (the backend that this frontend consumes)
- :ref:`auth` — Authentication architecture (JWT, MFA, refresh token flow)
- :ref:`graph_backends` — Graph backend configuration (PostgreSQL/pgvector,
  SurrealDB)
- :ref:`llm` — LLM provider configuration (OpenAI, Anthropic, etc.)
- :ref:`memory_context` — Memory ingestion, context assembly, hybrid search
- :ref:`workers` — Background job infrastructure (RQ workers for enrichment)
- :ref:`core` — Core platform concepts and data model

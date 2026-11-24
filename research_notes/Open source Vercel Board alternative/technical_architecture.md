# Technical architecture for an open-source Vercel Board alternative (November 2026)

## 1. What core architecture supports a reliable small-team board with realtime updates and self-hosting?

### Takeaway

Build a **single-process TypeScript application on Bun 1.4.5 using Effect 4.1.5 across the stack**: React for rendering, `@effect/atom-react` for browser state and data flows, `@effect/platform-bun` for HTTP/SSE/long-polling, and embedded SQLite as the sole durable store. Run HTTP, auth, background workers, and realtime fan-out as Effect fibers in one Bun process; ship it behind an optional reverse proxy with one persistent local data volume. Use WAL mode, short `BEGIN IMMEDIATE` transactions, an application-level single-writer gate, and a durable event/outbox log. PostgreSQL, Redis/Valkey, brokers, and Kubernetes are explicitly deferred. This fits the current repository because it contains no application code or stack commitment, but the product shape must be confirmed first: the actual Vercel Board is a channels/threads/tasks message board for humans and agents, not a conventional kanban board.

### Cited Findings

- Vercel introduced Board on **2026-11-18** as an experimental, harness-agnostic communication primitive. Its documented hierarchy is team → board → channel → thread → message, with labels, actors/profiles, grants, an ordered event log, tasks, CLI/SDK access, and sync connections. — [Vercel Board announcement](https://vercel.com/blog/introducing-vercel-board) (accessed 2026-11-24); [Vercel Board documentation](https://vercel.com/docs/board) (accessed 2026-11-24).
- Board's message ID is a cursor, its event log is ordered per board, and tasks support claiming, leases, dependencies, and completion. Agents normally act through a person's credentials under a named profile; a profile changes attribution, not authority. — [Vercel Board documentation](https://vercel.com/docs/board) (accessed 2026-11-24); [Vercel Board announcement](https://vercel.com/blog/introducing-vercel-board) (accessed 2026-11-24).
- Board is explicitly experimental, is free during the experiment, stores data in the United States with no region setting, and documents limits of 10,000 characters and 10 attachments of up to 100 MB per message. These are comparison facts, not automatically OpenBoard requirements. — [Vercel Board documentation](https://vercel.com/docs/board) (accessed 2026-11-24).
- `BunHttpServer` supports streaming bodies, multipart requests, file responses, WebSocket upgrades, and graceful shutdown. It defaults to no idle timeout, so a directly exposed server should set one or sit behind a reverse proxy. — [BunHttpServer](https://effect.website/docs/v4/api/platform-bun/BunHttpServer) (accessed 2026-11-24).
- Bun 1.4.5 is the current npm/official release as of the research date; it is a security release bundling SQLite 3.54.1. Bun's built-in `bun:sqlite` API is synchronous and provides prepared statements, transactions, serialization, strict mode, and safe integer handling. — [Bun v1.4.5 release](https://bun.com/blog/bun-v1.4.5) (accessed 2026-11-24); [Bun SQLite docs](https://bun.com/docs/runtime/sqlite) (accessed 2026-11-24); [npm `bun`](https://www.npmjs.com/package/bun/v/1.4.5) (accessed 2026-11-24).
- SQLite WAL allows concurrent readers with one writer, but all processes must be on the same host and WAL does not work over a network filesystem. `BEGIN IMMEDIATE` acquires the write lock up front and avoids read-to-write upgrade failures. — [SQLite WAL](https://www.sqlite.org/wal.html) (accessed 2026-11-24); [SQLite transactions](https://www.sqlite.org/lang_transaction.html) (accessed 2026-11-24).
- `@effect/sql-sqlite-bun` 4.1.5 connects Effect SQL to `bun:sqlite`, serializes database access, enables WAL, defaults to a five-second busy timeout, and uses `BEGIN IMMEDIATE` for writable transactions. Busy waits block the event loop because the driver is synchronous; streaming queries and `updateValues` are not implemented. — [Effect Bun SqliteClient](https://effect.website/docs/v4/api/sql-sqlite-bun/SqliteClient) (accessed 2026-11-24).
- Socket.IO supports multiple nodes, but its Redis adapter requires sticky sessions unless HTTP long-polling is disabled. Its adapter messages are not signed or encrypted, so Redis must be trusted, private, authenticated, and isolated. — [Socket.IO multiple-node guide](https://socket.io/docs/v4/using-multiple-nodes/) (accessed 2026-11-24); [Socket.IO Redis adapter](https://socket.io/docs/v4/redis-adapter/) (accessed 2026-11-24).
- Better Auth supports SQLite through `better-sqlite3`, `node:sqlite`, and **`bun:sqlite`**, and supports schema generation/migration. Its official Effect integration mounts the Web Standard auth handler with `HttpEffect.fromWebHandler`. — [Better Auth SQLite](https://www.better-auth.com/docs/adapters/sqlite) (accessed 2026-11-24); [Better Auth Effect integration](https://better-auth.com/docs/integrations/effect) (accessed 2026-11-24).
- As of 2026-11-24, the official Effect v4 API reference and npm registry identify **4.1.5** as current for `effect`, `@effect/platform-bun`, `@effect/sql-sqlite-bun`, and `@effect/atom-react`. Effect 4.1 adds crypto services, HTTP Problem Details, and pool metrics, but its release notes warn of breaking changes; many relevant HTTP, SQL, and Atom APIs are explicitly marked **unstable**, so exact-version pinning and adapter isolation are required. — [Effect v4 API](https://effect.website/docs/v4/api) (accessed 2026-11-24); [Effect 4.1 release](https://effect.website/blog/releases/effect/41) (accessed 2026-11-24); [npm `effect`](https://www.npmjs.com/package/effect/v/4.1.5) (accessed 2026-11-24).
- Plane, a comparable open-source project-management product, uses PostgreSQL for transactional data, Redis/Valkey for cache and realtime pub/sub, a broker plus Celery workers for asynchronous work, and S3-compatible object storage. Plane also documents the operational burden of self-hosting and recommends managed data services for production. — [Plane self-hosting architecture](https://developers.plane.so/self-hosting/self-hosting-101) (accessed 2026-11-24).
- Supabase's self-hosted Docker stack supplies Auth, REST, Realtime, and Storage, but its documented minimum is 4 GB RAM/2 cores/40 GB and its recommendation is 8 GB+/4 cores+/80 GB+. Unneeded services can be removed. — [Supabase Docker self-hosting](https://supabase.com/docs/guides/self-hosting/docker) (accessed 2026-11-24).
- Repository inspection found only `.git/`, `AGENTS.md`, `opencode.json`, `docs/agents/`, `reports/`, and `research_notes/`; there is no package manifest, application directory, `GLOSSARY.md`, or ADR to preserve. The repository instructions designate Linear as the issue tracker, GitHub as code hosting, and `docs/adr/` plus a lazy root glossary as the domain-documentation locations. — [AGENTS.md](file:///home/hvnxdr/dev/openboard/AGENTS.md), [issue-tracker instructions](file:///home/hvnxdr/dev/openboard/docs/agents/issue-tracker.md), and [domain instructions](file:///home/hvnxdr/dev/openboard/docs/agents/domain.md) (local files, accessed 2026-11-24).
- A Linear query for all issues in `STARHACKED` / `P-STR-10` returned no issues on the research date, so there is no verified product-definition ticket in the repository's designated tracker. — [OpenBoard Linear project](https://linear.app/heavenaldrico/project/openboard-dfcb51509e0d) (accessed 2026-11-24).

### Inferences

**Recommended MVP topology**

```text
Browser / CLI / agent
        |
  Optional TLS reverse proxy (Caddy/nginx)
        |
  One Bun 1.4.5 process
    ├── React static assets / initial snapshot shell
    ├── Effect HTTP API + auth + SSE/long-poll
    ├── Effect worker fibers (outbox, webhooks, backups)
    └── In-process PubSub for realtime fan-out
        |
  One local persistent volume
    ├── openboard.sqlite (+ -wal/-shm while open)
    └── attachments/<sha256>/...
```

- Keep web, API, and worker as separate **processes/containers but one deployable codebase and one release**. The API owns domain transactions and sockets; the worker owns slow or retryable work. This avoids running durable background work inside short-lived web requests while retaining a modular monolith rather than premature microservices.
- Pin Bun **1.4.5** and the Effect packages to **4.1.5**. Bun 1.4.5 bundles SQLite 3.54.1 and contains security updates; macOS uses the system SQLite, so production containers should use the Linux Bun image for a consistent SQLite build. — [Bun v1.4.5](https://bun.com/blog/bun-v1.4.5) (accessed 2026-11-24).
- Do not use PostgreSQL `LISTEN/NOTIFY` in v1. Every mutation writes a durable event/outbox row in the same SQLite transaction. A dispatcher fiber publishes committed events to an in-process Effect `PubSub`; browser SSE and Board-compatible long-poll clients subscribe to per-board queues. After a crash or missed message, clients catch up from the durable event log.
- Keep the Effect HTTP API on the same origin as the web app. That simplifies cookies, CSRF, CSP, and proxy configuration. Use HTTP long-poll for upstream Board SDK/CLI compatibility and SSE for the browser UI; both are implemented with Effect streams/queues. Socket.IO is not needed in the default MVP, avoiding sticky sessions and a second realtime protocol. — [BunHttpServer](https://effect.website/docs/v4/api/platform-bun/BunHttpServer) (accessed 2026-11-24); [Effect SSE](https://effect.website/docs/v4/api/effect/encoding/Sse) (accessed 2026-11-24); [Board SDK event API](https://vercel.com/docs/board/sdk) (accessed 2026-11-24).
- A client first fetches a snapshot plus `board_event_seq`, then subscribes. On reconnect it requests events after its last sequence. If that cursor predates retained events, it fetches a fresh snapshot. Presence and typing state are ephemeral and are not part of the durable model.
- Do not adopt CRDT/Yjs for MVP unless simultaneous editing of the same rich document is a confirmed requirement. Messages, task claims, labels, and status changes are better represented as authorized atomic commands with versions. Plane uses Yjs for collaborative pages/work items, but that is a stronger requirement than a message board. — [Plane self-hosting architecture](https://developers.plane.so/self-hosting/self-hosting-101) (accessed 2026-11-24).

**Build versus adopt**

| Concern | MVP decision | Rationale |
|---|---|---|
| Web renderer/state | **Adopt React + TypeScript + Bun-built static client + Effect Atom + `@effect/atom-react`** | React renders; Bun builds/serves the client in the same process; Effect owns asynchronous data, commands, and shared client state. Current v4 hooks include `RegistryProvider`, `useAtomValue`, `useAtomSet`, and `useAtomRefresh`. — [Bun bundler](https://bun.com/docs/bundler) and [Atom React hooks](https://effect.website/docs/v4/api/atom-react/Hooks) (accessed 2026-11-24). |
| API/realtime | **Adopt Bun 1.4.5 + Effect 4.1.5 Platform (`@effect/platform-bun`)** | Use `BunHttpServer`, `HttpRouter`, and Effect streams for HTTP, SSE, and Board-compatible long-polling in one process; no Fastify or Socket.IO. — [BunHttpServer](https://effect.website/docs/v4/api/platform-bun/BunHttpServer) (accessed 2026-11-24). |
| Authentication | **Adopt Better Auth with `bun:sqlite`, mounted inside Effect** | Better Auth supports Bun's built-in SQLite and its Web handler mounts with `HttpEffect.fromWebHandler`. Gate auth writes with the same application write semaphore as domain writes. — [Better Auth SQLite](https://www.better-auth.com/docs/adapters/sqlite) and [Effect integration](https://better-auth.com/docs/integrations/effect) (accessed 2026-11-24). |
| Database/access | **Adopt embedded SQLite + Effect SQL (`@effect/sql-sqlite-bun`) + `SqliteMigrator`** | One serialized client, WAL, `BEGIN IMMEDIATE`, typed `SqlError`, and transactional migrations. Better Auth uses a second `bun:sqlite` connection to the same file; a shared write gate prevents contention. — [Effect Bun SqliteClient](https://effect.website/docs/v4/api/sql-sqlite-bun/SqliteClient) (accessed 2026-11-24). |
| Validation/contracts | **Adopt Effect Schema; JSON Schema validator only for Board structured blocks** | Shared request/response/event/error schemas replace Zod. Upstream Board's user-registered structured blocks are JSON Schema, so that specific boundary still needs a compatible validator. — [Effect v4 API](https://effect.website/docs/v4/api) (accessed 2026-11-24); [Board SDK structured data](https://vercel.com/docs/board/sdk) (accessed 2026-11-24). |
| Domain model, grants, claims, event log | **Build** | These are OpenBoard's differentiators and must be small, explicit, auditable modules. |
| Durable background work | **Build a thin SQLite outbox/inbox worker as Effect fibers in the same process** | Claim jobs with an atomic conditional `UPDATE ... RETURNING` transaction; no broker and no `SKIP LOCKED`. |
| Redis/Valkey, RabbitMQ, Kafka | **Defer** | Add only for measured fan-out, queue throughput, or multi-region needs; each adds backup, security, and failure modes. |
| Object storage | **Use co-located local files in v1** | Store immutable content-addressed files under `/data/attachments`; metadata stays in SQLite. Add an S3-compatible adapter only when multi-host or large-volume storage is required. |
| Kubernetes | **Defer** | Compose plus documented VM deployment is the MVP. Kubernetes becomes justified by HA, multiple operators, or many isolated instances. |

**Alternatives considered but not selected**

1. **Supabase or another hosted database** is not selected for v1 because the user chose embedded SQLite. It may become relevant only in a future PostgreSQL migration, not as the application framework.
2. **Plane** is useful as an operational reference, not as OpenBoard's framework or a drop-in agent board. Its Community Edition is AGPLv3, Commercial/Airgapped editions are closed-source and license-gated, and its architecture is much larger than the proposed Effect monolith. — [Plane editions and licensing](https://developers.plane.so/self-hosting/self-hosting-101) (accessed 2026-11-24).
3. **Effect Cluster/Workflow/Durable services** are later options if multi-node scheduling or durable workflows become requirements. They are not needed for a single-process SQLite-backed MVP worker. — [Effect v4 API](https://effect.website/docs/v4/api) (accessed 2026-11-24).

**Must-have versus later**

- **MVP:** one Bun process and one local data volume; workspace authorization; boards/channels/threads/messages; tasks and leases if agent coordination is in scope; labels; ordered events; REST API; CLI; web UI; SSE/long-poll catch-up; one Git-provider adapter; durable SQLite webhook inbox; in-process worker/retries/replay; audit events; export; automated backup; health checks; metrics; and an upgrade runbook.
- **Later:** a future PostgreSQL migration for HA/PITR or multi-host scale, SCIM/SAML, plugin execution, full two-way Slack/GitHub synchronization, collaborative document editing, external search, and multi-region active-active.
- **Explicitly not MVP:** Kafka, RabbitMQ, Redis, Kubernetes, multiple app replicas sharing one SQLite file, a general-purpose policy language, a public plugin marketplace, event sourcing for every aggregate, or running every integration provider on day one.

### Gaps

- The assigned product-definition research is not present in the repository or Linear. Confirm whether OpenBoard targets Board's channels/threads/tasks model, a conventional project/work-item/kanban model, or both.
- Confirm whether agent leases, dependencies, watchers, schedules, structured payload schemas, reactions, and attachments are MVP requirements. They materially change the schema and CLI.
- Confirm single-tenant self-hosting versus a hosted multi-tenant service. The latter raises the authorization bar and makes database RLS, tenant-isolation testing, abuse controls, and operational separation much more important.
- No load test or benchmark exists for this repository. Capacity statements in these notes are engineering targets, not measured guarantees.

## 2. Which data model and consistency boundaries are needed?

### Takeaway

Use a **relational state model plus an append-only per-board event log**, not full event sourcing. The task/thread is the primary work aggregate; messages, labels, claims, and external links change through transactions that also append an activity event and enqueue follow-up work. Keep canonical task state separate from free-form labels, keep human-facing activity separate from security audit events, and treat every provider as an external system with explicit mappings rather than trying to maintain distributed transactions.

### Cited Findings

- Vercel Board's documented hierarchy is team → board → channel → thread → message; messages contain Markdown, labels, reactions, attachments, and structured data. A task is a thread with board-maintained state, and every board has an ordered event log. — [Vercel Board documentation](https://vercel.com/docs/board) (accessed 2026-11-24).
- Board documents task claims, renewable leases, dependencies, and completion, and its CLI exposes atomic claim/complete operations. — [Vercel Board announcement](https://vercel.com/blog/introducing-vercel-board) (accessed 2026-11-24).
- SQLite supports multiple readers but only one writer; `BEGIN IMMEDIATE` starts a write transaction immediately and avoids read-to-write upgrade `SQLITE_BUSY` failures. Bun's transaction wrapper must remain synchronous, and nested transactions use savepoints. — [SQLite transactions](https://www.sqlite.org/lang_transaction.html) (accessed 2026-11-24); [Bun SQLite transactions](https://bun.com/docs/runtime/sqlite#transactions) (accessed 2026-11-24).
- PostgreSQL row-level security can restrict normal reads and writes per row, but owners and `BYPASSRLS` roles bypass it unless explicitly forced; policies that consult other tables can introduce races. — [PostgreSQL row security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html) (accessed 2026-11-24).
- Better Auth's organization plugin supplies organizations, members, teams, roles, and permissions, and can store dynamic access-control roles in the database. — [Better Auth organization plugin](https://www.better-auth.com/docs/plugins/organization) (accessed 2026-11-24).
- GitHub can deliver webhooks out of order, and its documentation advises using payload timestamps or fetching current object state when an event may be stale. — [GitHub webhook troubleshooting](https://docs.github.com/en/webhooks/testing-and-troubleshooting-webhooks/troubleshooting-webhooks) (accessed 2026-11-24).

### Inferences

**Proposed core schema**

| Entity | Important fields and constraints | Purpose |
|---|---|---|
| `workspaces` | `id`, `name`, `created_at` | Tenant/team boundary. Maps to the assignment's “project container.” |
| `users`, `memberships` | Better Auth user plus `workspace_id`, role, status | Human identity and workspace membership. Better Auth supplies the identity tables. — [Better Auth](https://www.better-auth.com/docs/installation) (accessed 2026-11-24). |
| `principals` | `id`, `kind` (`user`, `project`, `service`, `app`), external identity reference | Authorization subject. Users may map one-to-one; deployments, agents, and apps get distinct principals. |
| `actor_profiles` | `principal_id`, `delegated_principal_id`, `display_name` | Attribution such as “fx via ada.” A profile has no independent permissions, matching Board's documented model. — [Vercel Board](https://vercel.com/docs/board) (accessed 2026-11-24). |
| `boards` | `workspace_id`, `name`, `event_seq`, `archived_at` | Top-level collaboration boundary and realtime cursor allocator. |
| `channels` | `board_id`, normalized `path`, `created_at` | Unique path per board, e.g. `dev` or `www/production`. |
| `grants` | `board_id`, `path_prefix`, principal/group, permission bitset, `deny`, priority | Fixed read/post/reply/manage/claim/export permissions with longest-prefix evaluation and explicit deny precedence. Workspace owners can administer all grants. |
| `threads` | `board_id`, `channel_id`, `root_message_id`, `created_by`, `created_at`, `archived_at` | Conversation/work container. |
| `messages` | `thread_id`, `author_actor_id`, `body_markdown`, `reply_to_id`, `version`, `edited_at`, `deleted_at`, `client_request_id` | Comments/replies and root messages. Unique `(thread_id, client_request_id)` makes retries idempotent. Deletion leaves a tombstone. |
| `labels` / `message_labels` | `key`, `value`, message association, unique key per message | Typed metadata and filtering; not the authority for task state. |
| `structured_payloads` | `message_id`, `schema_name`, `schema_version`, validated JSON | Optional Board-style typed agent data. Store the registered schema version with the value. — [Vercel Board announcement](https://vercel.com/blog/introducing-vercel-board) (accessed 2026-11-24). |
| `tasks` | `thread_id UNIQUE`, `state`, `assignee_principal_id`, `lease_expires_at`, `claim_version`, `priority`, `completed_at` | Canonical work-item state. Every MVP task is thread-backed, avoiding two competing sources of truth. |
| `task_claims` | `task_id`, actor, claimed/released/expired timestamps, lease token hash | Claim history and auditability. Only the current claim is authoritative. |
| `task_dependencies` | `task_id`, `depends_on_task_id`, unique pair | Agent-swarm readiness. Readiness is derived from dependency states. |
| `attachments` | message, local object key, size, MIME, SHA-256, uploader, scan/status | Bytes live as immutable files under the same local data volume; metadata and authorization live in SQLite. |
| `board_events` | `board_id`, `board_seq`, `type`, `aggregate_type/id`, actor, minimal payload, `created_at` | Ordered activity stream and realtime catch-up. Unique `(board_id, board_seq)`. |
| `audit_events` | workspace, actor, action, target, result, request ID, timestamp | Append-only security/admin history: grants, exports, auth events, integration changes, destructive actions. |
| `integration_connections` | workspace, provider, account/installation, encrypted credentials reference, scopes, status | One installation per provider account; credentials are encrypted or delegated to a secret store. |
| `external_links` / `sync_records` | connection, provider kind/id, local thread/channel/message, direction, external revision, visibility, last error | Durable provider mapping. Unique `(connection_id, provider_kind, external_id)`. |
| `webhook_deliveries` | connection, provider delivery ID, event type, payload/hash, received/processed timestamps, attempts, status | Durable inbound inbox and deduplication record. |
| `outbox` / `jobs` | kind, aggregate, idempotency key, payload, attempts, `next_attempt_at`, `locked_at`, terminal status | Transactional follow-up work, outbound sync, notifications, exports, reconciliation. |
| `idempotency_keys` | principal, route, key, request hash, stored response | Safe retries for CLI/API mutations. |

**Mapping the assignment's concepts to Board's actual model**

- **Project** → workspace or board, depending on the product definition. Do not introduce both until needed.
- **Work item** → `tasks.thread_id` in a Board-like product; optionally a standalone `work_items` table later if kanban issues are confirmed.
- **Status** → canonical `tasks.state` (`ready`, `claimed`, `in_progress`, `blocked`, `review`, `done`, `canceled`). Labels remain metadata. If configurable workflows are confirmed, add versioned `status_definitions` rather than overloading labels.
- **Assignment** → current claim plus lease history. This is stronger than a plain assignee field because multiple agents may race for the same task.
- **Comment** → `messages` reply; **activity** → `board_events`; **external link** → `external_links` plus `sync_records`.
- **Custom kanban status/order** → a later projection over work items, not a change to the message/event core.

**Consistency boundaries**

1. **Send message / reply:** in one transaction, authorize the channel, insert the message, increment its thread version, allocate a board sequence, append a `board_event`, and insert an outbox job if notifications/sync are needed. Return only after commit.
2. **Edit/delete message:** use optimistic concurrency (`If-Match`/version). A stale writer receives `409` and must refetch. Edits append events rather than erasing history; deletion writes a tombstone and may enqueue best-effort external deletion.
3. **Claim/renew/complete task:** lock the task row or perform a conditional update. A claim succeeds only if the task is ready and unclaimed or its lease expired. The update increments `claim_version`; stale lease tokens cannot renew or complete it. Append claim/state events in the same transaction.
4. **Reorder or status changes:** if kanban is later confirmed, store a stable rank such as an integer with gaps or a lexicographic key and version the affected aggregate. Never rely on floating-point positions or client timestamps.
5. **Grant changes:** update the grant and append an audit event atomically. Existing sockets are not trusted; authorization is re-evaluated when rooms are joined and when privileged operations occur. On revocation, disconnect or refresh affected sessions promptly.
6. **External ingestion:** never update local state in the webhook request handler's provider call. Persist the delivery first; a worker normalizes it and applies a separate local transaction with an idempotent upsert.
7. **Cross-provider operations:** no distributed transaction. OpenBoard is authoritative for planning/task state; GitHub is authoritative for PR/repository state; a deployment provider is authoritative for deployment state. Store the source and revision of mirrored data.

**Event and cursor design**

- Allocate `board_seq` by atomically incrementing the board row in the same transaction as the event. At small-team scale, this serialization is acceptable; if a single board becomes hot, allocate per-channel sequences or use a partitioned sequencer.
- Events contain IDs, types, versions, and minimal non-sensitive metadata. Clients fetch authorized details after receiving an event. Never broadcast secrets, full grant changes, or unrestricted payloads.
- Retain the event log at least as long as clients may reconnect. Archive old events rather than deleting them silently; if the requested cursor is no longer available, return a `resnapshot_required` response.
- SQLite triggers can enforce event insertion for critical tables, but application transactions should still make the invariant explicit and testable. Do not publish realtime events from a trigger; publish only after the transaction commits.

**Authorization placement**

- Enforce tenant/board scope in every repository/query path in MVP; do not accept a bare `thread_id` and then fetch it globally.
- SQLite v1 has no row-level security. Enforce tenant/board scope in every repository/query path and test cross-tenant access. If a future PostgreSQL migration or hosted multi-tenant service is undertaken, add forced PostgreSQL RLS as defense in depth. — [PostgreSQL row security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html) (accessed 2026-11-24).
- Keep the MVP permission vocabulary fixed and small. A general policy engine, custom roles UI, and arbitrary expressions can wait.

### Gaps

- It is not yet known whether “project” means a workspace, board, or repository grouping, or whether one work item can belong to multiple projects.
- Confirm whether assignments are single-owner, multi-assignee, or lease-based agent claims. The recommended model assumes lease-based claims because that matches Board.
- Confirm configurable statuses, task dependencies, reactions, structured data, and attachment requirements. Each optional feature should be removed from MVP unless product research marks it essential.
- Confirm retention and edit/delete semantics, especially whether synced external copies must be deleted and how tombstones should appear.

## 3. Which integrations provide real value in an MVP, and which can wait?

### Takeaway

The non-negotiable “integration” surface for an agent-board alternative is a **first-class HTTP API, CLI, event stream/tail/wait, and machine identities**. Beyond that, start with **one Git provider—GitHub, because this repository already uses GitHub—for linking and read-only PR/issue/check context**. Add one deployment-provider adapter only if deployment visibility is a confirmed use case. Full two-way Slack/GitHub synchronization, additional Git providers, and deployment write actions should wait.

### Cited Findings

- Board's primary interfaces are the CLI and JavaScript SDK; its announcement demonstrates agent task claiming, updates, lease renewal, completion, watchers, schedules, structured data, and two-way GitHub/Slack sync. — [Vercel Board documentation](https://vercel.com/docs/board) (accessed 2026-11-24); [Vercel Board announcement](https://vercel.com/blog/introducing-vercel-board) (accessed 2026-11-24).
- GitHub recommends subscribing only to necessary webhook events, using a secret and HTTPS, responding with a 2xx within 10 seconds, processing asynchronously, checking event/action, and using `X-GitHub-Delivery` to identify redeliveries. — [GitHub webhook best practices](https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks) (accessed 2026-11-24).
- GitHub webhook deliveries may arrive out of order; GitHub recommends payload timestamps or fetching the current object state. — [GitHub webhook troubleshooting](https://docs.github.com/en/webhooks/testing-and-troubleshooting-webhooks/troubleshooting-webhooks) (accessed 2026-11-24).
- GitHub Apps have no permissions by default, and GitHub explicitly recommends selecting the minimum permissions required. Installation access-token requests depend on app permissions and can be narrowed to selected repositories. — [Choosing GitHub App permissions](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app) (accessed 2026-11-24); [Generating installation access tokens](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token) (accessed 2026-11-24).
- Vercel webhook payloads include a delivery `id`, event `type`, `createdAt`, and payload. Vercel signs the raw body with `x-vercel-signature` using HMAC-SHA1 and retries non-2xx deliveries with exponential backoff for up to 24 hours before discarding them. — [Vercel Webhooks API](https://vercel.com/docs/webhooks/webhooks-api) (accessed 2026-11-24).
- Vercel's setup documentation says account webhooks are available on Enterprise and Pro plans; integration/webhook access must therefore be checked against the intended customer's plan and scopes. — [Vercel webhooks setup](https://vercel.com/docs/webhooks) (accessed 2026-11-24).
- Board warns that a synced channel is as readable as the external platform channel: Board grants do not constrain Slack, GitHub, or Linear readers, and each platform keeps its own copy. — [Vercel Board documentation](https://vercel.com/docs/board) (accessed 2026-11-24).

### Inferences

**Integration priority**

| Priority | Capability | MVP decision |
|---|---|---|
| P0 | REST API + OpenAPI + versioning | **Build.** Required for CLI, agents, scripts, and future integrations. |
| P0 | CLI with `whoami`, list/read/post/reply, task claim/renew/complete, tail/wait, export | **Build** if agent coordination is in scope. Keep CLI a thin API client. |
| P0 | Scoped service tokens or workload identity | **Build.** Never require an agent to reuse a human's broad session for unattended work. |
| P0 | Per-board event stream and webhook ingress framework | **Build.** Provides realtime catch-up, automation, and durable provider ingestion. |
| P1 | GitHub App: link repositories, issues, PRs; show state/checks | **Build one adapter.** Read-only first. This repository already links to GitHub. — [Repository instructions](file:///home/hvnxdr/dev/openboard/docs/agents/issue-tracker.md) (accessed 2026-11-24). |
| P1 conditional | GitHub inbound messages or branch/PR creation | Only if confirmed. Even then, gate writes behind explicit grants and audit them. |
| P1 conditional | Vercel project/deployment status and preview links | **Build only if deployment visibility is validated.** Keep it provider-specific and read-only. |
| P2 | Slack two-way channel/thread sync | Defer. Mapping identities, edits, deletes, reactions, permissions, and loops is substantial. |
| P2 | GitLab/Bitbucket, Netlify/Render/Fly adapters | Defer until demand is proven. Use a common adapter interface so each is additive. |
| P2 | Linear/Jira issue sync | Defer; the repository uses Linear for its own tracker, but that does not prove it belongs in OpenBoard's MVP. |
| P3 | Deploy/rollback/merge actions, public plugin runtime | Defer; high-risk writes require a much stronger permission and audit model. |

**GitHub adapter boundary**

- Connect through a GitHub App installed on selected repositories, not a personal access token. Request only the exact read permissions needed for the chosen endpoints and events; do not request administration or contents permissions merely to display a link. — [GitHub App permissions](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app) (accessed 2026-11-24).
- Persist installation ID, selected repositories, granted scopes, and encrypted short-lived token material. Generate installation access tokens on demand and refresh them; GitHub documents a one-hour expiry. — [GitHub installation tokens](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token) (accessed 2026-11-24).
- MVP writes should be limited to local links and comments. Mirror PR title/state/checks and issue state into a read model. Do not let an external PR automatically complete a local task unless product requirements explicitly call for it and the mapping is unambiguous.
- Model the mapping as `thread ↔ issue/PR`, not “repository = project.” A repository can contain many threads, and one thread may link multiple PRs.

**Deployment adapter boundary**

- Normalize provider events into a small internal vocabulary: `deployment.created`, `queued`, `building`, `ready`, `failed`, `canceled`, `promoted`, `rolled_back`.
- Store provider, project ID, deployment ID, target, URL, source revision, state, and provider event time. OpenBoard displays this context; it does not become the deployment system of record.
- Follow each provider's actual signature scheme. GitHub documents HMAC-SHA256; Vercel currently documents HMAC-SHA1. Use the provider's scheme with constant-time comparison over raw bytes rather than inventing a uniform but incompatible format. — [GitHub delivery validation](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries) (accessed 2026-11-24); [Vercel Webhooks API](https://vercel.com/docs/webhooks/webhooks-api) (accessed 2026-11-24).

**Common adapter contract**

Each adapter should implement: `validateConnection`, `listAvailableResources`, `verifyWebhook(rawBody, headers)`, `normalizeEvent`, `fetchCurrentResource`, `reconcile`, and optional `publish`/`delete`. Provider-specific payloads must not leak into domain services. Integration credentials stay behind this boundary.

### Gaps

- Product research must decide whether deployment visibility is core or incidental. “Alternative to Vercel Board” does not by itself require Vercel integration.
- Confirm the first Git provider and whether GitHub Enterprise Server must be supported; its API base URL and installation flow differ from github.com.
- Confirm whether two-way sync is an MVP promise. If yes, scope must include edits, deletions, reactions, identity mapping, permission warnings, loop prevention, and conflict rules.
- Confirm whether agents need outbound write actions (open PR, comment, merge, deploy). Those are not recommended for MVP without a separate high-risk permission model.

## 4. How should webhooks, retries, idempotency, and reconciliation work?

### Takeaway

Use a **durable inbox → normalized event → idempotent domain transaction → durable outbox** pipeline. A webhook endpoint verifies the signature on raw bytes, validates and persists the delivery, and returns 2xx only after durable acceptance. Workers process asynchronously with bounded exponential backoff and jitter, dead-letter permanent failures, and reconcile linked resources because providers can drop, duplicate, delay, or reorder events.

### Cited Findings

- GitHub expects a 2xx within 10 seconds and recommends a queue for asynchronous processing. Redelivery reuses the original `X-GitHub-Delivery` value. — [GitHub webhook best practices](https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks) (accessed 2026-11-24).
- GitHub may deliver events out of order. — [GitHub webhook troubleshooting](https://docs.github.com/en/webhooks/testing-and-troubleshooting-webhooks/troubleshooting-webhooks) (accessed 2026-11-24).
- Vercel retries non-2xx deliveries with exponential backoff for up to 24 hours, then discards them; its payload contains a delivery ID and event creation time. — [Vercel Webhooks API](https://vercel.com/docs/webhooks/webhooks-api) (accessed 2026-11-24).
- OWASP recommends raw-body signature verification, constant-time comparison, one secret per webhook, POST-only endpoints, payload limits, schema validation, event-ID deduplication, asynchronous processing, 401 for signature failures, and durable acceptance before returning 200. It also warns that GitHub's body-only signature does not authenticate a timestamp or delivery header, so downstream effects must be idempotent. — [OWASP Webhook Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Webhook_Security_Cheat_Sheet.html) (accessed 2026-11-24).
- OWASP recommends structured logs without secrets or full request bodies and alerting on signature-failure spikes and sustained errors. — [OWASP Webhook Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Webhook_Security_Cheat_Sheet.html) (accessed 2026-11-24).
- SQLite has one writer. A single in-process worker claims a job inside a short `BEGIN IMMEDIATE` transaction using a conditional `UPDATE ... RETURNING`, then performs provider I/O after the claim commits. — [SQLite transactions](https://www.sqlite.org/lang_transaction.html) (accessed 2026-11-24).

### Inferences

**Inbound pipeline**

1. Route accepts only `POST /integrations/{provider}/webhooks/{connectionKey}` over HTTPS.
2. Enforce a conservative body limit before parsing. Capture the exact raw bytes.
3. Look up the connection and verify its signature with the documented algorithm and a constant-time comparison. Reject missing/invalid signatures with a generic 401; reject malformed payloads with 400.
4. Validate the envelope and derive a stable delivery key: GitHub `X-GitHub-Delivery`, Vercel payload `id`, or a documented provider equivalent. Record the payload hash as well.
5. In one short transaction, insert into `webhook_deliveries` with unique `(connection_id, delivery_key)`. A duplicate that is already accepted returns 200 without repeating side effects.
6. Return 2xx only after the inbox insert commits. Do not call provider APIs, send notifications, or perform domain writes in the request path.
7. A worker fiber claims the delivery in a short SQLite `BEGIN IMMEDIATE` transaction with a conditional `UPDATE ... RETURNING`, validates the provider-specific schema, fetches current resource state when needed, and applies an idempotent local transaction.
8. The local transaction upserts external links/read models, appends a board event if user-visible, and inserts outbox work for notifications or sync.

**Deduplication and ordering**

- Deduplicate at two levels: the provider delivery ID and a domain fingerprint such as `(provider, external_kind, external_id, normalized_state, external_revision_or_payload_hash)`. A provider may send a new delivery ID for the same state transition.
- Never assume arrival order. Store `external_updated_at`/revision when available. If revisions are comparable, ignore stale updates; if not, fetch authoritative current state before overwriting.
- Preserve deletion as a tombstone. Do not hard-delete local history because a provider event says “deleted.”
- Use an `origin` field and source-message mapping on outbound events to prevent sync loops.

**Retry policy**

- Classify errors as permanent (bad schema, revoked installation, permission denied), transient (timeout, 429, 5xx), or retryable conflict (resource temporarily unavailable).
- Suggested starting policy: retry after 1, 5, 15, and 60 minutes, then hourly with jitter, capped at 10 attempts or 24 hours. Make these settings configurable per integration.
- Honor `Retry-After` and provider rate-limit headers. Isolate slow providers so one cannot exhaust the worker pool.
- After exhaustion, mark the job `dead_letter`, retain the delivery and error summary, alert an administrator, and expose an audited manual replay action. Never silently discard a failed event.
- Keep raw payloads only as long as needed for replay and debugging; redact secrets and restrict access. OWASP warns against logging full webhook bodies. — [OWASP Webhook Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Webhook_Security_Cheat_Sheet.html) (accessed 2026-11-24).

**Reconciliation**

- Reconcile active linked objects every 15–60 minutes and all linked objects nightly. These intervals are starting points, not provider guarantees.
- List or fetch the provider's current state, compare it with the local read model, and enqueue corrections through the same idempotent domain path used by webhooks.
- Reconcile after connection creation, token refresh/revocation, provider outage, dead-letter replay, and installation removal.
- Vercel's documented 24-hour retry window makes reconciliation necessary for missed delivery windows. GitHub also documents manual redelivery, but OpenBoard should not depend on an operator noticing a gap. — [Vercel Webhooks API](https://vercel.com/docs/webhooks/webhooks-api) (accessed 2026-11-24); [GitHub webhook best practices](https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks) (accessed 2026-11-24).

**Outbound webhooks and sync**

- Persist an outbox event in the same transaction as the local change. A worker delivers to a small allowlist of configured destinations.
- If user-defined outbound URLs are added later, enforce HTTPS, block private/link-local/metadata addresses at connection time, pin DNS resolution or validate in the connect hook, and disable or revalidate redirects. These controls are necessary to prevent OpenBoard from becoming an SSRF proxy. — [OWASP Webhook Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Webhook_Security_Cheat_Sheet.html) (accessed 2026-11-24).
- Sign outbound payloads with a versioned scheme that authenticates event ID, timestamp, and raw body where possible. Publish an `openboard-event@1` schema and a replay-safe delivery ID.
- Two-way sync must define a conflict policy: for MVP, prefer local edits for local-authored content and provider state for provider-authored fields; never silently overwrite a concurrent human edit.

### Gaps

- Exact provider ordering and revision guarantees differ. Each adapter needs contract tests using recorded, genuine signed deliveries.
- Product research must set payload retention, dead-letter retention, reconciliation frequency, and whether administrators may replay events after user deletion.
- Outbound user-configurable webhooks are not recommended for MVP. If demanded, SSRF, secret rotation, delivery signing, and abuse controls become release blockers.

## 5. What security, backup, upgrade, licensing, and observability requirements are non-negotiable?

### Takeaway

Before real users, OpenBoard needs authenticated and authorized access at every boundary, signed and bounded webhook ingestion, protected secrets, encrypted off-host backups with a tested restore, versioned migrations with a rollback plan, an explicit open-source license and dependency inventory, and usable logs/metrics/alerts. Use OWASP ASVS 5.0 as the security checklist. Treat board content as untrusted input—especially because agents may interpret it as instructions.

### Cited Findings

- OWASP ASVS 5.0.0, published 2025-05-30, is a framework of security requirements for designing, developing, and testing modern web applications and services. — [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/) (accessed 2026-11-24).
- OWASP's logging guidance calls for authentication successes/failures, authorization failures, administrative actions, imports/exports, and other high-risk events, with “when, where, who, and what”; it warns against logging passwords, tokens, secrets, session identifiers, and excessive PII. — [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) (accessed 2026-11-24).
- OWASP's webhook guidance covers raw-body signatures, constant-time comparison, replay/dedup, payload limits, POST-only routes, CSRF exemption only after signature verification, fail-secure responses, and redacted monitoring. — [OWASP Webhook Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Webhook_Security_Cheat_Sheet.html) (accessed 2026-11-24).
- SQLite's Online Backup API and `VACUUM INTO` produce consistent snapshots of a live database. `VACUUM INTO` creates a compact standalone file, purges deleted content, and requires the target not to exist; an interrupted output may be incomplete. — [SQLite backup](https://www.sqlite.org/backup.html) and [VACUUM INTO](https://www.sqlite.org/lang_vacuum.html#vacuum_into) (accessed 2026-11-24).
- Bun's HTTP server should sit behind a reverse proxy in production for TLS, request limits, and slow-client protection; set an explicit idle timeout if exposed directly. — [BunHttpServer](https://effect.website/docs/v4/api/platform-bun/BunHttpServer) (accessed 2026-11-24).
- Plane describes backup/restore as the non-negotiable day-2 operation, recommends testing restores in the first month, a staging environment, managed data services for production, and roughly 0.1–0.25 FTE of ongoing operation at small scale for its larger stack. — [Plane self-hosting](https://developers.plane.so/self-hosting/self-hosting-101) (accessed 2026-11-24).
- Plane Community is AGPLv3; Commercial and Airgapped editions are closed-source and license-gated. This illustrates why OpenBoard must state its own license and keep feature gating distinct from source licensing. — [Plane editions and licensing](https://developers.plane.so/self-hosting/self-hosting-101) (accessed 2026-11-24).
- Better Auth and Effect are MIT-licensed. — [Better Auth license](https://github.com/better-auth/better-auth/blob/main/LICENSE.md) and [Effect license](https://github.com/Effect-TS/effect/blob/main/LICENSE) (accessed 2026-11-24).
- Vercel Board warns that agents read messages as instructions, that secrets should not be posted, and that agents acting through a person can read/post wherever that person can. — [Vercel Board documentation](https://vercel.com/docs/board) (accessed 2026-11-24).

### Inferences

**Security release gates**

| Area | Non-negotiable MVP control |
|---|---|
| Authentication | Email verification and secure password reset; secure, HttpOnly, SameSite session cookies; rate-limited login/reset; optional OAuth. Administrators should use MFA before a hosted/multi-tenant launch. Better Auth can supply the core flows. — [Better Auth](https://www.better-auth.com/docs/installation) (accessed 2026-11-24). |
| Authorization | Server-side checks on every read/write, socket room, export, CLI command, and job. Fixed roles plus board/channel grants. Agents receive scoped, expiring credentials; a profile never expands permission. |
| Tenant isolation | Every query is workspace/board scoped. Automated cross-tenant tests are mandatory. SQLite has no RLS; a future PostgreSQL migration can add it before hosted multi-tenancy. |
| Input/content | Parameterized SQL; strict request schemas; Markdown sanitization; safe link handling; CSP and security headers; size limits on messages, payloads, and attachments. |
| Attachments | Private bucket, random object keys, authenticated download, MIME allowlist, content sniffing, size quotas, and malware scanning before untrusted files are served if external uploads are allowed. |
| Webhooks | HTTPS, raw-body signature verification, constant-time compare, replay/dedup, payload limits, asynchronous durable inbox, and CSRF exemption only for verified webhook routes. — [OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Webhook_Security_Cheat_Sheet.html) (accessed 2026-11-24). |
| Secrets | No secrets in Git, images, logs, exports, or event payloads. Encrypt provider tokens at rest with a key outside the database; support rotation. Keep database/cache private to the deployment network. |
| SSRF/egress | MVP uses fixed provider hosts and no user-defined outbound URLs. If outbound webhooks are added, apply OWASP destination validation and network isolation. — [OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Webhook_Security_Cheat_Sheet.html) (accessed 2026-11-24). |
| Audit | Append-only `audit_events` for grants, auth, exports, imports, integration changes, token issuance/revocation, and destructive/admin actions. Runtime role cannot update/delete audit rows. Keep content out of audit records. — [OWASP logging](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) (accessed 2026-11-24). |
| Agent safety | Board text is untrusted instruction data. Require explicit approval for destructive/high-risk tool actions; isolate agent credentials by board/channel; never auto-execute arbitrary commands from messages. — [Vercel Board warning](https://vercel.com/docs/board) (accessed 2026-11-24). |
| Supply chain | Pinned image digests and lockfiles, dependency scanning, SBOM, non-root containers, minimal base images, and a documented vulnerability-response owner. |
| Data lifecycle | Document retention and deletion for messages, attachments, events, webhook payloads, logs, and backups. Export must exclude secrets and sessions. |

**Backup and disaster recovery**

- **MVP minimum:** automated consistent SQLite snapshots with `VACUUM INTO`, co-located attachment backup, configuration backup, encryption, and off-host storage. A backup on the same VM is not disaster recovery.
- **Proposed starting objectives, subject to product approval:** RPO ≤24 hours and RTO ≤4 hours for a small self-hosted deployment. SQLite v1 has no PITR; if the board becomes business-critical, increase snapshot frequency or migrate to PostgreSQL with WAL archiving/PITR.
- Suggested retention: 30 daily backups plus 12 weekly backups, with a documented deletion policy. Store encryption keys separately from backup data.
- Run a restore into a clean directory at least quarterly and before major upgrades. Verify `PRAGMA integrity_check`, `PRAGMA foreign_key_check`, row counts, attachment checksums, permissions, and a real login—not merely that the backup file exists. Plane similarly warns that an untested backup is not a backup. — [Plane](https://developers.plane.so/self-hosting/self-hosting-101) (accessed 2026-11-24).
- `VACUUM INTO` creates a consistent compact snapshot; the SQLite Online Backup API is the incremental alternative. Neither replaces a tested off-host restore process. — [SQLite backup](https://www.sqlite.org/backup.html) (accessed 2026-11-24).

**Upgrade requirements**

- One immutable release artifact for web/API/worker, with a one-shot migration job. Keep migrations forward-only and backward-compatible using expand/contract changes.
- Before upgrade: take a fresh snapshot, rehearse in staging, read Bun/Effect release notes, and verify the bundled SQLite version.
- During upgrade: run migrations before new code, drain sockets/requests, and use health/readiness checks. Do not run multiple migration jobs concurrently.
- Rollback: retain the previous image and use a compatible schema. Never “roll back” by blindly reverting a destructive migration; restore data or use a documented forward fix.
- Pin supported major versions, publish a changelog and supported-upgrade matrix, and ship security releases on a predictable cadence.
- v1 is single-process, so there is no multi-instance cache or deployment-skew coordination. Keep the previous Bun image and a verified snapshot for rollback.

**Licensing**

- Choose **Apache-2.0** if broad adoption and permissive embedding are goals; choose **AGPLv3** if network-use source disclosure is desired. Do not mix the two casually. Obtain legal review before release.
- Keep all core board functionality in the licensed open-source edition. If enterprise features are added later, make the boundary explicit and avoid calling closed-source code “open source.”
- Maintain `LICENSE`, `NOTICE` where needed, an SPDX dependency inventory/SBOM, and a policy for transitive dependency licenses. The recommended MVP libraries are permissively licensed. — [Better Auth](https://github.com/better-auth/better-auth/blob/main/LICENSE.md) and [Effect](https://github.com/Effect-TS/effect/blob/main/LICENSE) (accessed 2026-11-24).
- Use a project-specific name and branding; do not imply affiliation with Vercel.

**Observability**

- Emit structured JSON logs to stdout with timestamp, level, service/version, request ID, actor ID, workspace/board ID, action, result, and duration. Exclude message bodies, tokens, signatures, session IDs, and unnecessary PII. — [OWASP logging](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) (accessed 2026-11-24).
- Expose Prometheus metrics: request rate/latency/errors, socket connections, event-delivery lag, DB pool saturation, job queue depth/oldest age, webhook acceptance/failure/signature-failure counts, integration rate limits, storage usage, and backup age.
- Add OpenTelemetry traces across HTTP → database → job → provider boundaries, but keep sampling and payload capture conservative.
- Alert on: sustained 5xx/error budget burn, DB saturation, oldest pending job, dead-letter growth, webhook signature-failure spikes, failed backups, backup age, TLS expiry, and disk capacity.
- Provide liveness and readiness endpoints. Readiness should fail when the database is unavailable; realtime/worker degradation should be visible without necessarily taking down the whole UI.

### Gaps

- RPO/RTO, data residency, retention, compliance obligations, and on-call ownership are product/business decisions, not established by the repository.
- Confirm whether the first release is internal-only, public self-hosted, or a hosted service. Hosted multi-tenancy requires stronger isolation, abuse prevention, privacy, and incident-response requirements.
- No project license has been selected. The recommendation above is a decision aid, not legal advice.
- Confirm whether administrator MFA, SSO, or SCIM is required at launch; that depends on the target customer.

## 6. How should data portability and extensibility work?

### Takeaway

Make the SQLite file plus co-located attachment directory the portable v1 data core, and provide a **versioned full export/import bundle** in addition to raw snapshots. Expose stable REST/CLI/event contracts and narrow integration-adapter interfaces. Do not build a plugin runtime in MVP; extensibility should come from APIs, versioned events, structured payload schemas, and replaceable adapters.

### Cited Findings

- Board exposes a small fixed data model and documents structured data schemas with explicit versions such as `security-finding@1`. — [Vercel Board documentation](https://vercel.com/docs/board) (accessed 2026-11-24); [Vercel Board announcement](https://vercel.com/blog/introducing-vercel-board) (accessed 2026-11-24).
- Bun exposes `Database.serialize()`/`Database.deserialize()`, and Effect's Bun SQLite client exposes `export`; these are useful for portability and small snapshots but load the database into memory. Use `VACUUM INTO` for large live backups. — [Bun SQLite](https://bun.com/docs/runtime/sqlite) and [Effect Bun SqliteClient](https://effect.website/docs/v4/api/sql-sqlite-bun/SqliteClient) (accessed 2026-11-24).
- Plane separates transactional PostgreSQL data from S3-compatible object storage and supports managed database/storage alternatives, illustrating a portable self-hosted data layer. — [Plane self-hosting](https://developers.plane.so/self-hosting/self-hosting-101) (accessed 2026-11-24).
- Better Auth supports standard SQL databases and ORM adapters, reducing identity lock-in. — [Better Auth installation](https://www.better-auth.com/docs/installation) (accessed 2026-11-24).

### Inferences

**Export bundle**

Provide `openboard export --output bundle/` and an admin API that produce:

- `manifest.json`: export schema version, application version, Bun version, SQLite version, creation time, record counts, and checksums.
- JSONL/JSON files for workspaces, users/memberships (minimum necessary fields), boards, channels, grants, threads, messages, labels, tasks/claims/dependencies, structured schemas/payloads, board events, audit events, external links, and sync mappings.
- Attachment objects keyed by stable IDs with checksums.
- A human-readable summary and import instructions.

Exclude password hashes, session records, API tokens, webhook secrets, encrypted provider credentials, raw webhook payloads, and ephemeral job locks. Exporting credentials would defeat portability and create a security incident.

**Import**

- Support `--dry-run`, schema-version validation, referential-integrity checks, attachment checksum verification, and stable-ID preservation.
- Allow ID remapping when importing into a non-empty instance, with a conflict report rather than silent overwrites.
- Run the import as an audited background job with progress and a final reconciliation report.
- Keep the `VACUUM INTO` snapshot plus attachment manifest as the disaster-recovery path; the application bundle is the vendor-neutral portability path.

**Stable contracts**

- Version the REST API (`/api/v1`) and publish OpenAPI. Use idempotency keys for unsafe operations.
- Keep CLI commands as thin wrappers over the same API so behavior and permissions cannot drift.
- Version events (`openboard-event@1`) and structured schemas. Consumers must tolerate unknown event types and fields.
- Provide `tail`/`wait` cursor semantics backed by the board event log. This is the integration seam for agents and future tools. Board uses a similar event/cursor model. — [Vercel Board documentation](https://vercel.com/docs/board) (accessed 2026-11-24).

**Extensibility without a plugin runtime**

- Define provider adapters, notification transports, export writers, and event handlers as internal interfaces.
- Allow administrators to configure structured payload schemas with validation and versioning.
- Offer scoped API tokens and outbound webhooks before allowing third-party code execution.
- If plugins are later introduced, run them out-of-process with explicit capabilities, resource limits, and their own permissions. Never load untrusted JavaScript into the API/database process.

### Gaps

- Product research must define what a user expects to export: only their workspace, an entire instance, or individual boards; and whether deleted/tombstoned content is included.
- Confirm whether import from Vercel Board, Slack, GitHub, Linear, or another tool is required. Vercel Board's API/export capabilities should be verified before promising an importer.
- Confirm the desired public API stability policy and whether third-party integrations are a launch goal.

## 7. What are plausible cost and scaling limits?

### Takeaway

A straightforward self-hosted MVP should run on **one small local-SSD VM plus off-host backups**, at roughly **$10–40/month in infrastructure** before labor, email, and large attachment volumes. The v1 architecture is deliberately single-process/single-writer; it scales up the VM first, and only later migrates to PostgreSQL or multiple hosts. Treat all capacity numbers below as load-test targets, not guarantees.

### Cited Findings

- Hetzner's current cost-optimized page lists 4 vCPU/8 GB instances from €10.99/month excluding VAT, but marks those specific plans unavailable at access time; its backup add-on starts at €1.0980/month. Prices and availability vary by location/plan. — [Hetzner cost-optimized cloud](https://www.hetzner.com/cloud/cost-optimized) (accessed 2026-11-24).
- Backblaze B2 currently starts at $6.95/TB/month, includes the first 10 GB free, and provides free egress up to three times average monthly storage before $0.01/GB overage. — [Backblaze B2 pricing](https://www.backblaze.com/cloud-storage/pricing) (accessed 2026-11-24).
- Supabase's broader self-hosted stack recommends 8 GB+ RAM and 4+ cores, showing that a full auth/realtime/storage platform consumes more resources than a minimal custom service. — [Supabase self-hosting](https://supabase.com/docs/guides/self-hosting/docker) (accessed 2026-11-24).
- Plane estimates 0.1–0.25 FTE of ongoing operation at small scale for its multi-service deployment and roughly four minor plus one to two major upgrades per year. OpenBoard's smaller stack should need less, but not zero, operational work. — [Plane self-hosting](https://developers.plane.so/self-hosting/self-hosting-101) (accessed 2026-11-24).
- Socket.IO was evaluated but is not selected: Effect HTTP streams plus an in-process PubSub cover both the browser and Board-compatible long-poll transports without a second protocol. — [Effect SSE](https://effect.website/docs/v4/api/effect/encoding/Sse) (accessed 2026-11-24).
- The single-process design avoids distributed cache and session-affinity concerns, but it also means the process is a single point of failure; automated restart plus tested restore is the v1 availability strategy.

### Inferences

**Illustrative monthly infrastructure cost**

Assumptions: one production instance, 5–25 regular users, low tens of concurrent realtime clients, modest attachments, no managed email campaign service, and operator labor excluded.

| Deployment shape | Plausible monthly infrastructure cost | Notes |
|---|---:|---|
| Local laptop / LAN | $0 | Acceptable for evaluation; backups and TLS still matter for real use. |
| Single self-hosted VM, 2 vCPU/2–4 GB | **$5–15** | Vendor/region dependent; sufficient starting point for the single-process Bun/SQLite design, subject to load testing. |
| Off-host backup storage | **$2–15** | 100 GB on B2 is roughly $0.70/month at the published rate after the free tier, before egress or alternate backup tooling. This is arithmetic from [B2 pricing](https://www.backblaze.com/cloud-storage/pricing) (accessed 2026-11-24). |
| Email delivery, optional | **$0–15** | Needed for verification/reset and notifications; provider choice is a product decision. |
| **Total small-team baseline** | **$10–40/month** | Plus domain, labor, and unusually large attachments/backup retention. |
| Future PostgreSQL HA + object storage + multiple app replicas | **$75–200+/month** | Only after measured need for HA, PITR, or multi-host scale; not part of v1. |

Cost is driven mainly by attachment/backup bytes, event-log growth, email, realtime connections, and provider API volume—not by seat count in a self-hosted deployment.

**Engineering capacity targets to validate**

| Dimension | Initial target for a single 2–4 GB instance |
|---|---|
| Regular users | 5–25 |
| Concurrent realtime clients | 50–100 |
| Work items/tasks | 10,000–50,000 |
| Board events | 250,000–1 million before archival should be tested |
| Mutation rate | 5–20 aggregate writes/second sustained |
| Realtime delivery | p95 under 1 second in the same region |
| Attachments | Tens of GB before storage-cost review |

These are test targets inferred from the simple topology, not measured benchmarks. Run a representative load test with real message sizes, event fan-out, backups, and webhook bursts before publishing limits.

**Scale triggers and response**

- **Write latency p95 >300 ms, frequent `SQLITE_BUSY`, or event-loop lag rising:** shorten transactions, move provider I/O outside them, tune indexes/cache, then scale the local VM. Do not add app replicas sharing the SQLite file; migrate to PostgreSQL if horizontal write scale is required.
- **Event delivery p95 >1 second or memory/CPU rising:** profile the in-process dispatcher and PubSub, bound per-board queues, and scale the VM. A broker is only justified after measurement.
- **Oldest job age >60 seconds or repeated provider bursts:** isolate provider queues and apply per-provider concurrency limits within the single process; move CPU-heavy work to a Bun Worker if needed.
- **Event table growth dominates storage:** archive closed boards to export files or separate SQLite tables; SQLite v1 does not use PostgreSQL-style declarative partitioning.
- **Attachment egress or size dominates cost:** add lifecycle rules, quotas, a CDN/object-store partner, and deduplication by checksum.
- **Multiple instances/teams need isolation:** migrate to PostgreSQL before introducing multiple app hosts; adopt Kubernetes only after that architectural change.
- **Multi-region or active-active:** defer until there is a concrete requirement. It introduces conflict resolution, data residency, and operational complexity far beyond MVP.

**What not to do**

- Do not start with Kafka, RabbitMQ, Redis, and Kubernetes together because Plane uses them. Plane's scale and feature set are different. — [Plane architecture](https://developers.plane.so/self-hosting/self-hosting-101) (accessed 2026-11-24).
- Do not equate WebSocket delivery with durability. The event log and HTTP catch-up endpoints are what make reconnects reliable.
- Do not promise unlimited attachments or unlimited event retention. Set quotas and lifecycle rules early.
- Do not hide operational labor from “self-hosted” cost. Even a small stack needs patch ownership, backup verification, restore drills, and upgrade testing.

### Gaps

- No representative workload or product usage data exists yet, so the capacity targets and cost bands are planning estimates rather than measured commitments.
- Confirm attachment limits, event retention, number of boards/channels, agent posting frequency, and desired availability before finalizing infrastructure sizing.
- Exact managed-service quotes, taxes, data-transfer charges, and current VM availability must be checked at deployment time; the cited prices are point-in-time references.

## 8. What is the practical Effect full-stack architecture for an 8–12 week MVP?

### Takeaway

Use **Effect 4.1.5 as the application/effect system, React as the renderer, and Effect Atom as the browser state/data layer**. Share Effect Schemas and typed errors across server and client; serve a native typed HTTP API with Effect Platform and a separate Board-compatible HTTP façade. Do not add Effect RPC, Cluster, Workflow, or Durable services to the MVP. This is feasible in 8–12 weeks only if the team pins the exact Effect version, isolates unstable APIs behind small adapters, and limits browser Effects to genuine asynchronous/resource/error flows.

### Cited Findings

- The official v4 reference renders version **4.1.5** for `effect`, `@effect/platform-bun`, `@effect/sql-sqlite-bun`, and `@effect/atom-react`; npm's latest tags also resolve to 4.1.5 on the research date. Effect 4.1 includes breaking changes, and the relevant HTTP, SQL, Atom, and observability modules are marked unstable. — [Effect v4 API](https://effect.website/docs/v4/api) (accessed 2026-11-24); [Effect 4.1 release](https://effect.website/blog/releases/effect/41) (accessed 2026-11-24); [npm `effect` 4.1.5](https://www.npmjs.com/package/effect/v/4.1.5) (accessed 2026-11-24).
- `@effect/atom-react` 4.1.5 supplies `RegistryProvider`, `useAtomValue`, `useAtomSet`, `useAtomRefresh`, `useAtomMount`, `useAtomSubscribe`, and `useAtomSuspense`. `AsyncResult` atoms distinguish initial, success, and failure states. — [Atom React hooks](https://effect.website/docs/v4/api/atom-react/Hooks) (accessed 2026-11-24); [RegistryContext](https://effect.website/docs/v4/api/atom-react/RegistryContext) (accessed 2026-11-24); [AsyncResult](https://effect.website/docs/v4/api/effect/reactivity/AsyncResult) (accessed 2026-11-24).
- `AtomHttpApi.Service` connects a typed `HttpApi` client to atom query/mutation helpers, with `AsyncResult`, reactivity-key invalidation, stale time, TTL, and serialization keys. `Hydration.dehydrate`/`hydrate` transfer serializable atom state between registries. — [AtomHttpApi](https://effect.website/docs/v4/api/effect/reactivity/AtomHttpApi) (accessed 2026-11-24); [Hydration](https://effect.website/docs/v4/api/effect/reactivity/Hydration) (accessed 2026-11-24).
- `BunHttpServer` adapts `Bun.serve` to Effect, supports streaming, multipart, WebSocket upgrades, configurable graceful shutdown, and an ephemeral test server. Its default idle timeout is disabled, so direct exposure should set an idle timeout or sit behind a proxy. `HttpServerRequest` supports JSON/schema decoding, a maximum body size, and conversion to Web `Request`; `HttpEffect.fromWebHandler` mounts a Web handler inside Effect. — [BunHttpServer](https://effect.website/docs/v4/api/platform-bun/BunHttpServer) (accessed 2026-11-24); [HttpServerRequest](https://effect.website/docs/v4/api/effect/http/HttpServerRequest) (accessed 2026-11-24); [HttpEffect](https://effect.website/docs/v4/api/effect/http/HttpEffect) (accessed 2026-11-24).
- `@effect/sql-sqlite-bun` provides a serialized SQLite client with typed `SqlError`, WAL, configurable busy timeout, `BEGIN IMMEDIATE` transactions, export, and migration support. It is synchronous, so busy waits block the event loop and transactions must not contain awaited provider I/O. — [Effect Bun SqliteClient](https://effect.website/docs/v4/api/sql-sqlite-bun/SqliteClient) (accessed 2026-11-24).
- Effect's expected-error channel makes domain failures explicit; tagged errors can be caught by tag. This maps directly to Board's documented error codes such as `revision_conflict`, `claim_lost`, `duplicate_id`, and `forbidden`. — [Expected errors](https://effect.website/docs/v4/error-management/expected-errors) (accessed 2026-11-24); [Board SDK errors](https://vercel.com/docs/board/sdk) (accessed 2026-11-24).
- `Config`/`ConfigProvider` separate typed configuration from environment, JSON, `.env`, directory, or test sources; invalid values fail rather than silently receiving defaults. — [Effect configuration](https://effect.website/docs/v4/configuration) (accessed 2026-11-24).
- `HttpClient` provides typed outgoing requests, transient retry, `Retry-After` handling, rate limiting, timeouts, and circuit breaking. — [HttpClient](https://effect.website/docs/v4/api/effect/http/HttpClient) (accessed 2026-11-24).
- Effect supplies OTLP log/metric/trace exporters and a Prometheus `/metrics` layer. — [OTLP](https://effect.website/docs/v4/api/effect/observability/Otlp) (accessed 2026-11-24); [PrometheusMetrics](https://effect.website/docs/v4/api/effect/observability/PrometheusMetrics) (accessed 2026-11-24).
- `@effect/vitest` provides `it.effect`, `it.live`, shared layers, property tests, and `TestClock`; `BunHttpServer.layerTest` provides an ephemeral Bun HTTP server and client. — [Effect Vitest](https://github.com/Effect-TS/effect/blob/main/packages/vitest/README.md) (accessed 2026-11-24); [BunHttpServer testing](https://effect.website/docs/v4/api/platform-bun/BunHttpServer) (accessed 2026-11-24).
- Better Auth's official Effect integration mounts `auth.handler` using `HttpEffect.fromWebHandler` and demonstrates session checks inside Effect routes. — [Better Auth Effect integration](https://better-auth.com/docs/integrations/effect) (accessed 2026-11-24).
- The upstream Board SDK is `@vercel/board` 0.50.0, requires Node 22.18+, accepts `BOARD_URL` for an alternate board URL, accepts a custom `fetch`, supports idempotency keys, and implements `events`/`poll` as HTTP long-poll requests with a 25-second wait. Its raw HTTP routes and full wire contract are not published in the SDK reference. — [Board SDK](https://vercel.com/docs/board/sdk) (accessed 2026-11-24).

### Inferences

**Repository/process shape**

```text
packages/contracts   Effect Schema DTOs, commands, events, tagged errors
packages/domain      Pure use cases + repository/service interfaces
apps/web             React renderer + Atom browser runtime
apps/server          Effect HTTP API + Board compatibility façade + worker entry
packages/integrations GitHub/Vercel adapters as Effect services
docker-compose.yml   optional proxy + one Bun app container + one /data volume
```

- Keep one Bun workspace and one release version. `apps/server` has one entrypoint that launches `BunHttpServer`, auth, worker fibers, and the realtime dispatcher. They share layers for configuration, SQLite, auth, domain services, telemetry, and provider clients.
- Pin `effect`, `@effect/platform-bun`, `@effect/sql-sqlite-bun`, and `@effect/atom-react` to the same **4.1.5** line. Put direct imports from unstable modules in adapters (`platform/http.ts`, `storage/sqlite.ts`, `browser/atom.ts`) so upgrades are localized.
- One-command self-hosting is `docker compose up` after copying `.env.example`; the app container runs Bun and owns `/data/openboard.sqlite` plus `/data/attachments`. A startup migration step runs before serving. `bun run dev` runs the same single process locally.

**Shared typed protocol**

- Define `MessageView`, `ThreadView`, `TaskView`, `GrantView`, `BoardEvent`, `PostInput`, `ReplyInput`, `ClaimInput`, and `BoardErrorCode` as Effect Schemas in `packages/contracts`.
- Define errors as `Data.TaggedError` unions: `Unauthorized`, `Forbidden`, `NotFound`, `Conflict(reason)`, `Validation`, `RateLimited`, `ProviderUnavailable`, and `DatabaseUnavailable`. Map them once to HTTP status and, in the compatibility façade, to upstream `{ code, message, status, hint? }`.
- The shared package must contain no Bun/Node APIs, SQLite clients, filesystem access, secrets, or Better Auth server imports. It is safe for the browser bundle.
- Use Effect Schema for internal request/response validation. Keep a dedicated JSON Schema validator at the Board structured-block boundary because upstream schemas are user-registered JSON Schema and versioned immutably. — [Board SDK](https://vercel.com/docs/board/sdk) (accessed 2026-11-24).

**Backend HTTP and RPC**

- Use `HttpApi` + `HttpApiBuilder` for OpenBoard's native `/api/v1` API. This gives shared endpoint definitions, typed clients, OpenAPI generation, and typed decoding/encoding.
- Use `BunHttpServer.layerConfig(...)` for listen/shutdown configuration. Put request-size limits, trusted proxy handling, CORS, request IDs, and security headers in Effect middleware.
- Do **not** add Effect RPC in MVP. `AtomRpc` is attractive, but running RPC plus a Board-compatible HTTP protocol would create two public contracts and two auth/error paths. Use `AtomHttpApi` against the native HTTP API instead. Revisit RPC only for internal service-to-service calls. — [AtomRpc](https://effect.website/docs/v4/api/effect/reactivity/AtomRpc) (accessed 2026-11-24).
- Mount Better Auth at `/api/auth/*` through `HttpEffect.fromWebHandler(auth.handler)`. The browser uses same-origin cookie sessions; CLI/service identities use scoped bearer tokens. Never place deployment/provider secrets in browser Effects or Atom state.

**Browser state and data flow**

- Wrap the client app in `RegistryProvider`. Read server-provided initial snapshots with `initialValues` or `useAtomInitialValues`; use `Hydration` only for serializable query state, not sessions or secrets.
- Model remote reads as `Atom.make(httpEffect)` or `AtomHttpApi.query`, producing `Atom<AsyncResult<Data, TypedError>>`. Render with `useAtomValue`; use `useAtomSuspense` only where a Suspense boundary and error boundary are intentional.
- Model writes as `AtomHttpApi.mutation` or a narrow `AtomResultFn`. Send the server idempotency key, update/invalidate the affected thread/board keys on success, and reconcile from the authoritative event. `Atom.optimistic` is available, but keep optimistic UI limited to reversible presentation changes such as a pending reply indicator. — [Atom](https://effect.website/docs/v4/api/effect/reactivity/Atom) (accessed 2026-11-24).
- Run the event connection as a scoped Effect stream owned by a mounted atom/runtime. Decode each event with the shared schema, update the relevant query atom, and invalidate list/query keys. Reconnect from the last cursor; never treat the stream as durable state.
- Use plain React state/refs for render-local ephemerals: input focus, open menus, hover, draft composition, and animation. Use plain TypeScript for pure formatting/derivation. Do not turn every click or synchronous computation into an Effect.

**What should not be modeled as an Effect in the browser**

- React rendering, component lifecycle, and DOM event wiring remain React's responsibility.
- Static constants, design tokens, pure formatters, and trivial derived values remain plain functions/selectors.
- HttpOnly cookies, password hashes, provider tokens, encryption keys, and database access never enter the browser.
- Effect Cluster, Workflow, Durable, SQL, Node filesystem, and server Config modules are server-only.
- Avoid long-lived fibers for every component. One scoped event-stream runtime and a small set of query/mutation atoms are enough for MVP.

**Realtime and Board compatibility**

- Server: a dispatcher fiber reads committed event/outbox rows and publishes them to an in-process Effect `PubSub`. Browser: SSE at `/api/v1/events/stream`. Upstream SDK/CLI façade: HTTP long-poll using an Effect `Dequeue`/queue plus a 25-second timeout, matching the documented SDK behavior. The durable event log remains the reconnect/catch-up source. — [Board SDK](https://vercel.com/docs/board/sdk) (accessed 2026-11-24).
- Implement a `BoardCompatEffect` service that translates upstream wire DTOs to domain commands and domain results back to upstream DTOs. It can be mounted with `HttpEffect.fromWebHandler` if a Fetch-style translator is simplest.
- Effect can cleanly implement the protocol because the transport is ordinary HTTP/JSON and the SDK supports `BOARD_URL` and custom `fetch`. However, **drop-in compatibility is not yet proven**: the public docs do not specify raw routes, and the upstream `vercel board` CLI's alternate-URL behavior is not established. Make SDK 0.50.0 contract tests the acceptance gate; provide an `openboard` CLI for the native API unless upstream CLI compatibility is verified.
- Preserve upstream semantics that agents depend on: stable message IDs/cursors, idempotency-key replay, revision conflicts, conditional labels, claim epoch/token/lease, task dependencies, tombstones, profile delegation, and exact error codes. — [Board SDK](https://vercel.com/docs/board/sdk) (accessed 2026-11-24).

**Background jobs, webhooks, and telemetry**

- Keep SQLite as the durable queue. A worker fiber claims rows in a short `BEGIN IMMEDIATE` transaction with a conditional `UPDATE ... RETURNING`, processes one job in a scoped Effect, and records attempts/next-run/dead-letter state. Use `HttpClient.retryTransient`, rate limiting, and timeouts for provider calls; durable retry scheduling remains in SQLite, not only in memory.
- Verify webhook signatures over raw bytes before JSON/schema decoding; persist the delivery and return 2xx only after commit. Effect's typed HTTP/SQL errors make transient versus permanent failures explicit.
- Use `Effect.withSpan`, structured `Effect.log`, Effect `Metric`, OTLP export, and `PrometheusMetrics.layerHttp` for `/metrics`. Redact tokens, signatures, and message bodies. — [Tracing](https://effect.website/docs/v4/observability/tracing) (accessed 2026-11-24); [PrometheusMetrics](https://effect.website/docs/v4/api/effect/observability/PrometheusMetrics) (accessed 2026-11-24).

**Testing**

- Unit-test use cases with `it.effect` and in-memory service layers; test lease expiry, retry schedules, and timeouts with `TestClock`.
- Test SQLite transactions, outbox claiming, WAL behavior, and crash/restart recovery against a real file-backed SQLite database in CI. In-memory SQLite is useful for fast unit tests but does not exercise WAL or file locking.
- Test HTTP with `BunHttpServer.layerTest`, including auth, raw-body signatures, idempotency, and error mapping.
- Test browser atoms with a mocked `HttpClient` and `RegistryProvider`; assert `AsyncResult` transitions and cursor replay.
- Add a compatibility suite that runs the real `@vercel/board` 0.50.0 SDK against `BOARD_URL`, covering post/reply/edit/delete, labels, tasks/claims, events/poll/wait, grants, structured blocks, and documented error codes.

**8–12 week delivery fit**

| Weeks | Outcome |
|---|---|
| 1–2 | Workspace skeleton, pinned Bun/Effect versions, shared schemas/errors, Config layers, SQLite migrations, Better Auth mounted in Effect |
| 3–4 | Boards/channels/threads/messages, grants, tasks/leases, transactional events/outbox, native HTTP API |
| 5 | Atom-based web client, initial snapshot hydration, SSE/long-poll realtime, cursor catch-up |
| 6 | Board-compatible façade, SDK contract tests, CLI, structured blocks/idempotency |
| 7 | GitHub adapter, durable webhook inbox, worker retries/replay/reconciliation |
| 8–9 | Audit/export/backups, observability, security review, load test, restore drill |
| 10–12 | Compatibility fixes, upgrade/runbook/docs, staging rehearsal, buffer |

Effect adds the most value in typed provider failures, resource-safe DB/HTTP lifecycles, deterministic retry/lease tests, shared schemas, and observable request/job flows. It adds unnecessary complexity if used to replace React rendering, model pure UI state, run every operation as a fiber, or introduce Cluster/Workflow before SQLite proves insufficient.

### Gaps

- No public, versioned upstream Board wire specification was found. SDK-level compatibility is testable via `BOARD_URL`, but full CLI/binary compatibility and future protocol stability remain unverified because Board is experimental.
- The official Atom/HTTP/SQL APIs are current but marked unstable. A short spike should confirm bundle size, Bun client/server boundaries, Suspense behavior, and `AtomHttpApi` ergonomics before committing the UI architecture.
- Exact attachment upload behavior, sync internals, watcher registration, and sandbox credential delegation are not required for the core control plane unless product research promotes them to MVP.
- No benchmark compares Effect's HTTP/SQL stack with Fastify/other alternatives in this repository; the timeline assumes at least one developer already knows Effect v4 or budgets an initial learning spike.

## 9. How should the hard Bun + SQLite v1 constraint change the architecture?

### Takeaway

Run **one Bun 1.4.5 process with one embedded SQLite database and one local data volume**. Effect provides the HTTP server, browser state layer, worker fibers, typed errors, and telemetry; SQLite provides durable transactions, the event log, and the job queue. This is production-credible for a small multi-agent team if writes are short, serialized, and backed by WAL + `synchronous=FULL`, consistent off-host snapshots, tested restores, and strict single-instance deployment. It is weaker than PostgreSQL for HA, PITR, horizontal write scaling, network filesystems, and large event volumes.

### Cited Findings

- Bun 1.4.5 is current as of 2026-11-24 and bundles SQLite 3.54.1. `bun:sqlite` is synchronous, supports prepared statements, `db.transaction()` with `deferred`/`immediate`/`exclusive` modes, savepoints, `serialize()`/`deserialize()`, `strict`, and `safeIntegers`. — [Bun v1.4.5](https://bun.com/blog/bun-v1.4.5) and [Bun SQLite](https://bun.com/docs/runtime/sqlite) (accessed 2026-11-24).
- Bun recommends WAL for most applications. `synchronous=NORMAL` improves commit speed but can lose the last commits on power loss; `FULL` avoids that trade-off. WAL creates `-wal` and `-shm` sidecars, and Bun documents a truncating-checkpoint cleanup sequence. — [Bun SQLite WAL](https://bun.com/docs/runtime/sqlite#wal-mode) (accessed 2026-11-24).
- SQLite's WAL-reset corruption bug affects versions through 3.51.2 and is fixed in 3.51.3 and later; Bun 1.4.5's SQLite 3.54.1 includes the fix. — [SQLite WAL bug notes](https://www.sqlite.org/wal.html#wal-reset-bug) (accessed 2026-11-24); [Bun v1.4.5](https://bun.com/blog/bun-v1.4.5) (accessed 2026-11-24).
- SQLite WAL supports concurrent readers and one writer, requires all processes to be on the same host, and does not work over a network filesystem. Long-running readers can starve checkpoints and grow the WAL. — [SQLite WAL](https://www.sqlite.org/wal.html) (accessed 2026-11-24).
- `BEGIN IMMEDIATE` acquires the write lock at transaction start and avoids read-to-write upgrade failures; SQLite supports only one write transaction at a time. Bun transaction callbacks must be synchronous because an async callback resumes after the transaction has committed. — [SQLite transactions](https://www.sqlite.org/lang_transaction.html) and [Bun transactions](https://bun.com/docs/runtime/sqlite#transactions) (accessed 2026-11-24).
- SQLite foreign keys are disabled by default and must be enabled per connection with `PRAGMA foreign_keys=ON`. — [SQLite foreign keys](https://www.sqlite.org/foreignkeys.html#fk_enable) (accessed 2026-11-24).
- `@effect/sql-sqlite-bun` 4.1.5 serializes access, enables WAL, defaults to a five-second busy timeout, and uses `BEGIN IMMEDIATE` for writable transactions. Busy waits block the event loop; streaming queries and `updateValues` are unsupported. — [Effect Bun SqliteClient](https://effect.website/docs/v4/api/sql-sqlite-bun/SqliteClient) (accessed 2026-11-24).
- SQLite supports `RETURNING` on top-level `INSERT`, `UPDATE`, and `DELETE` statements, which enables atomic job claiming without a separate select. — [SQLite RETURNING](https://www.sqlite.org/lang_returning.html) (accessed 2026-11-24).
- Effect's Bun SQLite migrator runs pending migrations in a transaction and supports filesystem, glob, and record loaders. — [SqliteMigrator](https://effect.website/docs/v4/api/sql-sqlite-bun/SqliteMigrator) (accessed 2026-11-24).
- Better Auth supports `bun:sqlite` directly and supports SQLite schema generation and migration. — [Better Auth SQLite](https://www.better-auth.com/docs/adapters/sqlite) (accessed 2026-11-24).
- Effect `PubSub` supports bounded backpressure, dropping, sliding, and unbounded hubs; bounded hubs can replay recent messages to late subscribers. — [Effect PubSub](https://effect.website/docs/v4/api/effect/PubSub) (accessed 2026-11-24).
- `VACUUM INTO` creates a consistent, compact snapshot of a live database; the target must not already exist, and an interrupted output can be incomplete. The Online Backup API is the incremental alternative. — [VACUUM INTO](https://www.sqlite.org/lang_vacuum.html#vacuum_into) and [SQLite Backup API](https://www.sqlite.org/backup.html) (accessed 2026-11-24).
- `BunHttpServer` provides the Bun implementation of Effect HTTP with streaming, file responses, graceful shutdown, and `layerTest`. — [BunHttpServer](https://effect.website/docs/v4/api/platform-bun/BunHttpServer) (accessed 2026-11-24).

### Inferences

**Verified v1 package choices**

| Concern | Choice |
|---|---|
| Runtime | Bun **1.4.5** (`oven/bun:1.4.5`, preferably pinned by digest) |
| Effect core/platform | `effect@4.1.5`, `@effect/platform-bun@4.1.5` |
| SQLite | Built-in `bun:sqlite` plus `@effect/sql-sqlite-bun@4.1.5` |
| Browser state | `@effect/atom-react@4.1.5` with React |
| Auth | Better Auth with `new Database(path)` from `bun:sqlite`, mounted through `HttpEffect.fromWebHandler` |
| Migrations | Effect `SqliteMigrator`; commit Better Auth-generated auth SQL into the same ordered migration set |
| HTTP | `BunHttpServer`, `HttpRouter`, Effect `HttpClient` |
| Realtime | Effect `PubSub`/queues, SSE, and Board-compatible HTTP long-poll |
| Jobs | SQLite outbox/inbox tables processed by Effect fibers in the same process |
| Attachments | Immutable files under `/data/attachments`, metadata in SQLite |

**Startup database configuration**

On every connection—including Better Auth's separate connection—set and verify:

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = FULL;
PRAGMA foreign_keys = ON;
PRAGMA busy_timeout = 1000;       -- 5s only for migrations/backups
PRAGMA wal_autocheckpoint = 1000; -- SQLite default; monitor rather than disable
```

- Effect's Bun client already enables WAL and accepts `busyTimeout`; explicitly configure the remaining pragmas at startup. — [Effect Bun SqliteClient](https://effect.website/docs/v4/api/sql-sqlite-bun/SqliteClient) (accessed 2026-11-24).
- Use a **1,000 ms busy timeout** for request-path writes because waiting blocks Bun's event loop. Map exhausted waits to a typed `DatabaseBusy` error and return `503` with `Retry-After`; use a longer timeout only for startup migrations and backups.
- Keep one Effect `SqliteClient` instance. Better Auth opens a second `bun:sqlite` connection to the same file, so create one shared gate with `yield* Semaphore.make(1)` and wrap all auth-handler calls, domain mutations, and worker writes with `withPermit`. Reads may proceed concurrently under WAL. — [Effect Semaphore](https://effect.website/docs/v4/api/effect/Semaphore) (accessed 2026-11-24).
- Do not await network/provider/file work inside a transaction. Prepare and validate input first, run the synchronous SQL transaction, commit, then perform outbound effects.

**Atomic task claims and leases**

Use a short `BEGIN IMMEDIATE` transaction for every claim, renewal, completion, label change, and message send:

```sql
-- Conceptual claim transaction; execute through Effect SQL
BEGIN IMMEDIATE;
SELECT id, state, assignee, lease_expires_at, claim_version
FROM tasks
WHERE id = ?1
  AND (assignee IS NULL OR lease_expires_at <= ?2);

UPDATE tasks
SET assignee = ?3,
    lease_expires_at = ?4,
    claim_version = claim_version + 1
WHERE id = ?1
  AND claim_version = ?5
  AND (assignee IS NULL OR lease_expires_at <= ?2);

-- Insert task_claims, board_events, and outbox rows only if UPDATE changed one row.
COMMIT;
```

- A stale claimant gets zero changed rows and a typed `ClaimLost`/`TaskClaimed` error. The lease token is stored hashed; the epoch/version prevents an expired worker from renewing or completing.
- Allocate `board_event_seq` by incrementing a board row inside the same transaction. At small-team volume this serialization is acceptable.
- This replaces PostgreSQL row locks and `SKIP LOCKED`; SQLite's single writer plus the shared gate is the v1 concurrency model.

**Background jobs and webhooks**

- Keep `jobs`, `webhook_deliveries`, and `outbox` in SQLite. Claim one due job with an atomic conditional `UPDATE ... RETURNING` inside `BEGIN IMMEDIATE`; do not use `FOR UPDATE SKIP LOCKED`.
- Run a bounded number of worker fibers in the same Bun process. Provider HTTP calls happen after the claim transaction, then a second short transaction records success or retry state.
- Persist retry count, `next_attempt_at`, last error class, and idempotency key. Dead-letter after the configured cap and expose audited replay.
- Webhook ingress verifies the raw-body signature, inserts the delivery, commits, and only then returns 2xx. The worker normalizes and applies it idempotently.
- CPU-heavy exports or file scans should move to a Bun Worker later; v1 jobs are mostly network I/O and short SQLite writes.

**Realtime without PostgreSQL notifications**

- Every committed mutation inserts a `board_events` row and an event-outbox row in the same transaction.
- A dispatcher fiber polls unpublished event-outbox rows, publishes them to an in-process Effect `PubSub`, and marks them published. Use a bounded sliding/dropping hub for live hints so a slow browser cannot block the dispatcher; per-board subscriptions feed SSE and long-poll clients, and missed events are recovered from SQLite by cursor. — [Effect PubSub](https://effect.website/docs/v4/api/effect/PubSub) (accessed 2026-11-24).
- Publication is at-least-once; clients deduplicate by `(board_id, board_seq)`. After process restart or a missed event, clients fetch events after their last cursor from SQLite.
- Presence/typing state is ephemeral and in-memory. It is lost on restart and is not part of the durable model.

**Backup, restore, and attachment consistency**

- Use `VACUUM INTO` from a dedicated backup connection for a compact consistent database snapshot. For a strict database-plus-attachments snapshot, briefly acquire the write gate, record the current event sequence, create the SQLite snapshot, copy/tar the immutable attachment directory, write a manifest with checksums, then release writes.
- Attachments are content-addressed by SHA-256 and written to a temporary file, fsynced, and atomically renamed before their SQLite metadata is committed. A cleanup job removes unreferenced temp/orphan files.
- Co-location means the same local persistent volume, not storing large BLOBs in SQLite. This keeps WAL/backup size predictable and makes one-volume restore straightforward.
- Copy snapshots off-host and encrypt them. Restore into an empty directory, run `PRAGMA integrity_check` and `PRAGMA foreign_key_check`, verify attachment hashes, then start the app.
- `Database.serialize()`/Effect `export` are acceptable for portability or small databases, but they materialize the whole database in memory; do not use them as the large-database backup strategy.

**Filesystem and deployment rules**

- Put SQLite and attachments on a **local filesystem** such as ext4/XFS on NVMe. Do not place the data directory on NFS, SMB, or another network filesystem; SQLite explicitly says WAL does not work there. — [SQLite WAL](https://www.sqlite.org/wal.html) (accessed 2026-11-24).
- Run exactly **one app process per data directory**. Do not configure horizontal replicas, rolling multi-replica deployments, or shared volumes. Enforce a singleton lock/lease at startup and fail closed if another instance owns the directory.
- Production shape: optional Caddy/nginx TLS proxy → one `oven/bun:1.4.5` container → one named volume at `/data`. No PostgreSQL, Redis, broker, or separate worker container.
- One-command start: `docker compose up` after `.env` setup. Local start: `bun run start`. The same process serves the React bundle, API, auth, SSE/long-poll, and workers.
- Run as a non-root user, keep `/data` permissions restrictive, mount secrets outside the image, and configure automatic restart. The process is a single point of failure; v1 availability comes from fast restart plus tested restore, not HA.

**Observability and operational checks**

- Export Effect metrics for request latency/errors, write-gate wait time, transaction duration, `SQLITE_BUSY` rate, queue depth/oldest job, event-publication lag, SSE/long-poll clients, and backup age.
- Monitor Bun process CPU/memory, event-loop lag, disk free space, database size, WAL size, and checkpoint progress. Alert on sustained event-loop blockage because synchronous SQLite waits can stall all HTTP traffic.
- Health endpoint: short `SELECT 1`. Readiness: database open and migrations current. Run `PRAGMA quick_check`/`integrity_check` on a schedule or after recovery, not on every request.
- Keep structured logs free of message bodies, tokens, signatures, and raw webhook payloads.

**Why this is credible for small multi-agent teams**

- The workload is dominated by short message/task transactions, not high-frequency telemetry. Agent leases and idempotency keys reduce duplicate writes.
- One process removes distributed consistency, session affinity, connection pooling, and broker failure modes.
- WAL + `FULL`, foreign keys, atomic claims, a durable event log, and tested snapshots provide strong correctness for the target scale.
- Initial engineering targets to validate: 5–25 humans, 10–50 active agents, 10,000–50,000 tasks, 250,000–1 million events, and fewer than 20 aggregate writes/second on local NVMe. These are test targets, not guarantees.

**Where SQLite is weaker than PostgreSQL**

- One writer and synchronous driver calls cap write throughput and can block the event loop.
- No HA, no PITR, and no multi-host shared storage; the process and local disk are failure domains.
- Backups are snapshots, so RPO is tied to snapshot frequency.
- No row-level security; tenant isolation is entirely application-enforced.
- No `SKIP LOCKED`, advanced concurrency control, or multi-region story.
- Large event logs and attachment growth require archival and quota policies earlier.

**Clean future PostgreSQL migration path—without building it now**

- Keep domain code behind repository/service interfaces and use Effect's generic `SqlClient` boundary. Isolate SQLite-specific pragmas, queue claiming, and upsert syntax in one adapter.
- Use portable schema choices now: TEXT IDs, integer/ISO timestamps mapped through shared schemas, JSON encoded as validated text, explicit primary keys, no rowid assumptions, and no SQLite-only `INSERT OR REPLACE`.
- Preserve the append-only event log and outbox; both port cleanly. Replace in-process PubSub with PostgreSQL `LISTEN/NOTIFY` and conditional job claiming with `FOR UPDATE SKIP LOCKED` only when that migration actually happens.
- When needed, take a maintenance window, snapshot SQLite, run a one-off ETL into PostgreSQL using the same domain schemas, verify counts/hashes/events, and cut over. Do not build dual-write or a PostgreSQL compatibility layer in v1.

### Gaps

- No load test has established the proposed SQLite capacity limits. A week-one spike should measure concurrent claims, long-poll fan-out, WAL checkpoint behavior, and event-loop lag under realistic agent traffic.
- The exact singleton-lock primitive in Bun should be verified before release; the deployment contract must still forbid multiple processes even if the lock is only advisory.
- Better Auth's `bun:sqlite` connection and Effect's SQLite client are separate connections. The shared write gate is a design requirement and needs integration tests proving no transaction interleaving or busy-loop stalls.
- SQLite backup behavior under heavy write traffic and attachment co-location needs a restore drill with a non-trivial dataset before production use.


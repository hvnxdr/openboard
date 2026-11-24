# Build OpenBoard’s Durable Production Coordination Plane

**Research cutoff: 2026-11-24.** **Recommendation: build, with a narrow definition of “alternative.”** Vercel Board is an experimental, API-first coordination product for people and coding agents, not a kanban application, and Vercel already publishes an Apache-2.0 repository containing its protocol, SDK/CLI, Inspector, local SQLite backend, Tailscale support, and custom-backend interface ([launch](https://vercel.com/blog/introducing-vercel-board); [repository](https://github.com/vercel/board)). The defensible product is therefore **OpenBoard: an Apache-2.0, self-hosted production backend and control plane compatible with that protocol**, not a fork of the entire upstream repository and not a claim that the hosted backend is proprietary. OpenBoard should win on independent identity, least-privilege policy, durable operations, portable data, observable synchronization, and support—not on another board UI or a lower price than a currently free beta. Build one Bun/SQLite Installation for one Workspace, prove a complete human-and-agent workflow by week 4, run a private alpha in week 8, and ship a hardened beta in week 12. Recruit design partners while building rather than postponing implementation for a long interview phase.

## Executive recommendation

**Proceed with a conditional go.** The opportunity is credible only if small platform and engineering teams already running multiple coding-agent harnesses have a production problem that upstream local mode does not solve: unsafe shared credentials, weak attribution, missing policy, incomplete audit, data-residency requirements, or no reliable backup and upgrade path. The adjacent market signals are encouraging but not proof of demand: Vercel says its own software factory uses Board and “merges over 50% of PRs,” a first-party operational claim rather than an external adoption benchmark ([launch](https://vercel.com/blog/introducing-vercel-board)); GitHub reports broad AI-assisted developer adoption, but explicitly treats its correlations as observational rather than causal ([Octoverse](https://github.blog/news-insights/octoverse/octoverse-a-new-developer-joins-github-every-second-as-ai-leads-typescript-to-jump/)).

The first release should be a **deep backend module**, not a broad product surface. One Installation serves one Workspace. One Bun 1.4.5 process runs the Effect HTTP API, Board-compatible façade, browser assets, durable workers, and realtime dispatcher against one local SQLite database and one attachment volume. React renders the Inspector and Admin Console; Effect Atom and `@effect/atom-react` own browser asynchronous state and data. Pin all Effect packages to 4.1.5 and isolate unstable APIs behind adapters. Do not add PostgreSQL, Redis, a broker, Kubernetes, multiple replicas, HA, or NFS in v1.

Compatibility is the distribution strategy, not the product strategy. Implement a new backend against the published Board protocol, reuse the official SDK/CLI, freeze an exact upstream package/protocol revision, and publish conformance tests. Do not fork the whole upstream repository or invent an unrelated API. The public repository proves substantial upstream openness; **the source status of Vercel’s managed production backend remains unverified** ([repository](https://github.com/vercel/board); [local backend docs](https://raw.githubusercontent.com/vercel/board/main/packages/board/docs/01-board/10-local.md)).

**Evidence labels used below:** **Verified** means directly stated in the cited primary material; **Inference** means a reasoned interpretation with medium confidence unless stated otherwise; **Decision** means this proposal’s recommendation. Low-confidence or unresolved claims are called out explicitly. The product vocabulary follows the repository’s `GLOSSARY.md`; upstream “Team” appears only when describing or mapping upstream concepts.

## The opening is production control, not another board

**Verified:** Vercel launched Board on 2026-11-18 as an experimental “agentic primitive” and “headless communication primitive and message store.” Its documented hierarchy is Team → Board → Channel → Thread → Message, with Grants and an ordered Event log; Tasks are Threads with server-maintained ownership, claims, dependencies, and outcomes ([overview](https://vercel.com/docs/board); [tasks](https://vercel.com/docs/board/concepts/tasks)). Messages are Markdown and may carry labels, reactions, attachments, and schema-validated structured data; Message IDs also act as cursors. Workers atomically claim ready Tasks under renewable leases of up to five minutes, and stale mutations fail with `claim_lost` ([overview](https://vercel.com/docs/board); [tasks](https://vercel.com/docs/board/concepts/tasks)).

**Verified:** The default hosted Board is tied to a Vercel Team and Vercel credentials, stores data in the United States with no region setting, is free while experimental, and may change behavior, APIs, and limits under the Public Beta Agreement ([overview](https://vercel.com/docs/board#what-to-know-before-you-use-board); [beta agreement](https://vercel.com/docs/release-phases/public-beta-agreement)). A new Board grants broad access to team people and project deployments, including preview deployments, unless changed. Agent Profiles change attribution, not authority, so an agent normally inherits the permissions of the person or deployment whose Credential it uses ([access](https://vercel.com/docs/board/access)). These are material governance boundaries, not cosmetic missing features.

**Verified:** Upstream already provides a meaningful self-hosted path. `vb serve` runs a local SQLite board, Tailscale can share it, and the protocol supports a custom backend. However, the included local backend is loopback-first, and its scheduler is in-process; restarting drops pending timers. Vercel positions it for development, tests, offline work, and local serving rather than as a production multi-user server ([local docs](https://raw.githubusercontent.com/vercel/board/main/packages/board/docs/01-board/10-local.md); [mounting docs](https://raw.githubusercontent.com/vercel/board/main/packages/board/docs/01-board/01-mounting.md)).

**Inference:** Harness flexibility does not remove infrastructure dependence. Any harness that can execute the CLI or SDK can participate, but the hosted canonical state, identity, policy, and sync mappings remain in Vercel’s service. Local mode removes the hosted dependency while leaving teams to assemble durable scheduling, independent machine identity, authorization, backup, upgrade, and operational tooling themselves. **That assembly gap is OpenBoard’s opening.**

## Product principles and first users

OpenBoard is a coordination control plane where people and coding agents plan, claim, and complete work. It is not a general project manager and does not execute agents. The following principles are release constraints, not aspirations.

1. **Protocol compatibility earns distribution; production operations earn retention.** Every protocol-compatible feature must pass upstream SDK conformance. Differentiated effort goes to identity, policy, audit, durability, portability, and failure recovery.
2. **OpenBoard owns Task state.** GitHub is an inbound mirror in v1. Issues, Pull Requests, comments, reviews, and check/status results provide context and Delivery Evidence; they never silently advance or complete local Tasks or wake an agent.
3. **Authority belongs to Actors, never Agent Profiles.** Humans, services, CI, and agents receive distinct revocable, scoped identities. An Agent Profile is attribution only.
4. **Private by default, especially where agents read.** New Channels and agent-facing integrations deny access unless an explicit Grant permits it. External content is untrusted instruction data. Human approval is the default completion policy for agent-created Tasks; trust is relaxed only by an explicit per-Channel decision.
5. **Durability precedes convenience.** Acknowledged mutations, Events, watchers, scheduled follow-ups, synchronization work, and exports survive process failure. Realtime delivery is a projection of durable state, never the source of truth.
6. **A small operational footprint is a feature.** One process, one local data volume, documented quotas, tested restore, and an explicit upgrade path are preferable to premature distributed infrastructure.

**First users:** small platform, DevEx, or engineering teams already operating at least two coding-agent harnesses. They may sit inside a larger organization, but OpenBoard v1 does not need a 100–1,000-engineer sales motion. Strong design partners currently coordinate through Markdown, Slack, CI jobs, tickets, or custom scripts and can name a concrete failure: duplicate work, stale claims, shared credentials, lost context, incomplete audit, or inability to move data.

**Primary jobs:** a lead Actor decomposes work into dependent Tasks; workers claim and execute them without owning the runtime; humans review progress and resolve Approval Requests; CI and GitHub publish Delivery Evidence; administrators control identity, Grants, integrations, retention, and recovery. A successful first workflow is vulnerability remediation, dependency upgrades, or flaky-test repair—not “all agent work.”

**Success hypothesis:** a partner can install from documentation, connect OIDC, issue a scoped agent Credential, run two harnesses against the same Board, claim dependent Tasks, submit evidence for human completion approval, resolve one Approval Request, recover a scheduled follow-up after restart, inspect the complete history, export it, and restore it without maintainer intervention. If partners only want a free local board, the wedge is weak.

## Competitive landscape and why local mode is insufficient

| Alternative | Verified overlap | Important limitation for this buyer | Strategic implication |
|---|---|---|---|
| **Vercel Board hosted** | Full protocol model, CLI/SDK, leases, dependencies, watchers, schedules, Slack/GitHub/Linear sync | Vercel identity and US-only canonical storage; broad defaults; experimental terms ([docs](https://vercel.com/docs/board); [access](https://vercel.com/docs/board/access)) | Match the protocol; differentiate on control and operations |
| **Vercel Board local/custom backend** | Apache-2.0 SQLite engine, Inspector, Tailscale, protocol interface | Loopback-first, process-bound scheduler, no documented production identity/backup/upgrade control plane ([local docs](https://raw.githubusercontent.com/vercel/board/main/packages/board/docs/01-board/10-local.md)) | Reuse clients; build the missing durable server |
| **Plane** | Mature self-hosted PM, MCP and agent support | AGPL Community Edition; many services including Postgres, Redis, broker, and object storage; different work-item model ([architecture](https://developers.plane.so/self-hosting/plane-architecture); [MCP](https://developers.plane.so/dev-tools/mcp-server)) | Operational reference, not framework or protocol substitute |
| **Vikunja + `veans`** | Established self-hosted task manager; agent claims, dependencies, human completion boundary | AGPL core; `veans` is experimental; task-manager model rather than Board Channels/Threads/Events ([Vikunja](https://vikunja.io); [`veans`](https://vikunja.io/docs/veans)) | Closest mature functional substitute for simple agent task tracking |
| **LokalBoards, Collattice, Kanzo, Janosik** | Atomic claims, agent identities, MCP/CLI, SSE or long-poll; some SQLite/local-first designs | Very young, small adoption signals, and no reviewed project demonstrates the full production-backend combination ([LokalBoards](https://github.com/florian-strasser/LokalBoards); [Collattice](https://github.com/MrBildo/collattice); [Kanzo](https://github.com/emanuele-em/kanzo); [Janosik](https://github.com/VitalyTolstik/janosik)) | Validate superiority on reliability, identity, and operations; do not assume an empty market |
| **WeKan, Kanboard, Planka, Kan** | Mature or visible self-hosted kanban | Generic visual task tracking; differing licenses, maturity, or release status ([WeKan](https://github.com/wekan/wekan); [Kanboard](https://github.com/kanboard/kanboard); [Planka license](https://github.com/plankanban/planka/blob/master/LICENSE.md); [Kan](https://github.com/kanbn/kan)) | Not the product category; do not compete on cards and columns |
| **OpenProject, Taiga, AppFlowy** | Broad agile PM or workspace capabilities | Larger planning/workspace models and licensing/operational differences ([OpenProject](https://github.com/opf/openproject); [Taiga](https://taiga.io/); [AppFlowy](https://github.com/AppFlowy-IO/AppFlowy)) | Solve different jobs |
| **Linear, Shortcut, GitHub Projects** | Fast code-native workflows, agent delegation, live activity, approvals, integrations | Proprietary hosted products, not self-hosted Board-compatible backends ([Linear agents](https://linear.app/docs/agents-in-linear); [Shortcut](https://www.shortcut.com); [GitHub Projects](https://docs.github.com/en/issues/planning-and-tracking-with-projects/learning-about-projects/about-projects)) | Set the UX and integration bar without defining the deployment model |

**Inference, medium confidence:** no reviewed project combines Board protocol compatibility, a durable multi-user self-hosted backend, independent scoped agent principals, lease/dependency semantics, narrow GitHub ingestion, a human approval inbox, and mature operational tooling. This is a narrow integration gap, not evidence of a large standalone market. Stars and launch attention are weak maturity signals; release cadence, deployment documentation, restore evidence, and maintenance notices are stronger ([competitive notes sources](https://github.com/mattermost-community/focalboard)).

Upstream local mode is insufficient when the Board becomes shared production infrastructure. A laptop or tailnet can coordinate a few agents, but it does not by itself provide OIDC joining, separate CI/service identities, durable scheduled work after restart, cross-scope authorization tests, off-host encrypted backups, versioned migrations, an auditable dead-letter queue, or a tested export/re-import path. **OpenBoard should make those properties intrinsic rather than asking each team to rebuild them.**

## MVP scope and what not to build

### Included in the beta

| Area | Beta commitment |
|---|---|
| Protocol | New backend implementing the pinned Board protocol; official SDK/CLI reuse; published conformance suite |
| Domain | Workspace, Boards, Channels, Threads, Messages, Tasks, exact upstream states, Leases, Dependencies, Approval Requests, Events, Revisions, Tombstones; per-Channel completion policy |
| Automation | Durable watchers and scheduled follow-ups persisted in SQLite jobs/outbox and recovered after restart; no in-memory timers |
| Identity | One-time local bootstrap administrator; OIDC-first humans; Invitations/allowlist; separate human, service, CI, and agent Actors |
| Authorization | Channel Grants, deny-by-default agent access, revocable scoped Credentials, server-side checks on every read/write/export/job; human-only enablement of Channel automation |
| Clients | Protocol API plus a small native Admin API; Inspector and Admin Console; browser SSE; Board cursor/wait/long-poll |
| Synchronization | GitHub App on selected repositories with explicit minimum permissions and signed webhooks—never personal OAuth/PAT; inbound Issues/PRs/comments/reviews/checks only; durable inbox/outbox, retries, dead letters, replay, reconciliation |
| Data operations | SQLite WAL, short transactions, single-writer gate, content-addressed attachments, automatic backups with off-host capability and restore drills, versioned JSONL Export Archive and attachment bundle |
| Operations | Structured logs, health/readiness, Prometheus metrics, OpenTelemetry, quotas, singleton guard, TLS proxy contract, host-volume encryption guidance |
| Delivery | Docker Compose and VM runbooks, forward migrations, rollback/restore guidance, install-from-docs test, Apache-2.0 licensing and SBOM |

### What not to build

Do not build a general PM/kanban clone, drag-and-drop board, Gantt chart, sprint/cycle planner, time tracker, portfolio, wiki, CRM-like field system, or custom workflow builder. Do not build an agent runtime, model gateway, token billing system, execution sandbox, or model-provider marketplace. Do not build Slack, Linear, Jira, or GitLab synchronization in v1. Do not build two-way GitHub synchronization, inline-review-comment ingestion, GitHub write-back, or deployment/merge/rollback actions. Do not build a plugin marketplace, public plugin runtime, mobile application, multi-region HA, active-active failover, multiple Workspaces per Installation, multiple app replicas, shared SQLite over NFS, or application-level SQLite encryption.

SAML, SCIM, legal hold, advanced DLP, certified air-gapped support, contractual HA, and formal compliance certifications are not community MVP commitments. They are candidates for separately licensed enterprise add-ons only after repeated buyer evidence. **Never gate basic authorization, OIDC, audit history, backup, or export behind a commercial license.**

## Domain model and exact Task lifecycle

### Core model

| Concept | OpenBoard definition and implementation boundary |
|---|---|
| **Installation / Workspace** | One deployed Installation serves exactly one Workspace, the boundary owning Boards, Actors, Credentials, policy, and data |
| **Board / Channel** | A Board groups coordination work and allocates its ordered Event cursor; a Channel is the permission and synchronization boundary containing Threads and Tasks, with its own completion and automation policy |
| **Actor** | Authenticated human, service, CI, or agent principal; every mutation records the Actor and Credential used |
| **Agent Profile** | Display attribution for an Agent Actor; grants no authority and cannot broaden a Credential |
| **Credential / Grant / Invitation** | Revocable scoped authentication proof; explicit action rules on a Channel or Channel pattern; one-time authorization for an OIDC identity to join |
| **Thread / Message** | Persistent root-plus-replies record; each Message is Markdown attributed to an Actor and may carry labels, attachments, and versioned structured data |
| **Revision / Tombstone** | Immutable prior Message version after an edit; durable deletion marker that preserves audit history without serving deleted content |
| **Task** | A Thread with an owner, canonical lifecycle state, optional claimant, Dependencies, and outcome |
| **Lease** | Time-limited exclusive right to work on a claimed Task, identified by a hashed token and claim epoch |
| **Dependency** | Prerequisite edge; a dependent Task becomes ready only after every prerequisite is completed |
| **Approval Request** | A blocked Task with a reason requiring an explicit human decision; not a separate Task state |
| **Event** | Immutable domain change with a Board-local sequence; retained for Workspace lifetime unless explicitly purged by an administrator |
| **External Artifact / Sync Mapping** | Imported GitHub Issue or Pull Request; durable relationship between it and an OpenBoard Thread or Task |
| **Promotion** | Explicit, audited transition from a Thread into a Task; GitHub artifacts become Threads by default |
| **Delivery Evidence** | External check/status/review result associated with a Task; informative, never automatically authoritative for local Task state |
| **Delivery Attempt** | One recorded attempt to deliver or synchronize an Event; retained separately from domain Events |
| **Export Archive** | Versioned JSONL domain export plus attachment bundle and manifest for backup portability or migration |
| **Inspector / Admin Console** | Read-oriented human work interface; identity, Grants, Credentials, integrations, retention, export, and operations interface |

The persistence model is relational state plus an append-only Event log, **not event sourcing for every aggregate**. A mutation transaction updates canonical state, allocates the next Board Event sequence, appends the Event, and inserts any required outbox work. Clients reconstruct live state from an initial snapshot plus subsequent Events; the database remains authoritative.

### States and transitions

Preserve these upstream states exactly: `waiting`, `ready`, `claimed`, `blocked`, `completed`, `failed`, `canceled`. Do not add `in_progress`, `review`, or `done`, and do not encode lifecycle in labels ([tasks](https://vercel.com/docs/board/concepts/tasks)).

| From | To | Required condition |
|---|---|---|
| `waiting` | `ready` | Prerequisites are completed and assignment/readiness policy permits claiming; exact creation defaults follow the pinned protocol |
| `ready` | `claimed` | Atomic claim succeeds; exactly one unexpired Lease exists afterward |
| `claimed` | `claimed` | Current claimant renews with valid token and epoch before expiry |
| `claimed` | `ready` | Claimant releases, or Lease expiry is observed and recovery makes the Task claimable |
| `waiting` or `ready` | `blocked` | Owner/manager blocks work with a reason; any existing claim ends |
| `claimed` | `blocked` | Worker or owner blocks work; claim ends and an Approval Request is recorded when human input is required |
| `blocked` | `waiting` or `ready` | Authorized human resolves the Approval Request; readiness is recomputed from Dependencies |
| `claimed` | `completed` | Current claimant completes with an outcome before Lease expiry |
| `claimed` | `failed` | Current claimant fails with an outcome before Lease expiry |
| nonterminal | `canceled` | Authorized owner/manager cancels; deleting a Task root cancels it unless already completed |
| `completed`, `failed`, or `canceled` | `waiting` or `ready` | Explicit authorized reopen; recompute readiness and append an Event |

A failed or canceled prerequisite **does not unblock** its dependents. Lease expiry does not require a timer to fire: the next claim or recovery pass compares the persisted expiry and advances the stale claim atomically. An expired worker’s later renewal, completion, or mutation fails with `claim_lost`. The default worker-visible lease remains compatible with upstream’s five-minute maximum unless the pinned protocol says otherwise.

An **Approval Request is a blocked Task with a reason**, not a second workflow engine. The Inspector presents a decisions-needed queue; the Admin Console controls who may resolve it. In agent-facing Channels, default policy requires a human Actor other than the blocked Task’s claimant to unblock it. This stricter policy is an OpenBoard control and must be represented without changing protocol Task states.

## Identity, permissions, and security

### Authentication and joining

Human authentication is **OIDC-first with no custom password system**. On first start, the Installation generates a single-use bootstrap secret, prints it to the local operator console, and exposes setup only on loopback or an explicitly protected setup route. The bootstrap administrator configures OIDC, creates the first Workspace administrator binding, and invalidates the setup secret. It must not become a permanent parallel login path. Thereafter, joining requires an Invitation or an administrator-managed allowlist bound to a verified OIDC identity.

Humans, services, CI, and agents are distinct Actor kinds. Credentials are scoped, expiring where appropriate, revocable, and stored hashed; only a prefix is displayed after issuance. A Credential is bound to one Actor and cannot be widened by selecting an Agent Profile. Session and Credential revocation takes effect on subsequent authorization checks; active SSE/long-poll streams are closed or refreshed when their Channel access is revoked.

Better Auth is a candidate OIDC/session adapter because it supports `bun:sqlite` and can mount inside Effect through `HttpEffect.fromWebHandler` ([SQLite adapter](https://www.better-auth.com/docs/adapters/sqlite); [Effect integration](https://better-auth.com/docs/integrations/effect)). Use it only after a spike proves the OIDC and session flows fit the bootstrap model. Do not adopt its password or organization plugins merely because they exist. If used, its connection shares OpenBoard’s single-writer gate and ordered migration set.

### Authorization

Every query is scoped by Workspace and Board in the repository layer; a bare global `thread_id` lookup is forbidden. SQLite has no row-level security, so application enforcement and cross-scope tests are the v1 isolation boundary. Reads, writes, attachment downloads, Events, exports, Admin actions, background jobs, and synchronization all authorize independently. A live connection is not proof of continuing authority.

Keep the permission vocabulary fixed. Preserve upstream’s `read`, `post`, `react`, `label`, `sync`, and `manage` Channel Grant semantics at the compatibility boundary ([access](https://vercel.com/docs/board/access#permissions-and-channel-patterns)). Native administration may expose narrower Credential scopes such as `tasks:claim`, `tasks:manage`, `exports:read`, and `integrations:manage`, but the Board façade must map them to upstream semantics rather than silently expanding them. Evaluate the most specific Channel pattern, then principal, with explicit deny precedence; never rely on a default Workspace-wide grant for an agent.

New private Channels deny by default. Creating a Channel requires authority over its parent path. Listing Channels, Events, or exports reveals only authorized resources. Grant changes are audited in the same transaction as the change.

### Content and integration safety

External GitHub content and agent-authored Messages are untrusted. The Inspector marks imported content as external, sanitizes Markdown and links, blocks active HTML, and displays a persistent warning that agent-read content may contain prompt injection. OpenBoard does not execute arbitrary commands from Messages. Agent Channels deny by default, and high-risk agent tool actions remain outside OpenBoard.

Webhook routes accept POST over HTTPS, enforce a conservative raw-body limit, verify the provider signature over exact bytes with constant-time comparison, deduplicate the delivery ID, and return success only after durable inbox insertion. Use fixed GitHub hosts and no user-configurable outbound URLs in v1, avoiding an SSRF surface. These controls follow OWASP webhook guidance ([OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Webhook_Security_Cheat_Sheet.html)).

Attachments are private, content-addressed by SHA-256, size-limited, authenticated on download, and written to a temporary file that is fsynced and atomically renamed before metadata commits. Reject active content where possible, sniff MIME type, and quarantine or scan untrusted uploads before serving them when a scanner is available. Secrets never belong in Messages, Events, logs, exports, images, or provider payloads. Provider encryption keys live outside the database and support rotation.

TLS terminates at a reverse proxy. The host or storage provider encrypts the data volume. **OpenBoard v1 does not implement application-level SQLite encryption.** Run as a non-root user, pin image digests and lockfiles, generate an SBOM, scan dependencies, and publish a security policy. Log authentication, authorization failures, Grant changes, exports, integration changes, and destructive actions without logging tokens, session IDs, signatures, message bodies, or unnecessary personal data ([OWASP logging](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)).

## Architecture: one deep Effect stack on Bun and SQLite

### Deployment shape

```text
Browser / official CLI / SDK / CI / agents
                    |
          Optional TLS reverse proxy
                    |
        One Bun 1.4.5 OpenBoard process
          ├── React assets + Inspector/Admin Console
          ├── Effect HTTP API + Board-compatible façade
          ├── OIDC/session adapter
          ├── SSE and long-poll dispatcher
          └── Bounded Effect worker fibers
                    |
          One local persistent data volume
            ├── openboard.sqlite (+ WAL/SHM while open)
            └── attachments/<sha256>/...
```

One Workspace per Installation, one app replica, and one local persistent volume are hard constraints. The data directory must be local ext4/XFS on NVMe or equivalent—never NFS, SMB, or another network filesystem. SQLite WAL permits concurrent readers and one writer, but requires all processes to be on the same host ([SQLite WAL](https://www.sqlite.org/wal.html)). Acquire an OS-level singleton lock at startup and fail closed if the directory is already owned; the documented deployment contract still forbids multiple processes.

### Stack decisions

| Layer | Decision | Reason and constraint |
|---|---|---|
| Runtime | Bun **1.4.5** | Bundles SQLite 3.54.1 and security updates as of the cutoff; pin the runtime and image digest ([Bun release](https://bun.com/blog/bun-v1.4.5)) |
| Server | Effect **4.1.5**, `@effect/platform-bun@4.1.5` | Typed HTTP, errors, streams, workers, configuration, and telemetry |
| Browser | React renderer + `@effect/atom-react@4.1.5` | React owns rendering; Effect Atom owns browser async state/data; Vite+ handles development and production builds |
| Toolchain | Vite+ **1.0** (`vp`) | Vite/Rolldown, Vitest, Oxlint/Oxfmt/tsgo, cached workspace tasks, and staged checks; Bun remains the package manager and server runtime |
| Storage | `bun:sqlite` + `@effect/sql-sqlite-bun@4.1.5` | Embedded durable store, typed SQL errors, migrations, WAL |
| Auth | OIDC/session adapter inside Effect | No custom password system; bootstrap is one-time and local |
| API | Native `/api/v1` plus Board-compatible façade | One domain core, two wire contracts; no Effect RPC in v1 |
| Realtime | SSE for browser; HTTP cursor/wait or long-poll for Board clients | No Socket.IO, broker, or sticky sessions |
| Jobs | SQLite inbox/outbox processed by Effect fibers | Durable scheduling without Redis or a message broker |
| Attachments | Immutable files on the same local volume | Keeps BLOBs out of SQLite and makes one-volume restore possible |
| Operations | Prometheus metrics, OpenTelemetry, structured JSON logs | Required in v1, not deferred enterprise polish |

Effect 4.1.5’s HTTP, SQL, Atom, and observability APIs are partly marked unstable, and the 4.1 release notes warn of breaking changes ([Effect API](https://effect.website/docs/v4/api); [Effect 4.1 release](https://effect.website/blog/releases/effect/41)). Pin every Effect package to the same exact version and confine direct unstable imports to adapters such as `platform/http.ts`, `storage/sqlite.ts`, `browser/atom.ts`, and `observability/otel.ts`. A week-one spike must validate Bun client/server boundaries, bundle size, Atom hydration, Suspense behavior, and SQLite integration before the UI architecture is treated as final.

### Tooling and monorepo workflow

Use **Vite+ 1.0** as the unified developer toolchain while retaining **Bun 1.4.5** as the package manager and production server runtime ([Vite+](https://viteplus.dev/); [monorepo guide](https://viteplus.dev/guide/monorepo); [dependency management](https://viteplus.dev/guide/install)). The root `package.json` declares Bun workspaces and `packageManager: "bun@1.4.5"`; the root `vite.config.ts` holds shared lint, format, check, test, and task configuration.

Standard commands are `vp install`, `vp check`, `vp test`, `vp build`, `vp run`, and `vp staged`. Vite+ supplies Vite/Rolldown for the web client, Vitest for tests, Oxlint/Oxfmt/tsgo for combined checks, and dependency-aware cached task execution. `@effect/vitest` runs under Vitest. Do not add Biome, Nx, or Turborepo.

Vite+ does not replace Bun in production: the server still runs on Bun, and server packaging must not silently target Node through `vp pack`. Pin the resolved Vite+ version in the lockfile and prove Bun package-manager integration, Effect tests, browser bundling, and workspace task ordering in week one.

### Database and event-loop discipline

Set and verify `journal_mode=WAL`, `synchronous=FULL`, `foreign_keys=ON`, a short request-path `busy_timeout`, and monitored WAL autocheckpointing on every connection. `synchronous=FULL` avoids the last-commit loss tradeoff of `NORMAL` on power failure ([Bun SQLite](https://bun.com/docs/runtime/sqlite); [SQLite transactions](https://www.sqlite.org/lang_transaction.html)). Use a 1,000 ms request-path busy timeout and return a typed `DatabaseBusy`/503 with `Retry-After`; reserve longer waits for startup migrations and backups.

The Bun SQLite driver is synchronous. **Busy waits and long transactions block Bun’s event loop.** Effect’s Bun SQLite client serializes access and uses `BEGIN IMMEDIATE` for writable transactions, but that does not make blocking work disappear ([Effect SQLite client](https://effect.website/docs/v4/api/sql-sqlite-bun/SqliteClient)). Use a shared `Semaphore(1)` write gate for domain writes, worker writes, backup writes, and any auth-adapter writes to a second connection. Reads may proceed under WAL. Prepare and validate before the transaction; never await provider HTTP, filesystem scanning, exports, or other slow work inside it. Measure transaction duration, gate wait, `SQLITE_BUSY`, and event-loop lag; shorten transactions before adding infrastructure.

### Mutation, Event, and realtime flow

Every acknowledged mutation follows the same boundary: authorize, validate, acquire the write gate, run one short `BEGIN IMMEDIATE`, update canonical rows, allocate `board_seq`, append the Event, insert outbox/inbox work, commit, then publish after commit. Message edits use optimistic concurrency and preserve a Revision; deletion writes a Tombstone. Idempotency keys store a request hash and response so a retried CLI command cannot duplicate a Message or state transition.

A dispatcher fiber reads unpublished event-outbox rows and publishes them to an in-process Effect `PubSub`. Per-Board queues feed SSE and long-poll subscribers. Publication is at-least-once; clients deduplicate by `(board_id, board_seq)` and reconnect from their last cursor. A client first loads a snapshot plus cursor, subscribes, and requests Events after its last sequence on reconnect. If the cursor is no longer available, it reloads a snapshot. Domain Events are retained for Workspace lifetime, so normal reconnect should use catch-up; an explicit administrative purge is the only deletion path.

Browser SSE is an optimization. The durable Event log is the recovery mechanism. Presence and typing are ephemeral and intentionally lost on restart. Long-poll compatibility follows the upstream SDK’s cursor/wait behavior, including its approximately 25-second wait ([Board SDK](https://vercel.com/docs/board/sdk)).

### Protocol compatibility

The Board-compatible façade translates upstream wire DTOs into domain commands and maps typed domain errors back to upstream codes such as `revision_conflict`, `claim_lost`, `duplicate_id`, and `forbidden`. Preserve stable Message IDs/cursors, idempotency replay, conditional labels, claim epochs and Lease tokens, Dependencies, Tombstones, profile delegation, and exact documented errors.

At the research cutoff the observed latest package release was `@vercel/board@0.56.1`, while an earlier architecture note evaluated SDK 0.50.0 ([release](https://github.com/vercel/board/releases/tag/%40vercel/board%400.56.1); [SDK](https://vercel.com/docs/board/sdk)). **Week one must reconcile that discrepancy and freeze one exact package/protocol revision.** The public docs do not fully specify raw routes, and alternate-URL behavior of the official CLI is not established. Reusing the SDK against `BOARD_URL` is testable; drop-in CLI compatibility is an acceptance test, not an assumption. If the official CLI requires its supported Node runtime, run it as an external client; OpenBoard’s server remains Bun-only.

## GitHub synchronization and failure recovery

GitHub is an **inbound mirror only** in v1. A GitHub App connects selected repositories with minimum read permissions. Imported Issues and Pull Requests become Threads containing an External Artifact and a durable Sync Mapping. Issue comments, Pull Request conversation comments, reviews, and check/status results become authorized context and Delivery Evidence. **Inline review comments are excluded. Nothing is written back to GitHub.** An explicit, audited Promotion turns a Thread into a Task; imports never do so implicitly.

OpenBoard remains authoritative for Task state. A merged Pull Request, green check, or closed Issue does not complete a Task automatically. Delivery Evidence can inform an Actor, but the claimant or authorized owner records the outcome. Synchronization failures are non-blocking: they are visible and replayable, but never change Task state or prevent completion.

The inbound path is: verify signature over raw bytes → validate envelope and size → insert a durable inbox row keyed by connection plus `X-GitHub-Delivery` → return 2xx → worker fetches current authoritative state when ordering is ambiguous → apply an idempotent domain transaction → append an Event and enqueue follow-up work. GitHub expects a response within ten seconds and may deliver events out of order, so synchronous provider calls and arrival-order assumptions are forbidden ([webhook best practices](https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks); [troubleshooting](https://docs.github.com/en/webhooks/testing-and-troubleshooting/webhooks/troubleshooting-webhooks)).

Deduplicate both the provider delivery ID and a normalized domain fingerprint. Store external revision/update time when available; otherwise fetch current state before overwriting. GitHub deletion creates a Tombstone or updates the External Artifact state; it does not erase local history. Retry transient failures with bounded exponential backoff and jitter, honor `Retry-After`, isolate providers, dead-letter permanent failures, and expose an audited replay action. Reconcile active linked objects periodically and after connection changes, outages, dead-letter replay, or token refresh. Every failure must show the Event, Sync Mapping, Delivery Attempt count, last error class, next retry, and operator-visible status.

## Backup, export, observability, and capacity

### Backup and portability

The disaster-recovery unit is a consistent SQLite snapshot plus the co-located attachment directory, configuration, and checksum manifest. Use `VACUUM INTO` or SQLite’s Online Backup API from a dedicated connection; briefly acquire the write gate when producing a strict database-plus-attachments snapshot so the recorded Event sequence and attachment set agree ([SQLite backup](https://www.sqlite.org/backup.html); [`VACUUM INTO`](https://www.sqlite.org/lang_vacuum.html#vacuum_into)). Do not use whole-database serialization as the large-database backup strategy.

Starting objectives are **RPO ≤24 hours and RTO ≤4 hours** for the reference small-team Installation. These are recommendations to validate, not availability guarantees. Keep encrypted off-host copies; a snapshot on the same VM is not disaster recovery. A suggested starting retention is 30 daily plus 12 weekly backups, with keys stored separately and a documented deletion policy.

A restore drill restores into an empty directory, runs `PRAGMA integrity_check` and `PRAGMA foreign_key_check`, verifies attachment counts and SHA-256 hashes, checks Grants and identity mappings, starts the service, and performs a real OIDC login plus read/write smoke test. Test before beta and before major upgrades.

The **Export Archive** is distinct from a disaster-recovery snapshot. It contains `manifest.json`, versioned JSONL for all domain state and Events, an attachment bundle keyed by stable IDs, checksums, counts, application/protocol versions, and re-import instructions. Include Workspace configuration, Actors and non-secret identity references, Invitations as appropriate, Boards, Channels, Grants, Threads, Messages, Revisions, Tombstones, Tasks, claim history, Leases, Dependencies, Approval Requests, Events, Sync Mappings, Delivery Evidence, and administrative audit records. Exclude password hashes, sessions, Credentials, webhook secrets, provider tokens, encryption keys, raw webhook payloads, and ephemeral job locks.

Import supports `--dry-run`, schema validation, referential-integrity checks, attachment checksum verification, stable-ID preservation, conflict reporting, and optional ID remapping into a non-empty Workspace. Run import as an audited background job with progress and a reconciliation report. Re-import must preserve meaning and relationships, not necessarily raw database bytes.

### Observability

Emit structured JSON logs with request/trace ID, version, Actor, Workspace/Board, action, result, and duration—never content or secrets. Expose liveness, readiness, and Prometheus metrics. Readiness requires an open database with current migrations; worker or realtime degradation is visible without unnecessarily taking down the Inspector. OpenTelemetry traces span HTTP → database → job → GitHub boundaries with conservative sampling and no payload capture ([Effect OTLP](https://effect.website/docs/v4/api/effect/observability/Otlp); [Prometheus metrics](https://effect.website/docs/v4/api/effect/observability/PrometheusMetrics)).

Track request rate/latency/errors, write-gate wait, transaction duration, `SQLITE_BUSY`, WAL/checkpoint state, event-loop lag, Event publication lag, SSE/long-poll clients, queue depth and oldest age, webhook acceptance/signature failures, provider throttles, dead letters, storage usage, and backup age. Alert on sustained 5xx, event-loop blockage, old jobs, dead-letter growth, signature-failure spikes, failed/old backups, TLS expiry, and disk pressure.

### Capacity envelope

The following are **engineering test targets, not guarantees**: 5–25 regular humans, 10–50 active agents, 50–100 concurrent realtime clients, 10,000–50,000 Tasks, 250,000–1 million Events, 5–20 aggregate writes/second, and p95 realtime delivery under one second in-region. Protocol-compatible Message limits remain 10,000 characters and ten attachments of up to 100 MB each; the Installation must also publish configurable aggregate storage quotas and reject new writes clearly before disk exhaustion ([Board limits](https://vercel.com/docs/board#what-to-know-before-you-use-board)).

Run representative load tests with real Message sizes, concurrent claims, long-poll fan-out, webhook bursts, WAL checkpointing, backups, and attachment hashing. If write p95 exceeds 300 ms, `SQLITE_BUSY` recurs, or event-loop lag rises, shorten transactions, remove slow work from the gate, tune indexes, and scale the VM. PostgreSQL is only a future migration path if measured horizontal write, HA, or PITR requirements justify it. Do not add replicas sharing the SQLite file.

A small reference VM plus off-host backup storage is plausibly **$20–60/month before labor**, email, and unusually large attachment volumes. This is a planning estimate from point-in-time VM and storage prices, not a quote ([Hetzner](https://www.hetzner.com/cloud/cost-optimized); [Backblaze B2](https://www.backblaze.com/cloud/storage/pricing)). Operator labor, upgrades, restore drills, and support remain real costs.

## Repository shape and release gates

The repository currently has no application code. Create one Bun workspace with one release version and these boundaries:

```text
packages/contracts        Effect Schemas, commands, Events, tagged errors
packages/domain           Pure use cases and repository/service interfaces
packages/storage-sqlite   Pragmas, migrations, transactions, queue claiming
packages/platform-http    Bun/Effect HTTP, auth mounting, SSE/long-poll adapters
packages/integrations     GitHub adapter and provider-neutral interfaces
apps/server               One process entrypoint: API, Board façade, workers
apps/web                  React renderer + Effect Atom browser runtime
apps/admin-cli            Bootstrap, export/import, health; not a protocol rewrite
tests/conformance         Pinned official SDK/CLI compatibility suite
tests/integration         File-backed SQLite, crash, auth, restore, sync tests
deploy                    Compose, reverse proxy example, migration job, runbooks
docs                      Architecture, protocol, security, backup, export, ADRs
```

Keep `packages/contracts` free of Bun/Node APIs, SQLite clients, secrets, and server-only auth imports. The CLI is a thin native client for administrative operations; the official Board CLI/SDK remains the protocol client. Put direct unstable Effect imports only in adapters. Store domain terminology in the existing root `GLOSSARY.md` and architectural decisions in `docs/adr/`; this report does not modify either. Implementation issues and specs belong in Linear project **P-STR-10**.

### Mandatory beta gates

| Gate | Pass condition |
|---|---|
| **Protocol conformance** | Pinned official SDK and, if verified, CLI pass post/reply/edit/delete, labels, structured data, Tasks/claims/leases, Dependencies, Events/poll/wait, Grants, idempotency, and documented error codes |
| **Atomic claims** | Concurrent race tests never produce two active claimants; stale tokens cannot renew, complete, or mutate; expired claims recover without a timer |
| **Crash recovery** | Kill tests at each transaction boundary lose no acknowledged write; inbox/outbox jobs resume exactly once at the domain-effect level; singleton ownership is enforced |
| **No silent Event loss** | Every mutation commits state, Event sequence, and outbox atomically; missed publication replays; clients detect gaps and resnapshot; lag/dead letters alert |
| **Restore drill** | Clean-host restore of database, configuration, and attachments meets documented RPO/RTO and passes integrity, checksum, authorization, and login checks |
| **Export/re-import** | Versioned archive preserves domain meaning, relationships, Events, and attachments; secrets are excluded; dry-run and conflict reports work |
| **Install from docs** | At least five design-partner teams install without maintainer intervention; a new Actor reaches a claimed/completed Task in under 30 minutes |
| **Cross-scope authorization** | Automated tests cover every route, read, Event, attachment, export, job, and Admin action across Actors, Channels, Boards, and revoked Credentials |
| **Observable sync failures** | Every GitHub failure is attributable, retried or dead-lettered, visible in the Admin Console, and safely replayable |
| **Security and upgrade** | OIDC/bootstrap tests, webhook raw-body verification, input limits, CSP/headers, dependency/SBOM checks, migration rehearsal, and restore-based rollback pass |

A gate failure blocks the public beta. “Works on the maintainer’s laptop” is not a release result.

## Twelve-week delivery plan

Recruit design partners in week one and run recruitment in parallel with implementation. Target five private-alpha teams with two harnesses each and one concrete workflow. Do not wait for fifteen interviews or a complete market study before writing the vertical slice.

| Weeks | Build and validate | Exit criteria |
|---|---|---|
| **1** | Repository skeleton, Apache-2.0 files, pinned Bun/Effect/Vite+, Vite+-driven CI/SBOM; Effect/Atom/Bun/Vite+ spike; exact upstream protocol pin; SQLite crash/load spike; recruit partners | Compatibility path demonstrated against one official SDK call; Vite+ checks/tests/tasks work with Bun workspaces and Effect; no unresolved blocker to one-process SQLite; five partner prospects identified |
| **2** | SQLite schema and migrations, write gate, Event/outbox/inbox, OIDC adapter, one-time bootstrap, Invitations | Fresh install and upgrade work; bootstrap cannot become a permanent password path; crash-safe transaction harness runs |
| **3** | Boards, Channels, Threads, Messages, labels, structured data, Revisions, Tombstones, idempotency | Message lifecycle and revision conflicts pass integration tests; every mutation appends an Event and required outbox work |
| **4** | Tasks, exact states, atomic Leases, Dependencies, Approval Requests; native API; minimal Inspector | **Working vertical slice:** OIDC human → Thread/Task → two racing workers → one claim → renew/complete → cursor catch-up in browser and SDK |
| **5** | Grants, Actor kinds, Credentials, revocation, Admin Console, cross-scope tests | Agent Channels deny by default; revoked access fails on all routes and streams; attribution never expands authority |
| **6** | Board-compatible façade hardening, SSE, long-poll/wait, official SDK/CLI tests, schema/error mapping | Protocol conformance suite passes the pinned revision; reconnect/gap/resnapshot behavior is proven |
| **7** | GitHub App, inbound Issues/PRs/comments/reviews/checks, Sync Mappings, Promotion, retries/dead letters/replay | No write-back; artifacts create Threads only; signed duplicate/out-of-order deliveries are safe and visible |
| **8** | Attachments, audit, export/import, backup job, structured logs/metrics/OTel, install docs | **Private alpha:** five teams install and run one real workflow; restore and export smoke tests pass; failures are observable |
| **9** | Security review, prompt-injection warnings, quotas, rate limits, backup/restore automation, upgrade runbook | Cross-scope, webhook, input, secret-handling, and restore gates pass; no critical security finding remains |
| **10** | Load, crash, race, and provider-outage tests; event-loop/WAL tuning; reconciliation | Reference capacity targets measured; no duplicate claims or silent acknowledged-write loss; restore meets RPO/RTO |
| **11** | Clean-room installs, support drills, documentation fixes, release artifact, compatibility fixes | Five independent installs pass without intervention; upgrade and restore-based rollback rehearsed |
| **12** | Hardened beta, published conformance report, runbooks, security policy, partner review | **Hardened beta:** every mandatory gate passes; three teams commit to continued production use; known limits and support policy are explicit |

If the team does not already know Effect v4, spend the first week proving the architecture rather than assuming the learning curve fits. The schedule is credible only with one deep backend, one integration, and no parity race against every upstream feature.

## Business model and adoption

The core is **fully open Apache-2.0**: protocol server, SQLite backend, Tasks/Leases/Dependencies, OIDC, basic Grants, distinct Actor identities, audit/export, GitHub adapter, Inspector, Admin Console, conformance tests, and deployment tooling. Preserve upstream Apache-2.0 notices if any code is reused, publish provenance, and do not imply affiliation with Vercel. Use a DCO or narrow non-exclusive contributor agreement; avoid copyright assignment unless dual licensing is a clearly understood community goal.

Monetize later in this order:

1. **Support and upgrade assistance**, priced from measured engineer time and response commitments.
2. **Managed single-tenant hosting**, only after three teams request it and restore, upgrade, monitoring, and support runbooks are proven. One managed customer still receives one Workspace per Installation.
3. **Separately licensed enterprise add-ons**, such as SAML/SCIM, advanced policy, retention/legal hold, dedicated HA/DR, or compliance reporting, only after repeated buyer evidence.

Do not use BSL, SSPL, custom competitor restrictions, retroactive relicensing, license phone-home, or feature degradation when a subscription expires. Never charge for basic security, audit history, export, or the ability to leave. Avoid per-token or per-agent billing: OpenBoard coordinates work rather than metering execution, and unpredictable billing would incentivize hidden agent activity.

GitLab, Grafana Labs, and Plane demonstrate that support, managed operations, and enterprise controls can sustain open-source infrastructure businesses, but their scale and markets cannot be extrapolated to this niche ([GitLab pricing](https://about.gitlab.com/pricing); [Grafana scale](https://grafana.com/press/2025/09/30/grafana-labs-surpasses-400m-arr-and-7000-customers-gains-new-investors-to-accelerate-global-expansion); [Plane self-hosting](https://plane.so/self-hosted)). Donations alone are not the base plan.

Adopt through upstream compatibility, platform-engineering communities, agent-harness maintainers, and reproducible production references. Publish a compatibility report and reference architecture rather than a feature checklist. Contribute generally useful protocol or backend improvements upstream where practical, but OpenBoard must remain useful if they are not accepted.

Six-month evidence targets are ten active self-hosted production Installations, three paid support or managed-hosting contracts, two independent references, and at least two buyers stating that identity, audit, residency, or operational control—not “open source”—drove the decision. Track time-to-first-task, install failures, duplicate claims, sync failures, restore success, support hours per Installation, and gross margin after storage and support.

## Risks, kill criteria, and pivot rules

| Risk | Assessment | Mitigation or decision rule |
|---|---|---|
| **No durable wedge** | High: upstream already has local/tailnet/custom-backend support and hosted Board is free while experimental | Recruit partners with a named production gap; stop if they only want a free local board |
| **Upstream closes the gap** | High: Vercel could ship production self-hosting, policy, or export | Compete on independent operations and portability; if parity arrives, pivot to policy/audit/export/support or stop |
| **Protocol drift** | High: Board is experimental and raw wire details are incompletely documented | Pin exact revision; conformance tests; adapter boundary; do not track latest silently |
| **CLI/SDK compatibility fails** | Medium/high: SDK `BOARD_URL` is documented, full CLI alternate-URL behavior is not | Prove in week one and gate alpha; do not compensate by forking the entire upstream client |
| **Wrong buyer or PM-suite demand** | High: generic task needs are already served | Stop if prospects consistently require kanban, Gantt, sprints, or a broad PM suite |
| **SQLite/event-loop limits** | Medium: synchronous driver and one writer constrain throughput | Short transactions, gate, quotas, metrics, load tests; scale VM first; PostgreSQL only on measured need |
| **Security or prompt injection** | High: Messages can instruct agents and external sync can widen audiences | Scoped Actors, deny-by-default agent Channels, warnings, no execution, inbound-only GitHub, authorization tests |
| **Sync fragility** | High: providers reorder, duplicate, throttle, and omit events | Durable inbox/outbox, current-state fetch, reconciliation, dead letters, replay, visible Delivery Attempts |
| **Maintenance/support burden** | High after traction | One integration and one deployment shape; measure support hours; do not launch always-on SaaS prematurely |
| **License/procurement trust** | Medium | Apache-2.0 before contributions; separate enterprise licenses; no retroactive restrictions |
| **Backup/storage cost** | Medium/unknown | Quotas, content addressing, lifecycle rules, off-host backup accounting, restore drills |

**Kill or pause criteria:** pause after week eight if fewer than three of five alpha teams can install and sustain a real workflow without maintainer rescue. Do not declare beta if any mandatory release gate fails. By six months, stop or narrow the project if fewer than three of ten pilot teams keep it in production, no buyer will pay for support/hosting after using the core, or support effort grows faster than paid demand. Pivot away from a broad control-plane roadmap if the durable value is only one narrow adapter, an audit/export layer, or compatibility maintenance. Do not respond to weak demand by adding kanban, an agent runtime, or many integrations.

## Confidence and unresolved questions

**High confidence:** Board’s launch date, product shape, documented protocol entities and Task states, Apache-2.0 public repository, local/custom-backend support, hosted US-only storage, broad default access, beta status, and documented lease behavior are directly supported by primary sources. The recommendation to reuse the protocol instead of forking the whole repository is a strategic inference with strong supporting evidence.

**Medium confidence:** the production-backend wedge is plausible, but direct interviews, willingness to pay, market size, retention, and the strongest purchase trigger are unmeasured. The claim that no reviewed alternative combines the full desired production feature set is bounded by the research cutoff, not a permanent market fact.

**Low confidence or unresolved:** the source status of Vercel’s managed production backend; its post-beta pricing, residency, self-hosting, and API-stability roadmap; exact raw protocol routes; official CLI alternate-URL behavior; the authoritative SDK/protocol revision to pin; external production adoption; willingness to pay; and whether small teams will accept an Inspector rather than a richer board UI. The public repository release snapshot and earlier SDK evaluation differ, so version selection is a week-one task. No independent security audit, SLA, or comparative multi-agent reliability benchmark was found for Board.

The product decisions in this report are intentionally settled: Apache-2.0 core, Bun/SQLite v1, one Workspace per Installation, OIDC-first identity, exact Task states, inbound-only GitHub, no kanban, and the week 4/8/12 milestones. The unresolved questions should be answered by implementation spikes and design-partner evidence, not by reopening the architecture.

## Sources

All URLs below are reproduced from the supplied research notes and were accessed on 2026-11-24 unless the source states its own date. Vendor claims are not treated as independent adoption evidence.

### Official Vercel Board sources

- [Introducing Vercel Board](https://vercel.com/blog/introducing-vercel-board)
- [Vercel Board overview](https://vercel.com/docs/board)
- [Board Tasks](https://vercel.com/docs/board/concepts/tasks)
- [Board Sync](https://vercel.com/docs/board/concepts/sync)
- [Board Access](https://vercel.com/docs/board/access)
- [Board SDK](https://vercel.com/docs/board/sdk)
- [Vercel Board changelog](https://vercel.com/changelog/vercel-board-is-now-available)
- [Public Beta Agreement](https://vercel.com/docs/release-phases/public-beta-agreement)
- [vercel/board repository](https://github.com/vercel/board)
- [vercel/board Apache-2.0 license](https://github.com/vercel/board/blob/main/LICENSE)
- [@vercel/board 0.56.1 release](https://github.com/vercel/board/releases/tag/%40vercel/board%400.56.1)
- [Backend mounting and custom backend documentation](https://raw.githubusercontent.com/vercel/board/main/packages/board/docs/01-board/01-mounting.md)
- [Local backend documentation](https://raw.githubusercontent.com/vercel/board/main/packages/board/docs/01-board/10-local.md)
- [Board package manifest and exports](https://raw.githubusercontent.com/vercel/board/main/packages/board/package.json)

### Runtime, Effect, storage, and security sources

- [Vite+ unified toolchain](https://viteplus.dev/)
- [Vite+ monorepo guide](https://viteplus.dev/guide/monorepo)
- [Vite+ dependency management](https://viteplus.dev/guide/install)
- [Bun v1.4.5 release](https://bun.com/blog/bun-v1.4.5)
- [Bun SQLite](https://bun.com/docs/runtime/sqlite)
- [SQLite WAL](https://www.sqlite.org/wal.html)
- [SQLite transactions](https://www.sqlite.org/lang_transaction.html)
- [SQLite foreign keys](https://www.sqlite.org/foreignkeys.html#fk_enable)
- [SQLite backup API](https://www.sqlite.org/backup.html)
- [SQLite VACUUM INTO](https://www.sqlite.org/lang_vacuum.html#vacuum_into)
- [Effect v4 API](https://effect.website/docs/v4/api)
- [Effect 4.1 release](https://effect.website/blog/releases/effect/41)
- [Effect Bun SQLite client](https://effect.website/docs/v4/api/sql-sqlite-bun/SqliteClient)
- [BunHttpServer](https://effect.website/docs/v4/api/platform-bun/BunHttpServer)
- [Effect SSE](https://effect.website/docs/v4/api/effect/encoding/Sse)
- [Effect Atom React hooks](https://effect.website/docs/v4/api/atom-react/Hooks)
- [Effect AtomHttpApi](https://effect.website/docs/v4/api/effect/reactivity/AtomHttpApi)
- [Effect OTLP](https://effect.website/docs/v4/api/effect/observability/Otlp)
- [Effect Prometheus metrics](https://effect.website/docs/v4/api/effect/observability/PrometheusMetrics)
- [Better Auth SQLite](https://www.better-auth.com/docs/adapters/sqlite)
- [Better Auth Effect integration](https://better-auth.com/docs/integrations/effect)
- [OWASP Webhook Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Webhook_Security_Cheat_Sheet.html)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/)

### GitHub integration sources

- [GitHub webhook best practices](https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks)
- [GitHub webhook troubleshooting](https://docs.github.com/en/webhooks/testing-and-troubleshooting/webhooks/troubleshooting-webhooks)
- [Validating GitHub webhook deliveries](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries)
- [Choosing GitHub App permissions](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app)
- [Generating GitHub installation access tokens](https://docs.github.com/en/apps/authenticating-with-a-github-app/generating-an-installation-access-tokens)

### Competitive and business sources

- [Plane self-hosting](https://plane.so/self-hosted)
- [Plane self-hosting architecture](https://developers.plane.so/self-hosting/self-hosting-101)
- [Plane architecture](https://developers.plane.so/self-hosting/plane-architecture)
- [Plane MCP](https://developers.plane.so/dev-tools/mcp-server)
- [Plane repository](https://github.com/makeplane/plane)
- [Vikunja](https://vikunja.io)
- [Vikunja veans](https://vikunja.io/docs/veans)
- [LokalBoards](https://github.com/florian-strasser/LokalBoards)
- [Collattice](https://github.com/MrBildo/collattice)
- [Kanzo](https://github.com/emanuele-em/kanzo)
- [Janosik](https://github.com/VitalyTolstik/janosik)
- [WeKan](https://github.com/wekan/wekan)
- [Kanboard](https://github.com/kanboard/kanboard)
- [Kan](https://github.com/kanbn/kan)
- [Planka license](https://github.com/plankanban/planka/blob/master/LICENSE.md)
- [OpenProject](https://github.com/opf/openproject)
- [Taiga](https://taiga.io/)
- [AppFlowy](https://github.com/AppFlowy-IO/AppFlowy)
- [Focalboard maintenance warning](https://github.com/mattermost-community/focalboard)
- [Linear agents](https://linear.app/docs/agents-in-linear)
- [Shortcut](https://www.shortcut.com)
- [GitHub Projects](https://docs.github.com/en/issues/planning-and-tracking-with-projects/learning-about-projects/about-projects)
- [Kargo](https://github.com/akuity/kargo)
- [Backstage](https://github.com/backstage/backstage)
- [GitHub Octoverse 2025](https://github.blog/news-insights/octoverse/octoverse-a-new-developer-joins-github-every-second-as-ai-leads-typescript-to-jump/)
- [GitLab pricing](https://about.gitlab.com/pricing)
- [Grafana Labs scale announcement](https://grafana.com/press/2025/09/30/grafana-labs-surpasses-400m-arr-and-7000-customers-gains-new-investors-to-accelerate-global-expansion)
- [OpenTofu manifesto](https://opentofu.org/manifesto)
- [OpenTofu fork announcement](https://opentofu.org/blog/opentofu-announces-fork-of-terraform)
- [Hetzner cost-optimized cloud](https://www.hetzner.com/cloud/cost-optimized)
- [Backblaze B2 pricing](https://www.backblaze.com/cloud/storage/pricing)

### Local inputs

- `/home/hvnxdr/dev/openboard/GLOSSARY.md`
- `/home/hvnxdr/dev/openboard/research_notes/Open source Vercel Board alternative/vercel_board.md`
- `/home/hvnxdr/dev/openboard/research_notes/Open source Vercel Board alternative/technical_architecture.md`
- `/home/hvnxdr/dev/openboard/research_notes/Open source Vercel Board alternative/product_strategy.md`
- `/home/hvnxdr/dev/openboard/research_notes/Open source Vercel Board alternative/competitive_landscape.md`

## Conclusion

The important correction is that OpenBoard should not compete with Vercel Board’s openness, protocol, or local prototype; it should make that protocol safe to operate as shared production infrastructure. The product’s moat is the combination of independent Actor identity, explicit Grants, durable Leases and Events, inbound-only GitHub evidence, portable archives, and recovery evidence that a team can verify—not a prettier board.

That correction also sharpens the failure mode. If design partners need only local coordination, upstream already serves them. If they need policy, attribution, durability, residency, and operational control enough to adopt and pay for support, a small one-process backend can be a credible community product. Build the vertical slice first, let partner evidence decide the wedge, and refuse scope that turns OpenBoard into another project-management suite.

<details>
<summary><strong>Agent notes: implementation detail</strong></summary>

These notes expand the main proposal for implementers. They are recommendations unless tied to a cited verified fact.

## 1. Freeze the protocol before building the façade

Record the exact upstream commit, `@vercel/board` package version, CLI version, protocol document hash, and Node runtime requirement in `docs/protocol-compatibility.md`. Treat the SDK as an executable specification where raw routes are undocumented. Run the SDK against a recording proxy and a test backend to derive request/response fixtures without copying upstream implementation code into OpenBoard.

The compatibility suite must cover:

- Channel listing and path semantics, including subtree behavior.
- Thread creation, replies, edits, deletes, Tombstones, and revision conflicts.
- Labels, reactions if included by the pinned protocol, attachments, and versioned structured blocks.
- Idempotency-key replay with identical and conflicting request bodies.
- Task creation, assignment, claim ordering, renewal, release, expiry, completion, failure, cancellation, reopen, Dependencies, and block/unblock.
- Cursor reads, `tail`, `wait`/poll, reconnect, and gap behavior.
- Grants, attribution profiles, and upstream error payloads.
- Authentication expected by the SDK and CLI when `BOARD_URL` points at OpenBoard.

Do not mark a command compatible because a happy-path response looks similar. Assert status, error code, cursor allocation, ordering, Lease behavior, and idempotent replay.

## 2. Recommended SQLite tables and constraints

Use TEXT IDs, explicit primary keys, foreign keys, ISO-8601 timestamps or integer milliseconds mapped through shared schemas, and validated JSON stored as text. Avoid rowid assumptions, SQLite-only `INSERT OR REPLACE`, and SQL that will block a future PostgreSQL migration.

| Table group | Minimum rows and constraints |
|---|---|
| `installations`, `workspaces` | One Workspace row per Installation; singleton configuration and schema version |
| `actors`, `agent_profiles`, `credentials`, `invitations` | Actor kind; OIDC subject/provider reference; hashed Credential; expiry/revocation; profile references Actor only |
| `boards`, `channels` | Board owns `event_seq`; unique normalized Channel path per Board |
| `grants` | Board, Channel pattern, principal, action set, deny flag, priority; most-specific pattern evaluation |
| `threads`, `messages`, `message_revisions`, `tombstones` | Thread root/reply ordering; Message version; unique `(thread_id, client_request_id)`; immutable Revision history |
| `labels`, `structured_payloads`, `attachments` | Typed metadata; immutable schema version; SHA-256, size, MIME, uploader, storage key |
| `tasks`, `task_claims`, `task_dependencies`, `approval_requests` | `thread_id UNIQUE`; exact state; owner; claim epoch; hashed Lease token; expiry; unique Dependency pair; block reason |
| `board_events` | Unique `(board_id, board_seq)`; type/version, Actor, aggregate, minimal payload, timestamp |
| `audit_events` | Append-only administrative/security record; runtime cannot update/delete; explicit purge is separately audited |
| `event_outbox`, `jobs` | Durable publication and follow-up work; idempotency key; attempts; next attempt; lease/lock; terminal status |
| `webhook_deliveries` | Unique `(connection_id, delivery_key)`; payload hash; received/processed state; attempts; error summary |
| `integration_connections`, `sync_mappings`, `delivery_evidence`, `delivery_attempts` | Provider/account/repository; encrypted Credential reference; external ID/revision; local Thread/Task; attempt history |
| `idempotency_keys` | Actor, route, key, request hash, stored response, expiry |

Add indexes for Board Event cursor lookups, Channel-scoped Thread/Task queries, ready Task selection, Lease expiry, due jobs, oldest unpublished outbox rows, and Sync Mapping lookups. Do not index or log Message bodies unnecessarily.

## 3. Transaction invariants

A message transaction inserts the Message, increments Thread version, allocates Board sequence, appends the Event, and inserts outbox work. An edit verifies the expected version, copies the prior rendered/raw representation into a Revision, updates current content, and appends an Event. A delete writes a Tombstone and removes the Message from normal reads without erasing its audit trail.

A claim transaction reads Task state under `BEGIN IMMEDIATE`, verifies readiness and eligibility, expires a stale claim if necessary, updates claimant/Lease expiry/claim epoch, inserts claim history, allocates the Board Event, and inserts outbox work. Commit is the only success point. Renewal and completion require both current epoch and hashed Lease token. A zero-row conditional update maps to `claim_lost` or a not-found response without leaking another claimant’s identity beyond protocol requirements.

A Grant transaction updates the rule and appends an audit Event atomically. A webhook transaction only durably accepts the delivery. A GitHub apply transaction upserts the external read model and Sync Mapping, appends a user-visible Event when appropriate, and inserts follow-up outbox work. No provider HTTP call occurs inside any transaction.

## 4. Event and outbox correctness

Use a single monotonic `board_seq` allocated by incrementing the Board row inside the mutation transaction. Event payloads contain minimal non-sensitive metadata; clients fetch authorized detail separately. Never include Credentials, Grant internals, full provider payloads, or unrestricted Message bodies in public Event envelopes.

The dispatcher may publish at-least-once. Mark publication only after the in-process publish succeeds, and replay unmarked rows after restart. Client deduplication uses Board ID plus sequence. An SSE `Last-Event-ID` or native cursor drives catch-up. Heartbeats keep proxies from closing idle streams. A missed range is fetched from SQLite; a purged or otherwise unavailable cursor returns a typed resnapshot signal.

Domain Events are retained for Workspace lifetime. Administrative purge is explicit, scoped, authorized, audited, and reflected in export documentation. Delivery Attempts and raw provider payloads have separate configurable retention so debugging data does not dictate domain-history retention.

## 5. Worker and retry implementation

Use a bounded set of Effect fibers in the same process. Claim one due job with a conditional update inside a short `BEGIN IMMEDIATE`, commit, perform provider I/O with timeout/rate limiting, then record success or retry in a second short transaction. Do not hold the write gate during HTTP calls. Use provider-specific queues or concurrency limits so GitHub cannot starve Event publication or backup work.

Classify failures as permanent, transient, or retryable conflict. A starting policy is 1, 5, 15, and 60 minutes, then hourly with jitter, capped at ten attempts or 24 hours; make it configurable. Honor `Retry-After`. Dead-letter with a redacted error class and operator-readable summary. Manual replay creates a new Delivery Attempt against the same idempotent domain path rather than mutating history.

Reconcile active linked GitHub objects every 15–60 minutes and all linked objects nightly as starting points. Fetch current state whenever arrival order cannot be established. Never let reconciliation overwrite a newer known external revision or silently modify local Task state.

## 6. OIDC and bootstrap sequence

Generate a high-entropy one-time bootstrap secret at first start. Bind the setup listener to loopback by default; require an explicit public-setup flag plus TLS and a short expiry for remote bootstrap. Setup permits only OIDC configuration, the first Workspace administrator binding, and initial Invitation/allowlist policy. Hash and invalidate the bootstrap secret after success. Disable setup permanently once initialized.

Use state and nonce validation, PKCE where supported, exact redirect URI allowlisting, and issuer/audience checks. Bind the resulting human Actor to a stable OIDC subject, not an editable email address. Enforce Invitation/allowlist after bootstrap. Administrator MFA is delegated to the OIDC provider; require suitable assurance claims only when policy demands them. CLI/service authentication uses scoped Credentials, not browser sessions.

## 7. Browser state boundaries

Use React for rendering and render-local ephemerals such as focus, menus, drafts, hover, and animation. Use Effect Atom for remote reads, mutations, shared asynchronous state, and the scoped Event stream. Wrap the application in `RegistryProvider`; hydrate serializable query snapshots, never sessions or secrets. Render `AsyncResult` states explicitly. Keep optimistic UI limited to reversible indicators such as “sending”; reconcile from the authoritative Event after commit.

Maintain one scoped SSE runtime rather than a long-lived fiber per component. On each Event, validate with the shared schema, update or invalidate affected atoms, and continue from the durable cursor. Do not put Effect Cluster, Workflow, SQL, filesystem, or server Config into the browser bundle.

## 8. Backup and export algorithms

For a strict backup: acquire the write gate, record the current Board sequence and attachment manifest generation, create a unique temporary SQLite snapshot, copy/tar immutable attachment files, fsync outputs, write checksums and metadata, release the gate, then upload encrypted artifacts off-host. An interrupted snapshot is incomplete and must not replace the last verified backup. Cleanup removes unreferenced temporary and orphan attachment files only after a grace period.

For export: take a consistent read transaction or equivalent snapshot marker, stream JSONL rows in dependency order, copy attachment objects, write `manifest.json` last, and verify counts/checksums. Exclude all authentication secrets and sessions. Import into a staging set of tables or an empty Installation, validate references and schema version, verify attachments, then atomically activate. Record an audit Event and reconciliation report. Never silently overwrite conflicting stable IDs.

## 9. GitHub permission and event matrix

Use a GitHub App installed on selected repositories. Request only permissions required by the implemented endpoints; the likely read surface is Issues, Pull Requests, and Checks/statuses, but verify exact requirements rather than requesting broad administration or Contents access by default. Store installation ID and selected repositories. Generate short-lived installation tokens on demand and keep the private key outside the database.

Ingest Issue metadata and issue comments; Pull Request metadata, conversation comments, reviews, and check/status results. Exclude inline review comments in v1. Store repository, external kind/ID, external revision/update time, actor identity, visibility, and source delivery. Map artifacts to a Channel by explicit repository/Channel configuration. Create a Thread on first observation; create a Task only through Promotion. OpenBoard task state never follows GitHub state automatically.

## 10. Crash and failure test matrix

Inject process death before and after commit, during Event publication, after inbox acceptance, after provider success but before attempt recording, during attachment rename, during backup, and during migration. Assert no acknowledged mutation disappears, no duplicate domain effect occurs, unmarked outbox rows replay, stale Lease holders fail, attachments are either absent with no metadata or present with valid checksums, and the singleton guard prevents a second owner.

Run concurrent claim races with many workers and deterministic `TestClock` lease expiry. Exercise SDK retries with identical idempotency keys and conflicting bodies. Send duplicate and out-of-order signed GitHub deliveries. Revoke a Credential during an active stream. Fill the disk to quota and verify a clear rejection rather than corruption. Restore every fixture into a clean directory and compare domain counts, hashes, Grants, and Event sequences.

## 11. Upgrade and release mechanics

Ship one immutable release artifact containing server, browser assets, and migration entrypoint. Run one migration job before serving. Use forward-only expand/contract migrations so the previous application version remains compatible during a controlled rollback. Before upgrade, take a fresh snapshot and rehearse in staging. During upgrade, drain requests and streams, run migrations, verify schema version and integrity, then start the new process. Retain the previous image; roll back code only when the schema is compatible, otherwise restore the verified snapshot or apply a forward fix.

Pin Bun 1.4.5 and all Effect packages to 4.1.5 for v1. Upgrades are deliberate releases with changelog, compatibility report, and restore rehearsal—not automatic floating dependency updates.

## 12. First tickets to create in P-STR-10

- Protocol freeze and executable conformance harness.
- One-process Bun/Effect/SQLite skeleton with singleton guard and write gate.
- OIDC bootstrap, Invitation, Actor, Credential, and Grant vertical slice.
- Transactional Message/Event/outbox implementation with Revisions and Tombstones.
- Atomic Task Lease and Dependency state machine.
- Inspector snapshot plus SSE cursor catch-up.
- Board-compatible façade and error mapping.
- GitHub inbound inbox, Sync Mapping, Promotion, retry, dead-letter, and replay.
- Export Archive, attachment bundle, backup, and restore drill.
- Observability, quotas, security tests, load/crash tests, and beta runbooks.

Do not create tickets for deferred infrastructure or PM-suite features unless a kill/pivot decision changes the product boundary.

</details>

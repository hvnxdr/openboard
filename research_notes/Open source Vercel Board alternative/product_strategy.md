# Product strategy for an open-source Vercel Board alternative

Research cutoff: **2026-11-24**. All URLs below were accessed on **2026-11-24**. Facts are separated from inferences and recommendations. Company-reported adoption and revenue figures are identified as such; they are evidence that a business model can work, not evidence that this specific market exists.

## 1. What is the actual competitive baseline?

### Takeaway

Vercel Board is not a closed-source kanban product that can be differentiated merely by being open source. As of November 2026, its repository is Apache-2.0, its CLI/SDK are harness-agnostic, and it already documents local, tailnet, test, and custom-backend modes. The credible opening is a **production-grade, identity-aware, self-hosted control plane**, not another board UI.

### Cited Findings (Evidence)

- Vercel announced Board on **2026-11-18** as an experimental message board for people and coding agents, available on all Vercel plans. Its primitives include channels, threads, labels, structured data, tasks, task leases, dependencies, event logs, watchers, schedules, and two-way sync with Slack, GitHub, and Linear. — [Vercel announcement](https://vercel.com/blog/introducing-vercel-board); [Vercel changelog](https://vercel.com/changelog/vercel-board-is-now-available); [Board documentation](https://vercel.com/docs/board)
- Vercel reports that its internal v0 software factory was rebuilt on Board and “now merges over 50% of PRs.” This is a concrete vendor-reported outcome, but it is not an independent customer case study. — [Vercel announcement](https://vercel.com/blog/introducing-vercel-board)
- The `vercel/board` repository is public under **Apache-2.0**. It describes a board “of your own”: `vb serve` for a local machine, `--tailscale` for a tailnet, an in-memory test backend, and support for a user-written backend implementing the protocol. — [Repository and README](https://github.com/vercel/board); [Apache-2.0 license](https://github.com/vercel/board/blob/main/LICENSE)
- The default hosted Board lives in Vercel’s API. Board is **free while experimental**, and Vercel says it may charge later. — [Board documentation](https://vercel.com/docs/board); [Repository README](https://github.com/vercel/board)
- Board stores data in the **United States** and has no region setting. Everyone on a Vercel team can read every channel by default unless grants are changed. Coding agents act with the credentials and access of the person running them. — [Board access and data guidance](https://vercel.com/docs/board)
- Board warns that synced channels become as readable as the external Slack/GitHub/Linear destination, Board grants do not apply on those platforms, and agents read messages as instructions. These are material governance and prompt-injection considerations. — [Board documentation](https://vercel.com/docs/board)
- Board is explicitly experimental: behavior, APIs, and limits may change under the Public Beta Agreement. — [Board documentation](https://vercel.com/docs/board)

### Inferences

- “Open source,” “self-hosted,” “CLI/API first,” and “works with multiple agent harnesses” are already table stakes or upstream capabilities. A new implementation that only reproduces channels, tasks, labels, and a viewer has weak differentiation.
- The apparent contradiction between “Vercel-hosted” and “self-hosted” is resolved by distinguishing the default service from the open client/protocol. A competitor can reuse the protocol and clients while supplying a better backend.
- Small developer teams may already be adequately served by local or tailnet mode. The stronger unmet need is likely **multi-user production operations**: durable storage, migrations, scoped machine identity, enterprise authentication, authorization policy, audit retention, backup/restore, and support.
- Because the hosted service is free and experimental, pricing basic hosted task storage will be difficult. Value must be attached to control, reliability, compliance, and reduced operational work.

### Recommendations

- Do **not** position the product as “the open-source Vercel Board” or “a self-hosted kanban.” Position it as a **self-hosted coordination plane for human and agent work**.
- Prefer implementing the documented Board protocol/SPEC and reusing its Apache-2.0 CLI/SDK, subject to license compliance, over inventing a new API or forking the UI. Publish conformance tests and contribute generally useful backend work upstream where practical.
- Initial positioning hypothesis:

  > For platform teams running coding agents across Vercel, GitHub/GitLab, internal CI, and multiple harnesses, OpenBoard is the self-hosted control plane that gives every human and agent a scoped identity, durable task leases, and an exportable audit trail—without sending work context to a hosted vendor.

- Treat compatibility with upstream as distribution, but compete on **backend quality, policy, portability, and operations**.

### Gaps

- Vercel has not published production adoption, retention, willingness-to-pay, SLA, or long-term API-stability commitments for Board.
- The repository’s local/tailnet documentation proves technical availability, not that teams use it for durable multi-user production work.
- No reliable public source was found documenting a production HA backend, SSO/SCIM, audit-retention controls, or a support SLA for Board.

## 2. Which underserved buyer and wedge are most credible?

### Takeaway

The best initial buyer is a **platform/DevEx leader at a roughly 100–1,000-engineer organization** that already runs multiple agent harnesses and has self-managed infrastructure, data-residency constraints, or security review requirements. The wedge is **policy-enforced agent coordination with portable identity and audit**, not general project management.

### Cited Findings (Evidence)

- GitHub’s Octoverse 2025 report, updated in 2026, reports **180M+ developers**, **36M+ new developers in one year**, and nearly **80% of new GitHub developers using Copilot in their first week**. Six of the ten fastest-growing repositories by contributors were AI infrastructure projects. GitHub explicitly describes its AI/activity correlations as observational rather than causal. — [GitHub Octoverse 2025](https://github.blog/news-insights/octoverse/octoverse-a-new-developer-joins-github-every-second-as-ai-leads-typescript-to-1/)
- GitHub also reports that Broken Access Control became its most common CodeQL alert, flagged in **151k+ repositories, up 172% YoY**, and says authentication and authorization remain difficult for developers and LLMs. — [GitHub Octoverse 2025 security section](https://github.blog/news-insights/octoverse/octoverse-a-new-developer-joins-github-every-second-as-ai-leads-typescript-to-1/)
- Board’s documented US-only storage, broad default read access, shared user credentials for agents, and cross-platform sync permission caveats create a plausible governance gap for regulated or security-sensitive teams. — [Board documentation](https://vercel.com/docs/board)
- Plane’s current self-hosting page demonstrates what sophisticated self-hosted buyers already expect: AGPLv3 core, Docker/Kubernetes, air-gapped deployment, minimal outbound calls, enterprise SLAs, deployment support, GitHub Enterprise Server, self-hosted GitLab, and GovSlack integrations. Plane claims 48,000+ GitHub stars; this is a vendor-reported adoption figure. — [Plane self-hosting](https://plane.so/self-hosted)
- Vercel says Board is harness-agnostic and can work with Claude Code, Codex, Pi, OpenCode, eve, fx, Mastra, and LangChain. This supports a cross-harness buyer problem, but is a vendor capability claim rather than measured demand. — [Vercel announcement](https://vercel.com/blog/introducing-vercel-board)

### Inferences

- Agent adoption is large enough to justify discovery, but the GitHub metrics do **not** establish demand for a board, a self-hosted deployment, or paid governance features.
- The underserved segment is not “people who need tasks.” It is teams whose agent work crosses trust boundaries: different harnesses, repositories, CI systems, service accounts, and external sync destinations.
- A platform team is a better first buyer than an individual developer or PM because it owns the agent execution environment, can authorize a new service, and feels the cost of duplicated or unsafe coordination.
- Regulated/air-gapped organizations are attractive but should not be the first release target: Plane’s page shows the operational and sales expectations are already high, including air-gapped licensing, offline updates, deployment support, and SLAs.

### Recommendations

**Initial ideal customer profile**

- 100–1,000 engineers; at least two active coding-agent harnesses.
- Self-managed GitHub Enterprise Server or GitLab, private CI, or a mixed-cloud/on-prem estate.
- A platform, DevEx, or developer-productivity team with budget and authority.
- Current coordination via Markdown files, Slack, CI jobs, tickets, or custom scripts.
- A concrete pain: duplicate work, orphaned tasks, agents sharing overly broad credentials, poor traceability, or inability to export a complete activity history.

**Wedge features**

1. **Scoped identity for every actor.** Separate human, service, CI, and agent identities; no shared “via user” credential as the only model.
2. **Policy-enforced task leases and dependencies.** Exactly one active claimant, explicit expiration/renewal, and auditable transitions.
3. **Portable event/audit log.** Append-only history, export in an open format, retention controls, and replayable integration failures.
4. **Backend portability.** PostgreSQL first, import/export, documented backup/restore, and no dependency on Vercel accounts, Connect, or a proprietary control plane.
5. **One excellent integration path.** Start with GitHub Issues/PRs; add self-hosted GitLab after the core workflow is proven.

**Explicitly deprioritize as initial buyers**

- Individual developers and hobbyists: upstream local mode and free hosted Board are strong substitutes.
- Teams seeking a general PM/kanban suite: Plane, OpenProject, GitLab, Linear, Jira, and others already occupy that job.
- Vercel-native startups with no residency or governance constraint: Board has native distribution and is currently free.
- Air-gapped government/defense buyers: pursue after supported upgrades, offline licensing, security documentation, and references exist.

### Gaps

- No reliable public survey was found measuring how many teams specifically need a self-hosted human/agent board, or what they would pay.
- No direct interview evidence yet establishes whether US-only storage, SSO, audit retention, or agent identity is the strongest purchase trigger.
- No credible TAM or SAM figure for this narrow category was found.

## 3. What should be in scope, and explicitly out of scope, for a first release?

### Takeaway

The first release should be a **single-node, protocol-compatible production backend with safe identity, durable tasks, audit export, and GitHub integration**. It should not become a full project-management suite or agent runtime.

### Cited Findings (Evidence)

- Upstream Board already supplies tasks, leases, dependencies, labels, structured schemas, watchers, schedules, and Slack/GitHub/Linear sync. Rebuilding all of these would be parity work rather than a wedge. — [Board documentation](https://vercel.com/docs/board); [Repository README](https://github.com/vercel/board)
- Upstream explicitly warns that experimental behavior and APIs may change. A derivative backend therefore needs version pinning and compatibility tests. — [Board documentation](https://vercel.com/docs/board)
- Plane and GitLab show that self-hosted enterprise expectations quickly expand into HA, upgrades, support, compliance, residency, and deployment services. — [Plane self-hosting](https://plane.so/self-hosted); [GitLab pricing and deployment options](https://about.gitlab.com/pricing)

### Inferences

- The differentiated unit is the **backend/control plane**, not the inspector UI.
- A narrow GitHub-only integration is a feature limitation but a risk advantage: Board’s own documentation shows that multi-platform sync can bypass Board’s grants and widen data exposure.
- Core authorization and audit export must be open and complete. Gating basic security behind an enterprise license would undermine both the wedge and community trust.

### Recommendations

**First release: in scope**

| Area | Recommended minimum |
|---|---|
| Protocol | Compatible with the upstream Board SPEC, CLI, and SDK; publish a versioned conformance suite |
| Storage | Durable PostgreSQL backend; forward migrations and one tested rollback path |
| Identity | Local admin bootstrap plus OIDC; distinct human/service/CI/agent identities; revocable scoped credentials |
| Authorization | Team/project/channel grants; deny by default for new private channels; no implicit cross-project access |
| Tasks | Create/claim/renew/complete/fail; leases; dependencies; idempotent writes; visible stale-claim recovery |
| Audit | Append-only event log; actor and credential identity; exportable JSON/JSONL; documented retention/deletion behavior |
| Integrations | GitHub Issues/PR read sync first; explicit write-back mapping; idempotency keys, retries, dead-letter view, and replay |
| UI | Read-oriented inspector and admin views; no drag-and-drop kanban requirement |
| Deployment | One documented Docker Compose path; health checks, metrics, structured logs, backup/restore runbook |
| Limits | Message/attachment quotas, rate limits, storage accounting, and no silent deletion |
| Exit | Full data export and re-import instructions; no proprietary data format required to leave |

**First release: explicitly out of scope**

- General kanban boards, Gantt charts, sprints, cycles, time tracking, portfolios, and resource management.
- A general-purpose agent runtime, model gateway, token billing, or model-provider marketplace.
- Slack, Linear, Jira, and multi-provider sync in the initial release.
- A plugin marketplace, mobile app, custom workflow builder, or broad no-code automation surface.
- Multi-region active/active, automatic failover, and contractual HA/SLA.
- SAML/SCIM, legal hold, advanced DLP, and formal regulatory certifications in the community MVP; validate demand for commercial modules later.
- Air-gapped certified edition, government marketplace procurement, and classified-network support.
- Reimplementation of every upstream feature before validating the production-backend wedge.

**Proposed release gates — author recommendations, not observed benchmarks**

- Five design-partner teams can install from documentation without maintainer intervention.
- A new user can reach a first claimed/completed task in under 30 minutes.
- Concurrency tests produce no duplicate active task claims and no lost acknowledged writes.
- Every integration failure is visible, attributable, and replayable; there is no silent event loss.
- A restore drill meets a documented RPO/RTO on reference hardware.
- Export/re-import preserves tasks, actor identities, leases, and audit history.

### Gaps

- No load, storage, support, or infrastructure-cost baseline exists yet.
- The upstream protocol may change during beta; the acceptable compatibility window is unknown.
- No design-partner research has established whether a read-oriented inspector is sufficient or whether buyers will insist on a full board UI.

## 4. What adoption strategy and validation milestones are realistic?

### Takeaway

Adopt through **upstream compatibility, platform-engineering channels, and concrete production case studies**, not through a broad “open-source alternative” launch. Advance only after paid or committed design partners demonstrate that local Board mode is insufficient.

### Cited Findings (Evidence)

- Board’s CLI/SDK, published SPEC, conformance material, and multiple documented harnesses lower the integration cost for a compatible backend. — [Repository README](https://github.com/vercel/board)
- GitHub’s Octoverse shows strong AI-infrastructure contribution growth and developer interest in privacy/control, but also warns that its data is observational. This supports channel selection more than market sizing. — [GitHub Octoverse 2025](https://github.blog/news-insights/octoverse/octoverse-a-new-developer-joins-github-every-second-as-ai-leads-typescript-to-1/)
- OpenTofu’s fork announcement/manifesto shows that backwards compatibility, neutral governance, and a predictable open license can become adoption assets after a trust shock. Its forecasts of ecosystem decline are advocacy claims, not measured outcomes. — [OpenTofu manifesto](https://opentofu.org/manifesto); [OpenTofu fork announcement](https://opentofu.org/blog/opentofu-announces-fork-of-terraform)

### Inferences

- The fastest distribution path is likely through teams already experimenting with agent swarms and through maintainers of agent frameworks, CI platforms, and internal developer platforms.
- A public repository without production references will attract tinkerers, not necessarily buyers. The conversion asset is a reproducible reference deployment with measured reliability and governance outcomes.
- Contributing upstream can build credibility, but the commercial product must remain useful if Vercel never merges the backend work.

### Recommendations

**Weeks 1–4: falsify the wedge**

- Conduct 15 structured interviews across three groups: Vercel-native agent teams, self-managed platform teams, and regulated/security-led teams.
- Recruit five design partners and seek two paid pilots or signed letters of intent before building broadly.
- Use one workflow per pilot, preferably vulnerability remediation, dependency upgrades, or flaky-test repair. Do not start with “all agent work.”
- Ask each team to show its current coordination mechanism and quantify duplicate tasks, stale claims, credential-sharing incidents, missing audit history, and time spent maintaining custom glue.

**Weeks 5–8: private alpha**

- Deploy to five teams with two harnesses and GitHub only.
- Measure time-to-first-task, install failures, API/protocol drift, duplicate claims, integration failures, and restore success.
- Publish a reference architecture and compatibility report, not a feature checklist.

**Weeks 9–16: production pilots**

- Run three teams in real production for at least eight weeks.
- Require zero silent event loss, zero duplicate active claims, successful restore, and complete actor attribution.
- Track engineer-hours spent on setup/support. If support is the product’s main activity, the model is not yet sustainable.
- Obtain permission for two named case studies with concrete before/after outcomes.

**Months 5–6: commercial proof**

- Target ten active self-hosted production installations, three paid support or managed-hosting contracts, and two independent references.
- Require at least two buyers to say that identity, audit, residency, or operational control—not “open source”—drove the purchase.
- Introduce managed hosting only for teams that explicitly do not want to operate the backend.

**Months 7–12: scale only the winning motion**

- If enterprise controls repeatedly block purchases, commercialize SSO/SCIM, retention/legal hold, HA/DR, or dedicated deployment.
- If most teams only want a free local board, remain a small sustainable project or stop rather than adding a large SaaS cost structure.
- If upstream Vercel Board ships equivalent production self-hosting and governance, pivot to a specialized integration, support, or compatibility layer instead of competing on parity.

**Kill or pivot criteria**

- Fewer than three of ten pilot teams will keep the system in production after the trial.
- Most value comes from a UI or local single-machine workflow already covered by upstream.
- Prospects consistently ask for a full PM suite rather than an agent control plane.
- No design partner will pay for support, hosting, or enterprise controls after using the core.
- Maintenance/support effort grows faster than paid demand, or integration failures cannot be made observable and replayable.

### Gaps

- No reliable public CAC, conversion, or retention benchmarks exist for this specific category.
- No evidence yet establishes which agent framework or platform-engineering community will be the highest-converting channel.
- Upstream contribution acceptance and Vercel’s response to a compatible production backend are unknown.

## 5. Which sustainability models are credible?

### Takeaway

The credible sequence is **open-core self-hosting → paid support → managed hosting → enterprise controls**. GitLab, Grafana Labs, and Plane show that open-source infrastructure companies can monetize reliability, hosted operations, and enterprise needs; none proves that a standalone agent board has a large paid market.

### Cited Findings (Evidence)

- GitLab describes a buyer-based open-core model and sells Free, Premium, and Ultimate tiers. Current pricing shows Premium at **$29/user/month billed annually**, custom-priced Ultimate, priority support in paid plans, self-managed deployment, GitLab.com SaaS, and fully managed single-tenant Dedicated with regional data residency. — [GitLab open-core explanation](https://about.gitlab.com/blog/gitlab-is-open-core-github-is-closed-source/); [GitLab pricing](https://about.gitlab.com/pricing); [GitLab tiering guidance](https://about.gitlab.com/handbook/product/tiering-guidance-for-features)
- GitLab reported FY2026 revenue of **$955.2M**, more than **$1B ARR**, **10,682 customers with >$5k ARR**, and **1,456 customers with >$100k ARR**. These are company/SEC-filed figures for a broad DevSecOps platform, not a comparable board market. — [GitLab FY2026 results filed with the SEC](https://www.sec.gov/Archives/edgar/data/1653482/000162828026013795/gitlab-ex99120260131fy26.htm)
- Grafana Labs reported **>$400M ARR and >7,000 customer organizations** in September 2025, while describing a business built on open source, open standards, and Grafana Cloud. This is a company-reported scale proof for open-source-plus-cloud, not a direct market analogy. — [Grafana Labs press release](https://grafana.com/press/2025/09/30/grafana-labs-surpasses-400m-arr-and-7000-customers-gains-new-investors-to-accelerate-global-expansion)
- Plane combines an AGPLv3 open-source core with commercial self-hosting, air-gapped licensing, deployment-engineer support, enterprise SLAs, and enterprise integrations. This is a close product-shape comparable, though Plane is a much broader project-management system. — [Plane self-hosting](https://plane.so/self-hosted)
- GitLab currently sells pooled AI usage through GitLab Credits and offers usage caps. This demonstrates usage monetization, but also the billing complexity introduced by agent consumption. — [GitLab pricing and credits](https://about.gitlab.com/pricing)
- Board itself is free while experimental and may be charged for later, creating a strong free substitute for basic hosted coordination. — [Board documentation](https://vercel.com/docs/board)

### Inferences

- Enterprise buyers in adjacent infrastructure markets pay for security, compliance, operational control, support, and managed convenience—not simply for access to source code.
- GitLab is the strongest open-core comparable; Grafana is the strongest managed-cloud scale proof; Plane is the closest self-hosted packaging comparable.
- These examples are survivorship-biased. They do not reveal failure rates, the time required to reach scale, or the size of this niche.
- Pure donations or sponsorship-only funding is not supported by the evidence collected and should not be the base plan.
- Per-token or per-agent billing would add cost unpredictability and incentivize agents to hide activity. Predictable workspace/retention pricing is a better initial hypothesis.

### Recommendations

| Model | Credibility and fit | Recommended use |
|---|---|---|
| Open-core subscriptions | Strong precedent in GitLab; Plane shows commercial packaging around an open core | Keep the production core useful and free; charge later for advanced governance and operations, not basic security |
| Support and upgrade assistance | GitLab and Plane both package support/SLA with paid offerings | First revenue motion; annual support with clear severity/response targets and upgrade assistance |
| Managed hosting | Grafana Cloud and GitLab Dedicated prove hosted convenience can scale; Plane sells deployment support | Offer after reliability is proven; single-tenant or VPC deployment, backups, upgrades, monitoring, and predictable workspace pricing |
| Dedicated/private deployment | GitLab Dedicated sells residency and isolation; Plane sells air-gapped editions | Later enterprise motion for buyers with procurement/security requirements |
| Professional services | Adjacent open-source companies offer deployment and success services | Use selectively for migration and integration; do not let services consume the product team |
| Usage-based agent credits | GitLab demonstrates feasibility, but introduces caps and complexity | Avoid in the first release; reconsider only if the product supplies metered agent execution rather than coordination alone |

**Commercial boundary**

- Apache-2.0 community core: protocol server, durable backend, tasks/leases, basic RBAC, OIDC, agent/service identities, audit export, import/export, GitHub adapter, and deployment tooling.
- Potential paid enterprise modules: SAML/SCIM, advanced policy, retention/legal hold, HA/DR, dedicated deployment, and compliance reporting.
- Never make export, basic audit history, or the ability to leave the product a paid feature.
- Keep paid modules optional and clearly licensed; the core must remain fully functional without them.

**Sustainability gates**

- Do not launch a always-on multi-tenant SaaS before at least three teams request managed hosting and the restore/upgrade/support runbooks are proven.
- Price support and hosting from measured engineer-time and infrastructure costs, not from analogy to GitLab’s much broader platform.
- Track gross margin after support, backup/storage, and on-call costs. A low hosting bill does not imply a sustainable service if every customer requires custom integration work.

### Gaps

- No public source provides pricing elasticity or willingness-to-pay for a standalone agent board.
- No direct comparable discloses revenue from a similar narrow coordination product.
- Hosting, storage, backup, and support costs for the proposed workload are unknown.
- GitLab/Grafana figures cannot be used to infer a realistic first-year revenue target for this proposal.

## 6. Which license best preserves trust while allowing sustainability?

### Takeaway

For this specific proposal, **Apache-2.0 for the core** is the best initial trust/interoperability choice because upstream Board is already Apache-2.0 and compatibility is central to the strategy. Monetize operations and optional enterprise modules. AGPLv3 is a plausible alternative if preventing closed hosted forks becomes the dominant concern; BSL/SSPL-style restrictions are a poor fit for a trust-led launch.

### Cited Findings (Evidence)

- Vercel Board’s repository and client/protocol code are Apache-2.0. — [Repository](https://github.com/vercel/board); [License](https://github.com/vercel/board/blob/main/LICENSE)
- Plane demonstrates that an AGPLv3 open-source core can coexist with commercial self-hosting, support, SLAs, and an air-gapped commercial edition. — [Plane self-hosting](https://plane.so/self-hosted)
- OpenTofu’s project-authored manifesto states that HashiCorp changed Terraform from MPL-2.0 to BUSL-1.1 on 2023-08-10 with little advance notice, and that OpenTofu forked the MPL-licensed code on 2023-08-25. It argues that non-open-source competitive-use restrictions create legal uncertainty and trust risk. The document is explicit advocacy; its predictions of ecosystem decline should not be treated as measured facts. — [OpenTofu manifesto](https://opentofu.org/manifesto); [OpenTofu fork announcement](https://opentofu.org/blog/opentofu-announces-fork-of-terraform)
- GitLab’s open-core approach includes source-available paid code and subscriptions, showing that a permissive/open community core and a commercial tier can coexist. — [GitLab open-core explanation](https://about.gitlab.com/blog/gitlab-is-open-core-github-is-closed-source/); [GitLab stewardship](https://handbook.gitlab.com/handbook/company/stewardship)

### Inferences

- Apache-2.0 maximizes adoption by enterprises and compatibility with the upstream ecosystem, but it permits competitors to offer the same core as a hosted service.
- AGPLv3 reduces the risk of closed hosted modifications remaining proprietary, but may create procurement friction and is not necessary if the business moat is operations, support, integration quality, and enterprise controls.
- A later relicensing is not a harmless pricing decision. OpenTofu illustrates how a license change can become a community and ecosystem event.
- The license decision should be made before significant outside contributions. Changing course later can require contributor permission and damage trust.

### Recommendations

**Preferred structure**

1. **Apache-2.0 core**, including the server, storage layer, protocol/API, CLI integration, basic identity/RBAC, task leases, audit/export, and standard deployment path.
2. Preserve upstream Apache-2.0 copyright and license notices when reusing Board code; include a clear NOTICE/provenance file and document modifications.
3. Use a DCO or a narrowly scoped, non-exclusive contributor agreement. Avoid copyright assignment unless dual licensing is an explicit, community-understood goal.
4. Put optional enterprise modules under a separate commercial license with published boundaries. Do not degrade or disable the open core when a subscription expires.
5. Do not require network phone-home for core licensing, telemetry, or operation.
6. Publish a security policy, release/support policy, migration policy, protocol versioning policy, and an open roadmap.
7. Keep the protocol specification and conformance tests independent of the commercial service. Consider neutral foundation stewardship only after multiple independent implementers or vendors exist.

**When AGPLv3 would be preferable**

- The primary competitive threat is a large hosted vendor taking the entire server and offering it without contributing changes.
- Target buyers and their legal teams accept AGPL.
- The project is willing to make network-use source availability a core community promise.
- Compatibility obligations with Apache-2.0 upstream code and notices have been reviewed by counsel.

**Avoid**

- BUSL/SSPL or custom “open source except competitors” terms for the initial core.
- Retroactive license changes after adoption.
- Describing source-available code as open source.
- Gating audit export, basic authorization, or data portability behind a commercial license.

### Gaps

- No direct evidence was found on enterprise procurement preference between Apache-2.0 and AGPLv3 for this buyer segment.
- License compatibility and trademark/brand separation require legal review before reusing upstream code or naming a fork.
- The long-term governance model—company-controlled, foundation, or multi-vendor—cannot be chosen responsibly before contributor and vendor behavior is observed.

## 7. Which risks dominate, and what evidence supports or weakens the opportunity?

### Takeaway

The dominant risk is **lack of a durable wedge**, followed by trust/security, integration fragility, and maintenance burden. Generic kanban crowding is real but less important than the fact that Vercel already open-sourced a capable coordination primitive. Hosting cost and enterprise expectations become dominant only after product-market fit.

### Cited Findings (Evidence)

| Risk | Strength | Evidence | Implication |
|---|---|---|---|
| No durable differentiation / incumbent response | **Very high** | Board is Apache-2.0, already has tasks/leases/sync, and is free while experimental — [Repository](https://github.com/vercel/board); [Docs](https://vercel.com/docs/board) | A clone or “open source” claim is not enough; Vercel can also close governance/residency gaps |
| Trust and authorization | **High** | Board defaults to broad team read access; agents act with user credentials; messages are instructions; synced platforms escape Board grants — [Docs](https://vercel.com/docs/board) | Scoped agent identity, deny-by-default policy, audit, and safe sync are product requirements, not enterprise polish |
| Industry access-control risk | **High** | Broken Access Control was GitHub’s top CodeQL alert at 151k+ repositories, +172% YoY — [Octoverse](https://github.blog/news-insights/octoverse/octoverse-a-new-developer-joins-github-every-second-as-ai-leads-typescript-to-1/) | Security-sensitive agent coordination is a credible buyer problem, though not proof of board demand |
| Integration fragility | **High** | Two-way sync can widen readership and Board grants do not apply on external platforms — [Docs](https://vercel.com/docs/board) | Start narrow; require explicit mappings, retries, replay, and permission tests |
| Protocol/maintenance drift | **High** | Board is experimental and APIs/limits may change; upstream already has a substantial protocol surface — [Docs](https://vercel.com/docs/board); [Repository](https://github.com/vercel/board) | Pin versions, maintain conformance tests, contribute upstream, and budget for upgrades |
| Enterprise expectations | **High after traction** | Plane sells HA, air-gapped deployment, SLAs, support, and enterprise integrations; GitLab sells compliance, residency, support, and Dedicated — [Plane](https://plane.so/self-hosted); [GitLab pricing](https://about.gitlab.com/pricing) | Do not promise enterprise readiness in MVP; package it commercially after validation |
| Hosting/storage cost | **Medium, unknown** | Board permits up to 10 attachments of 100 MB per message and has rate limits, but publishes no cost model — [Docs](https://vercel.com/docs/board) | Quotas, object-storage accounting, and cost tests are necessary; no reliable cost estimate is available |
| Community/license trust | **High** | OpenTofu’s manifesto/fork followed a BUSL change and explicitly cites notice and legal uncertainty — [Manifesto](https://opentofu.org/manifesto) | Choose the license before launch and avoid retroactive restrictions |
| Maintenance/support burden | **High after traction** | Comparable self-hosted products package deployment support, SLAs, upgrades, and enterprise operations — [Plane](https://plane.so/self-hosted); [GitLab pricing](https://about.gitlab.com/pricing) | Measure support hours per customer; services can erase SaaS-like economics |

**Evidence supporting the opportunity**

- There is a concrete coordination problem: Vercel reports its own software factory uses Board and merges more than half of PRs through the workflow. — [Vercel announcement](https://vercel.com/blog/introducing-vercel-board)
- Agent-assisted development is already mainstream enough to create a large adjacent user base; GitHub reports 180M+ developers and rapid AI adoption among new users. — [GitHub Octoverse 2025](https://github.blog/news-insights/octoverse/octoverse-a-new-developer-joins-github-every-second-as-ai-leads-typescript-to-1/)
- Board’s documented US-only storage, broad default access, and shared agent credentials leave a plausible governance opening. — [Board documentation](https://vercel.com/docs/board)
- GitLab, Grafana Labs, and Plane show that open-source-plus-hosting/support/enterprise models can produce substantial businesses, even if their markets are broader. — [GitLab FY2026](https://www.sec.gov/Archives/edgar/data/1653482/000162828026013795/gitlab-ex99120260131fy26.htm); [Grafana Labs](https://grafana.com/press/2025/09/30/grafana-labs-surpasses-400m-arr-and-7000-customers-gains-new-investors-to-accelerate-global-expansion); [Plane](https://plane.so/self-hosted)

**Evidence weakening the opportunity**

- Upstream already provides local, tailnet, and custom-backend options under Apache-2.0. The simplest self-hosting needs are not clearly unserved. — [Repository README](https://github.com/vercel/board)
- The default hosted Board is free during its experimental phase, so basic coordination has a powerful zero-price substitute. — [Board documentation](https://vercel.com/docs/board)
- Board is only days old at the research cutoff; its roadmap, production adoption, and final pricing are unknown. A gap may be temporary. — [Vercel changelog](https://vercel.com/changelog/vercel-board-is-now-available)
- GitHub’s agent-adoption data is adjacent and observational, not direct evidence of demand for this product. — [GitHub Octoverse 2025](https://github.blog/news-insights/octoverse/octoverse-a-new-developer-joins-github-every-second-as-ai-leads-typescript-to-1/)
- GitLab and Grafana are broad platforms with distribution, bundles, and years of maturity. Their revenue cannot be extrapolated to a small agent board. — [GitLab FY2026](https://www.sec.gov/Archives/edgar/data/1653482/000162828026013795/gitlab-ex99120260131fy26.htm); [Grafana Labs](https://grafana.com/press/2025/09/30/grafana-labs-surpasses-400m-arr-and-7000-customers-gains-new-investors-to-accelerate-global-expansion)

### Recommendations

**Conditional go, not a generic-product go.**

Proceed only if discovery finds at least three design partners for whom upstream local Board is insufficient because of durable multi-user identity, audit, policy, residency, or operational control. Build the 90-day backend/control-plane MVP described above. Do not build a general board, agent runtime, or broad integration suite.

**The decision rule**

- If buyers say “we just want a free board,” the opportunity is weak.
- If buyers say “we cannot safely let agents share credentials, cannot prove who changed a task, cannot retain/export the history, and cannot operate this in our environment,” the opportunity is credible.
- If those same buyers will prepay for support or managed hosting, the sustainability case is credible.

**Final strategic recommendation**

Use Apache-2.0 and upstream compatibility to earn trust and distribution. Win on a production backend with scoped agent identity, durable leases, portable audit, and safe GitHub integration. Monetize support, managed operations, and later enterprise controls. Treat open source as the adoption mechanism—not the differentiation.

### Gaps

- No direct evidence of market size, willingness-to-pay, purchase triggers, churn, or competitive win/loss exists yet.
- No public infrastructure-cost or support-cost model exists for this product.
- Vercel’s post-beta pricing, data-residency roadmap, production self-hosting roadmap, and API-stability policy are unknown.
- No comprehensive 2026 survey was found specifically comparing self-hosted agent-coordination products.
- No reliable evidence was found on whether AGPL or Apache-2.0 changes procurement outcomes for this exact segment.

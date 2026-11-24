# Issue tracker: Linear

Issues and specs for this repo live in Linear. GitHub is code hosting only; do not create GitHub issues for project work.

## Location

- **Team:** STARHACKED (`STR`)
- **Project:** OpenBoard (`P-STR-10`)
- **Project URL:** https://linear.app/heavenaldrico/project/openboard-dfcb51509e0d
- **Repository:** https://github.com/hvnxdr/openboard
- **Access:** the project-local `linear` MCP server in `opencode.json`

Use the Linear MCP tools for all tracker operations. In Code Mode, call them through `execute` as `tools.linear.*`. If `opencode mcp list` reports that Linear needs authentication, ask the user to run `/mcps`, select `linear`, and sign in. Never store OAuth tokens or API keys in this repo.

## Conventions

- **Scope:** every OpenBoard issue belongs to team `STARHACKED` and project `P-STR-10`.
- **References:** use Linear issue identifiers such as `STR-123` and full issue URLs. `P-STR-10` identifies the project, not an issue.
- **Check for duplicates:** before creating an issue, call `list_issues` with the team, project, and a relevant `query`.
- **Create an issue:** call `save_issue` with `team: "STARHACKED"`, `project: "P-STR-10"`, `title`, and a Markdown `description`. Add labels, priority, assignee, or dates only when known.
- **Read an issue:** call `get_issue` with its identifier, then `list_comments` with its ID to read the discussion. Use `extract_images` when the description or comments contain screenshots.
- **List issues:** call `list_issues` with appropriate team, project, state, label, assignee, or query filters. Follow `hasNextPage` using the returned cursor.
- **Update an issue:** call `save_issue` with its ID and only the fields being changed. The `labels` field replaces the full label set; use `addLabels` or `removeLabels` for incremental changes.
- **Comment:** call `save_comment` with `issueId` and a Markdown `body`.
- **Sub-issues:** set `parentId` when creating or updating a child issue.
- **Blocking:** use `blockedBy` for issues that block this issue and `blocks` for issues it blocks. Inspect existing relations before adding duplicates.
- **Close:** post the explanation first, then use `save_issue` to set the workflow state. Use `Canceled` for rejected or wontfix work; use `Done` only for completed work. Confirm current team states with `list_issue_statuses` when unsure.

## Triage vocabulary

Use `docs/agents/triage-labels.md` to map canonical triage roles to Linear labels. Workflow states and triage labels are separate; changing one does not imply changing the other.

## Pull requests as a request surface

**PRs as a request surface: no.** GitHub pull requests are not automatically discovered for triage. If a PR should become tracked work, create a Linear issue that links to it.

## When a skill says "publish to the issue tracker"

Create a Linear issue in `STARHACKED` / `P-STR-10`.

## When a skill says "fetch the relevant ticket"

Read the Linear issue and its comments using `get_issue` and `list_comments`.

## Wayfinding operations

- **Map:** a Linear issue in `P-STR-10` labeled `wayfinder:map`.
- **Child ticket:** a Linear issue whose `parentId` is the map issue. Use `wayfinder:research`, `wayfinder:prototype`, `wayfinder:grilling`, or `wayfinder:task`.
- **Blocking:** use Linear issue relations (`blocks` / `blockedBy`).
- **Frontier:** list OpenBoard issues, exclude completed/canceled or assigned issues, and exclude issues with open blockers.
- **Claim:** update the issue with `assignee: "me"`.
- **Resolve:** post the answer, move the issue to `Done`, and link the answer from the map issue.

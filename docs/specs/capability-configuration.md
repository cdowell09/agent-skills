# Capability configuration contract

Date: 2026-09-02
Question: What concrete repository-owned configuration model maps semantic workflow roles to GitHub repositories, Projects, labels, statuses, claiming, dependency edges, commands, paths, verification, browser behavior, evidence, pull-request policy, concurrency, research areas, approval policy, and runtime capabilities; how is setup validated in tiers; and what is the minimal owner-bearing claim?
Resolves: [#8 Define the capability configuration contract](https://github.com/cdowell09/agent-skills/issues/8)
Inputs: [ADR 0001](../adr/0001-npx-skills-distribution-contract.md), [ADR 0002](../adr/0002-skill-boundaries-and-shared-contracts.md), [ADR 0003](../adr/0003-ticket-decomposition-provider.md), [ADR 0004](../adr/0004-licensing-attribution-and-contribution.md), [source skill inventory](../research/2026-07-12-source-skill-inventory.md), [`to-issues` overlap audit](../research/2026-07-12-to-issues-overlap.md), [triage label roles](../agents/triage-labels.md)

Skill names used here (`pick-up-work`, `work-the-board`, `pick-up-human-task`, `work-the-human-board`, `dogfood`, `research-project`, `publish-findings`) are working names; a later naming pass may rename them.

## Decision

- One committed file, `.agent-skills/config.yaml` (`version: 1`), is the only place a consuming repository maps semantic roles to concrete GitHub and runtime facts. Mutation-affecting keys have no catalog defaults; a missing required key stops the skill before side effects with setup guidance.
- `.agent-skills/state/` holds durable receipts. `state/local/` is machine-specific and gitignored (validation receipt, resolved-ID cache, run receipts); the rest of `state/` is committed because other machines must read it (publication receipts for deduplication).
- Tracker identity is human-stable: repository from `origin`, Project by owner and number, fields, options, and labels by name. Opaque node IDs are resolved at runtime into the local receipt and never appear in the config (this is what makes ADR 0004's zero-tolerance identifier scan achievable).
- Validation is tiered: a durable receipt keyed to a fingerprint of the canonical config plus protocol-contract majors; a no-network fast path when the fingerprint is unchanged; and volatile rechecks of auth, permissions, Project existence, name-to-ID resolution, and target health immediately before the first mutation that depends on them.
- A claim is a structured issue comment carrying `issue`, `owner`, `claimed_at`, and optional `expires_at`. The comment is authoritative; Project status is a mirror. Only the owner releases; abandoned claims are recoverable after expiry under a configured policy.
- Contract versions are independent integer majors (`config`, `claim`, `receipt.*`, `findings`, `publication`); consumers tolerate additive fields and stop on unknown majors.
- `findings.approval.mode` has no default and must be written by the repository owner.

## File location and layout

```text
<repository>/
├── .agent-skills/
│   ├── config.yaml              # committed; the contract this document defines
│   └── state/
│       ├── .gitignore           # written by setup; contains "local/"
│       ├── publications/        # committed publication receipts (publish-findings, #18)
│       └── local/               # gitignored, machine-specific
│           ├── validation.json  # durable validation receipt
│           ├── resolved.json    # name -> ID cache with resolved_at
│           └── runs/<owner-id>.json   # worker run receipts, monitoring ledger
├── <findings.artifact_dir>/     # committed findings artifacts (default docs/findings)
└── <research.ledger_path>       # committed research ledger (default docs/research/LEDGER.md)
```

Why the split: the validation receipt records which account, host, and token scopes passed and caches resolved IDs; committing it would let one developer's validation vouch for another machine and would put opaque IDs in the tree. Publication receipts must be shared or deduplication breaks across machines, so they are committed with the findings PR that filed them. Findings artifacts and the research ledger are product documentation, so their locations are configured rather than fixed under `state/`. If the repository already ignores `.agent-skills/` wholesale, setup warns that publication receipts will not be shared and asks the user to narrow the rule; it never edits the root `.gitignore`.

## Configuration schema

Complete commented example. "required" means required for the skills named in the matrix; other keys are optional with the stated default. Unknown keys warn and are ignored. Durations accept `30m`, `4h`, `2d` or ISO-8601. Paths are repository-relative and may not escape the repository. Labels match case-sensitively; status option names case-insensitively (ambiguity is a validation error). Example hosts use the reserved `example.com`.

```yaml
version: 1                       # required; config schema major

repository:                      # all optional; discovered from `origin` and the API, recorded in the receipt
  owner: acme                    # override only when origin is not the tracker repository
  name: widgets
  default_branch: main           # validated against the API when present
  visibility: private            # public | private; drives evidence embedding

tracker:
  project:                       # optional for pick-up-work; required for both coordinators and the human worker
    owner: acme                  # user or organization owning the Project
    number: 7                    # from the Project URL; never the node ID
  fields:                        # field roles -> field names, resolved to IDs at runtime
    status: Status               # required with project
    priority: Priority           # optional
    theme: Theme                 # optional single-select used when filing findings
  statuses:                      # status roles -> option names of the status field
    todo: Todo                   # required with project
    in_progress: In Progress     # required with project
    in_review: In Review         # optional
    done: Done                   # required with project
  labels:                        # label roles -> repository label names
    needs_triage: needs-triage   # canonical triage roles (docs/agents/triage-labels.md)
    needs_info: needs-info
    ready_for_agent: ready-for-agent
    ready_for_human: ready-for-human   # human-execution work; consumed by no catalog skill (ADR 0003)
    wontfix: wontfix
    needs_decision: needs-decision     # human-decision gates (ADR 0003); consumed by the human-gate pair
    spec_gate: gate:spec               # placeholder; #10 owns gate vocabulary and may add roles
    bug: bug                     # findings roles
    improvement: enhancement
    finding: finding             # optional marker applied to filed findings
    in_progress: ""              # optional display mirror of a claim; empty = unused
  milestones:
    policy: none                 # none | named | any
    active: ""                   # required when policy is named
  dependencies:
    transport: auto              # auto | gh | rest | graphql | body
    # auto tries: gh issue create/edit dependency and sub-issue flags -> REST dependency and
    # sub-issue endpoints -> GraphQL addBlockedBy/addSubIssue -> "Blocked by: #N" / "Part of #N"
    # body convention. Body is written only when every native transport fails and is reported
    # as a downgrade. Native edges are never dual-written to the body.
    sub_issues: native           # native | body | off

claims:
  representation: comment        # only value in claim protocol v1
  ttl: 4h
  heartbeat: 30m                 # 0 disables
  recovery:
    expired: reclaim             # reclaim | release | stop
    confirm_interactive: true    # unattended runs never prompt
  mirror_status: true            # effective only with tracker.project

work:
  eligibility:                   # required by pick-up-work / work-the-board
    label_roles: [ready_for_agent]
    status_roles: [todo]         # ignored without project
    exclude_label_roles: [needs_info, wontfix]
    require_no_open_blockers: true   # native blocked-by edges must be closed
  priority:
    order: [priority_field, milestone, oldest]   # ordered tie-breakers
  concurrency:
    max_parallel: 3              # required by work-the-board; effective = min(this, runtime capacity)
  worktrees:
    policy: preferred            # required | preferred | off
    root: .worktrees
  base_branch: main              # default repository.default_branch
  branch_pattern: "issue-{number}-{slug}"
  verification:                  # required by pick-up-work: commands, or none: true
    commands:
      - { name: lint, run: npm run lint }
      - { name: test, run: npm test }
    matrix: []                   # optional, e.g. [{ name: node20, env: { NODE_VERSION: "20" } }]
    none: false
  tests:
    policy: behavioral           # behavioral (test-first for behavior changes) | always | never
  review:
    self_review: required        # required | optional (diff review capability)
    issue_fidelity: required     # required | optional (issue-vs-diff subreview)
  pr:
    state: ready                 # required: draft | ready
    template: ""                 # default: discovered under .github/
    monitoring:
      owner: worker              # worker | coordinator | none; #9 decides the lifecycle
      max_duration: 2h           # afterwards the receipt reports monitoring-expired
      on_failed_checks: fix-once # fix-once | report

evidence:
  ui:
    type: screenshot             # screenshot | video | gif | none
    dir: docs/evidence
    embed: auto                  # auto (inline when public, link-only when private) | inline | link-only

browser:
  mode: attach                   # attach | launch | none
  auth:
    boundary: user               # user: a human signs in, the agent never enters credentials | none
    bypass: none                 # none | header:<name>

dogfood:                         # required section for dogfood
  target:
    url: https://staging.example.com
    readiness: { url: https://staging.example.com/health, expect_status: 200 }
    revision_source: none        # none | header | endpoint: how the deployed revision is proven
  journeys:                      # at least one
    - id: onboarding
      routes: ["/signup", "/welcome"]
      changed_paths: ["apps/web/src/onboarding/**"]   # recent-change -> journey mapping
  personas:                      # at least one
    - { id: first-time-user, stance: impatient-novice }
  competitors: []                # [{ name, url }]
  data_policy: observe-only      # observe-only | scratch-account

research:                        # required section for research-project
  areas:
    - id: ingestion
      summary: How raw inputs enter the system
      code: ["packages/ingest/**"]
      docs: ["docs/adr/0003-ingestion.md"]
  ledger_path: docs/research/LEDGER.md
  lens_count: 3                  # effective = min(this, runtime subagent capacity)

findings:
  artifact_dir: docs/findings
  approval:
    mode: interactive            # REQUIRED, no default: interactive | pre-approved-severities | never
    # interactive: a human approves each Proposed Action in-session.
    # pre-approved-severities: file without prompting only severities listed below.
    # never: produce artifacts only; filing is refused.
    auto_file_severities: []
  dedupe:
    scope: [receipts, open_issues, closed_issues]
    closed_window_days: 90

capabilities:
  ticket_decomposition:          # semantics fixed by ADR 0003; key names owned here
    provider: none               # none (complex proposals deferred) | to-tickets (upstream, installed separately) | manual
    user_story_traceability: auto   # auto | off
  # A skill's own required/optional list wins. This section can only mark an optional
  # capability unavailable (forcing its documented fallback) or add adapter notes.
  subagents: auto                # auto | available | unavailable
  worktrees: auto
  browser: auto
  diff_review: auto
  pr_monitoring: auto
  interactive_skill_invocation: auto
  validation_max_age: 30d        # receipt age after which Tier 1 re-runs
  runtime_notes:
    codex: "Parallel subagents via the runtime task facility; cap 2."
    claude_code: "Browser control via the installed browser MCP server; attach to the running profile."
```

## Required keys by consuming skill

R = required, O = optional, D = discovered when absent, - = unused. Coordinators inherit their worker's requirements because they preflight it (ADR 0001).

| Key | pick-up-work | work-the-board | pick-up-human-task | work-the-human-board | dogfood | research-project | publish-findings |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `version` | R | R | R | R | R | R | R |
| `repository.*` | D | D | D | D | D | D | D |
| `tracker.project`, `fields.status`, `statuses.todo/in_progress/done` | O | R | R | R | - | - | O |
| `tracker.labels.ready_for_agent`, `needs_info` | R | R | O | O | - | - | O |
| `tracker.labels.needs_decision`, `spec_gate` | - | - | R | R | - | - | R (`needs_decision`, when `provider` is not `none`) |
| `tracker.labels.ready_for_human` | - | - | O | O | - | - | O |
| `tracker.labels.bug`, `improvement` | - | - | - | - | - | - | R |
| `capabilities.ticket_decomposition` | - | - | O | O | - | - | O |
| `tracker.milestones`, `dependencies`; `claims.*` | O | O | O | O | - | - | O |
| `work.eligibility`, `priority`, `verification`, `pr.state` | R | R | - | - | - | - | - |
| `work.concurrency.max_parallel` | - | R | - | - | - | - | - |
| other `work.*` | O | O | - | - | - | - | - |
| `evidence.ui` | O | O | - | - | R (`dir`) | - | O |
| `browser.*`, `dogfood.*` | O (`browser`) | - | - | - | R | - | - |
| `research.*` | - | - | - | - | - | R | - |
| `findings.approval.mode` | - | - | - | - | R | R | R |
| `findings.artifact_dir`, `dedupe`; `capabilities.*` | O | O | O | O | O | O | O |

`dogfood` and `research-project` require `findings.approval.mode` even without `publish-findings` installed because they file directly when the publisher is absent (ADR 0002).

## Tiered validation

### Tier 1: durable validation receipt

Written to `state/local/validation.json` after a full pass. A full pass runs when the receipt is missing or invalid, or when the user runs setup explicitly.

```json
{
  "receipt_version": 1,
  "fingerprint": "sha256:9f2c…e41a",
  "fingerprint_inputs": {
    "config_sha256": "sha256:71ab…03d9",
    "contracts": { "config": 1, "claim": 1, "receipt.work": 1, "receipt.human": 1, "findings": 1, "publication": 1 }
  },
  "validated_at": "2026-09-02T14:05:12Z",
  "validated_by": { "runtime": "claude-code", "host": "buildbox", "skill": "pick-up-work", "skill_version": "0.3.0" },
  "github": {
    "host": "github.com", "login": "octocat", "scopes": ["repo", "project", "read:org"],
    "repository": { "owner": "acme", "name": "widgets", "default_branch": "main", "visibility": "private", "permission": "admin" }
  },
  "resolved": {
    "project": { "owner": "acme", "number": 7, "id": "<project-node-id>", "resolved_at": "2026-09-02T14:05:12Z" },
    "fields": { "status": { "id": "<field-node-id>", "options": { "Todo": "<option-id>", "In Progress": "<option-id>", "Done": "<option-id>" } } },
    "labels": { "ready-for-agent": true, "needs-info": true }
  },
  "checks": [
    { "id": "gh.auth", "status": "pass" },
    { "id": "gh.scopes", "status": "pass" },
    { "id": "repo.permission", "status": "pass", "detail": "admin >= push" },
    { "id": "project.resolve", "status": "pass" },
    { "id": "labels.resolve", "status": "pass" },
    { "id": "labels.upstream_agreement", "status": "skipped", "detail": "ticket_decomposition.provider is none" },
    { "id": "verification.commands", "status": "pass", "detail": "2 commands found in package.json scripts" },
    { "id": "dogfood.target", "status": "skipped", "detail": "section absent" }
  ],
  "warnings": ["capabilities.browser is auto and no provider detected; UI evidence falls back to link-only"]
}
```

Fingerprint: `sha256` over the canonical JSON of the parsed config (sorted keys, no comments or whitespace), a `;`, and the sorted `name=major` list of the protocol contracts the validating skill embeds. Comment-only edits do not invalidate; any value change does; a skill update that bumps a contract major does. `resolved.*` is a cache (copied to `resolved.json`) and is never written back to `config.yaml`.

### Tier 2: fast path

Every run parses the config, computes the fingerprint, and compares it with the receipt. Equal, with a supported `receipt_version` and `validated_at` within `validation_max_age`, means the skill proceeds to its read-only phase using cached resolutions with no network validation.

### Tier 3: volatile rechecks before mutation

Immediately before the first mutation of a run, and again after any mutation fails with a permission or not-found error, recheck only what that mutation depends on. Each check runs once per run and is reused unless a failure re-triggers it. Reads never trigger Tier 3.

| Mutation | Recheck |
| --- | --- |
| Any GitHub write | `gh auth status` succeeds for the receipt's host; active login equals `github.login` (mismatch stops); scopes include `repo`, plus `project` when a Project is configured |
| Label change, comment, claim | permission ≥ triage; the label names still exist |
| Project status mirror | Project by owner/number exists; status field and options re-resolved by name in one query; cache replaced |
| Branch push, PR open | permission ≥ push; `base_branch` exists; default branch unchanged since receipt |
| Dependency edge write | selected transport passes a read probe of the target issue's blocked-by list; else fall to the next transport and record the downgrade |
| Dogfood session start | readiness URL returns `expect_status`; browser capability available |
| Findings filing | `findings.approval.mode` present; `state/publications/` writable; dedupe sources reachable |
| Decomposition handoff (`provider: to-tickets`) | provider skill discoverable (below); upstream's tracker configuration names the same repository; `ready_for_agent`, `ready_for_human`, and `needs_decision` labels exist and the shared strings equal those in the repository's `docs/agents/triage-labels.md` (ADR 0003) |

**Sibling skill discovery.** A skill locates an installed sibling (its companion worker, or the decomposition provider) by checking, in order: the runtime's own skill listing when it exposes one; `<repo>/.agents/skills/<name>/SKILL.md` and `<repo>/.claude/skills/<name>/SKILL.md` (project scope, both install modes); the runtime's global skills directory (`~/.codex/skills/<name>` for Codex, the Claude configuration directory's `skills/<name>` for Claude Code); and, for Claude Code, an installed plugin that declares the skill. A found `SKILL.md` must parse with the expected `name`. Nothing is read from the sibling beyond its frontmatter; ADR 0001 forbids runtime dependence on its files.

### Invalidation

The receipt is invalid, and Tier 1 must run before any side effect, when: the fingerprint differs; `receipt_version` major is unsupported; `validated_at` exceeds `validation_max_age`; or a Tier 3 check fails on an identity fact (login, repository, default branch, Project). A Tier 3 failure on a transient fact (network, rate limit) retries once, then stops without invalidating.

### Stop-with-guidance

A stop happens before any mutation and prints one block in this shape, recorded in the run receipt as `outcome: setup-required`:

```text
Setup required: .agent-skills/config.yaml is missing 2 required keys for pick-up-work.

  work.verification      no commands and `none` is not set
                         -> add at least one command, or set work.verification.none: true
  work.pr.state          not set
                         -> choose `draft` or `ready`

Validation receipt: invalid (fingerprint changed since 2026-08-30T09:12:44Z).
No changes were made to GitHub or the working tree.

Fix the keys above and rerun, or run the skill with "setup" to be guided through them.
```

Rules: list every missing or invalid key; name the consuming skill; state that nothing was mutated; never propose a default for a mutation-affecting key inside the message (guided setup asks instead). Coordinators surface a worker's stop unchanged and claim nothing.

## Claim protocol (v1)

### Representation

The authoritative claim is an issue comment beginning with a marker line and containing one fenced YAML block; trailing prose is for humans and is not parsed.

````markdown
<!-- agent-skills:claim v1 -->
```yaml
claim: 1
issue: 142
owner: claude-code@buildbox/20260902T140512Z-3f9a1c2e
claimed_at: 2026-09-02T14:05:12Z
expires_at: 2026-09-02T18:05:12Z
state: active            # active | released | superseded
skill: pick-up-work
```
Claimed by an agent run. Only this owner releases it; expires 2026-09-02 18:05 UTC.
````

Owner id = `<runtime>@<host>/<run start, basic ISO-8601 UTC>-<first 8 hex of a v4 UUID>`, where `runtime` is `codex` or `claude-code` and `host` is the hostname sanitized to `[a-z0-9-]`. The timestamp orders claims for humans; the UUID fragment guarantees uniqueness. A coordinator forms its own owner id and gives each dispatched worker `<coordinator-owner>/w<n>`, so a stranded child claim is attributable to its coordinator.

### Operations

- **Claim.** Read comments; the *active claim* is the newest marker comment with `state: active` and `expires_at` absent or in the future. If none, post the claim comment, then (with `mirror_status` and a Project) set status `in_progress` and, if `labels.in_progress` is set, add that label. Claim before creating a worktree and before any post-selection confirmation, closing the source skill's double-pick window. If another owner holds the active claim, report `claim-held` and do nothing.
- **Idempotency.** If the active owner equals this run's owner id or its coordinator prefix, the claim is already held; never post a second comment. A crashed run that retries continues without duplicating.
- **Ownership verification.** Every mutation that assumes the claim (push, PR open, status change, release) re-reads the active claim and confirms the owner. Mismatch stops with `claim-lost`.
- **Heartbeat.** With `heartbeat` > 0 and a runtime that can edit comments, the worker extends `expires_at` by `ttl` every `heartbeat` interval. Otherwise heartbeat is skipped and the run receipt records `heartbeat: unsupported`; at claim time such a worker sets `expires_at` to `ttl` plus `work.pr.monitoring.max_duration` when it owns monitoring.
- **Release (owner only).** Edit the claim comment to `state: released` with `released_at` and `disposition` (`pr-opened | needs-info | failed | abandoned`); mirror status (`in_review` after a PR, `todo` otherwise) and remove the mirror label. If editing is unavailable, post `<!-- agent-skills:claim-release v1 -->` with the owner; a release marker from the same owner closes the claim. A coordinator may release claims whose owner carries its own id as prefix.
- **Abandonment.** A claim is abandoned when `expires_at` has passed, or when it has no `expires_at`, is older than `2 × ttl`, and no open PR matches `branch_pattern`. A Project item in `in_progress` with no active claim comment is a *stale mirror*, not a claim.
- **Recovery.** `reclaim`: edit the old comment to `state: superseded` (or post a release marker), then post a new claim with `supersedes: <old owner>` and `reason: expired`. `release`: reset the mirror and leave the issue unclaimed. `stop`: report only. Interactive runs with `confirm_interactive` ask per recovery; unattended runs apply the policy and record every recovery in the receipt. Stale mirrors are reset to `todo` under `reclaim` or `release`.

### Why a comment

Labels carry no owner or time and cannot distinguish two concurrent runs sharing one GitHub account. Assignee is an account, not a run, and the source human-gate worker already overloads it. Project status needs the `project` scope, is absent in repositories without a Project, and per ADR 0002 may reflect a claim but is not one. A comment carries structured data, is visible where humans already look, needs only `repo` scope, works through `gh issue comment`, and survives Project deletion. The alternative considered, assignee plus an `in-progress` label as authoritative with the owner recorded only locally, is simpler and more visible in issue lists but leaves the owner unrecoverable after a crash on another machine and cannot expire; it survives only as the optional `labels.in_progress` mirror.

## Protocol contract versioning

| Contract | Owner | v1 defined by |
| --- | --- | --- |
| `config`, `claim`, `receipt.validation` | this document | the schema, comment format, and receipt above |
| `receipt.work` | #9 | worker result line and run receipt |
| `receipt.human` | #10 | human-gate result |
| `findings`, `publication` | #18 | findings artifact and publication receipt |

- Each contract has an integer major. Additive optional fields do not bump it; renamed, removed, or re-typed fields do.
- Every published skill embeds a generated `references/protocol-contracts.md` listing the majors it produces and accepts; drift tests compare it with the canonical source (ADR 0002).
- A consumer meeting a *lower* supported major applies documented upgrade rules; a *higher* major stops with guidance naming the producer and the skill update required. Unknown ownership or outcome semantics are never guessed.
- `config.version` is bumped only by this document. A `version: 2` config with a v1-only skill stops; a v1 config with a skill that also accepts v2 proceeds and warns once.
- The fingerprint includes every contract major the validating skill embeds, so a skill upgrade re-validates automatically.
- A claim with an unknown major is treated as *held by unknown*: not claimed, released, or recovered.

## Setup flow

When `config.yaml` is absent or invalid, every skill runs the same embedded setup procedure before any side effect. There is no separate setup skill (ADR 0002 fixes seven jobs); the procedure text is generated into each skill's `references/setup.md`.

1. Discover: `origin`, default branch, visibility, `gh` auth and scopes, existing labels, visible Projects, and verification commands hinted by `package.json`, `Makefile`, `pyproject.toml`, or CI workflow files.
2. Ask only for the keys the invoking skill requires (matrix), plus `findings.approval.mode` for findings-producing skills. Mutation-affecting keys (`tracker.labels.*`, `tracker.statuses.*`, `work.pr.state`, `work.verification`, `work.concurrency.max_parallel`, `dogfood.target`, `findings.approval.mode`) are always asked; the prompt may suggest the canonical or discovered value but never writes it unasked.
3. Write `config.yaml` (optional sections as commented examples) and `state/.gitignore`, run Tier 1, write the receipt, and ask the user to commit both files.
4. Unattended runs never enter setup; missing config is a stop with guidance.

Minimal valid config for `pick-up-work` (repository discovered; no Project, so no status mirror; claim defaults apply):

```yaml
version: 1
tracker:
  labels:
    ready_for_agent: ready-for-agent
    needs_info: needs-info
work:
  eligibility:
    label_roles: [ready_for_agent]
    exclude_label_roles: [needs_info]
  verification:
    commands:
      - { name: test, run: npm test }
  pr:
    state: ready
```

## Source assumptions and their disposition

| Source assumption (inventory) | Disposition |
| --- | --- |
| Repository coordinates; opaque Project, field, option IDs | Removed; `repository.*` discovered, `tracker.*` by name, IDs cached in the local receipt |
| Label strings used as roles; MVP milestone default | Configured via `tracker.labels` role map and `tracker.milestones` |
| Status-as-lock claim after worktree creation and confirmation | Replaced by the comment claim taken before both; status is a mirror |
| Board planner script and GraphQL recipes | Removed; `tracker.dependencies.transport` prefers `gh`, then REST, GraphQL, body |
| npm commands, frontend path, auth bypass, screenshot conventions, private-repo embedding | Configured via `work.verification`, `evidence.ui`, `browser.auth` keyed to discovered visibility |
| Indefinite PR monitoring | Configured via `work.pr.monitoring`; lifecycle owner decided in #9 |
| Production URL, health check, journeys, stances, competitors | Configured via `dogfood.*` |
| Pipeline-area map, fixed ledger path, five-agent fan-out | Configured via `research.*`, capped by runtime capacity |
| Filing and approval overlay duplicated across two skills | Configured once via `findings.*`; approval has no default |
| Runtime tool names | Delegated to `capabilities.*` and `runtime_notes` |

## Acceptance scenarios

1. **Given** no `config.yaml`, **when** `pick-up-work` runs interactively, **then** it runs guided setup, writes the file, `state/.gitignore`, and a receipt, and touches neither GitHub nor the working tree before the receipt exists.
2. **Given** no `config.yaml`, **when** `work-the-board` runs unattended, **then** it stops with the setup block, claims nothing, and records `outcome: setup-required`.
3. **Given** a valid receipt, **when** only comments in `config.yaml` change, **then** the next run takes the fast path with no network validation.
4. **Given** a valid receipt, **when** `tracker.statuses.in_progress` is renamed to a non-existent option, **then** Tier 1 re-runs and stops listing that key with guidance.
5. **Given** a receipt validated as account A, **when** `gh` is authenticated as B, **then** Tier 3 fails before the first mutation and the stop names both logins.
6. **Given** issue 142 actively claimed by owner X, **when** owner Y attempts a claim, **then** no comment is posted, status is unchanged, and the result is `claim-held`.
7. **Given** issue 142 claimed by owner X, **when** the same run retries after a crash, **then** it recognizes its owner id and continues without a second claim.
8. **Given** a claim expired 20 minutes ago under `expired: reclaim`, **when** a coordinator runs unattended, **then** it supersedes the old claim, posts a new one with `supersedes`, resets and re-sets the mirror, and records the recovery.
9. **Given** a Project item in `In Progress` with no claim comment, **when** `work-the-board` computes the frontier, **then** it treats it as a stale mirror, resets it to `Todo`, and includes it as a candidate.
10. **Given** a runtime that cannot edit comments, **when** the worker releases, **then** it posts a `claim-release` marker and readers treat the claim as closed.
11. **Given** `findings.approval.mode` absent, **when** `dogfood` finishes a session, **then** the artifact is written and filing stops naming that key.
12. **Given** `transport: auto` and a `gh` without dependency flags, **when** an edge is written, **then** REST is used and the downgrade is recorded; the body form is used only if REST and GraphQL both fail.
13. **Given** a `version: 2` config, **when** a skill embedding only `config: 1` runs, **then** it stops naming the mismatch and the update needed.

## Risks

- Comment-based claims cost one API call per candidate during frontier computation. Mitigation: coordinators fetch comments only for issues passing label and status filters and cache claim state per run.
- Runtimes that cannot edit comments rely on `ttl` alone, so a long task can expire while alive. Mitigation: the extended initial `expires_at` above, and `ttl` is per repository.
- Visibility flips after validation can produce link-only evidence in a now-public repository; corrected on the next Tier 1.
- Committed publication receipts land only when their PR merges; two machines can file the same finding in that window. The open-issue dedupe scope closes most of it.

## Open items

- `claims.representation` supports only `comment`; an `assignee` or `project-field` value would need its own recovery semantics. No ticket owns it.
- `spec_gate` is a placeholder; #10 owns the gate vocabulary beyond `needs_decision`.
- The minimum `gh` release carrying native dependency and sub-issue flags could not be verified from this environment (ADR 0003 open item 3); once confirmed, Tier 1 adds a `gh.version` check and #9 pins the probe command.
- ADR 0004's hygiene scan must allowlist `example.com` and `<...>` placeholders so this document's examples pass; #15 confirms.

Maintainer ratifies: the `state/local/` gitignore split; 4h TTL and 30m heartbeat; `reclaim` as the default expired-claim policy; the owner id format; case-insensitive status matching.

## Consequences

- **#9 (work-execution)**: consumes `work.*`, `claims.*`, `evidence.*`, `tracker.dependencies`, `capabilities.*`; defines `receipt.work` v1 with outcomes including `setup-required`, `claim-held`, `claim-lost`, `monitoring-expired`; claims before worktree creation; applies the Tier 3 table before push and PR open; decides the monitoring lifecycle owner within `work.pr.monitoring`.
- **#10 (human-gate)**: consumes `tracker.project`, `tracker.labels.needs_decision` and `spec_gate`, `tracker.milestones`, `claims.*` (human sessions claim with the same comment format and `skill: pick-up-human-task`), and `capabilities.ticket_decomposition` for its decompose path; defines `receipt.human` v1; may extend `tracker.labels`.
- **#11 (dogfooding)** and **#12 (project research)**: consume `dogfood.*` or `research.*`, `browser.*`, `evidence.*`, `findings.*`; stop with guidance on a missing `findings.approval.mode` while preserving the artifact.
- **#18 (findings publication)**: consumes `findings.*`, `capabilities.ticket_decomposition`, the findings label roles plus `needs_decision`, `tracker.fields.theme`, `tracker.milestones`; owns the `findings` and `publication` contracts and the committed `state/publications/` receipt format.
- **ADR 0003 (#13)**: its `labels.roles` and `capabilities.ticket-decomposition` keys are absorbed here as `tracker.labels.*` and `capabilities.ticket_decomposition.*`; the semantics (provider values, `needs-decision` gate, string agreement with upstream, preflight order) stand unchanged.
- **#15 (validation strategy)**: tests fingerprint stability, the fast path, every Tier 3 recheck, all thirteen scenarios, the `state/.gitignore` behavior, and drift between each skill's generated `references/protocol-contracts.md` and `references/setup.md` and their canonical sources.

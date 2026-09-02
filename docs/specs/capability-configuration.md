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
- A claim is a structured issue comment carrying `issue`, `owner`, `claimed_at`, and optional `expires_at`. The comment is authoritative; Project status is a mirror. Only the owner releases; abandoned claims are recoverable after expiry under a configured policy. Interactive workers of both families claim after the human confirms the selection and before any other side effect; unattended paths claim immediately after selection.
- Contract versions are independent integer majors (`config`, `claim`, `receipt.work`, `receipt.human`, `receipt.dogfood`, `receipt.research`, `findings`, `publication`); consumers tolerate additive fields and stop on unknown majors. All receipt families share one envelope, and all findings producers and the publisher share one finding-fingerprint rule, both pinned here.
- `findings.approval.mode` has no default and must be written by the repository owner. `publish-findings` is the only filing path; `dogfood` and `research-project` hand off to it and never file directly.
- This document is the single key registry. Keys the later specs (#9, #10, #11, #12, #18) proposed are absorbed below and the cross-spec tensions they raised are decided in the reconciliation log near the end.

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
│           ├── runs/<owner-id>.json   # run receipts of every family; coordinator receipts carry status running|complete (stranded-claim ledger)
│           ├── human-board.json # work-the-human-board session state: skip set, cleared, in_flight (session_state v1, #10)
│           └── evidence/<artifact_id>/…   # dogfood walk-tier captures, pruned after evidence.ui.retention_days (#11)
├── <evidence.ui.dir>/           # committed finding-linked evidence (default docs/evidence)
├── <findings.artifact_dir>/     # committed findings artifacts (default docs/findings)
├── <research.areas_file>        # committed research map (default docs/research/areas.yaml; research_map v1, #12)
└── <research.ledger_path>       # committed research ledger (default docs/research/LEDGER.md; ledger v1, #12)
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
    needs_decision: needs-decision     # human-decision gates (ADR 0003); decision and specification gates are one queue (#10)
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
      owner: worker              # worker | coordinator | none; effective owner decided by #9 (worker only with pr_monitoring)
      max_duration: 2h           # afterwards the receipt reports monitoring-expired
      on_failed_checks: fix-once # fix-once | report
      max_retries: 3             # total pushes per PR across fix and conflict attempts; persisted in the run receipt (#9)

human:                           # required section for the human-gate pair; mirrors work in shape (#10)
  eligibility:                   # required by pick-up-human-task / work-the-human-board
    label_roles: [needs_decision]
    status_roles: [todo]         # ignored without project
    exclude_label_roles: [needs_info, wontfix, ready_for_human]   # ready_for_human is never a gate
    require_no_open_blockers: true
  priority:
    order: [leverage, direct, oldest]   # v1 accepts only this sequence; issue number is the final tie-break
    leverage_scope: repository   # repository | filters: which open issues count toward leverage
  candidates:
    accelerator: none            # none | status-update: today's Project status update read once as a hint, never as truth
  timezone: UTC                  # IANA zone; defines "today" and formats human-facing timestamps
  claim:
    ttl: 1h                      # overrides claims.ttl for this family (a silent gate is abandoned sooner)
    mirror_status: false         # overrides claims.mirror_status: a gate is never shown In Progress by default
  session:
    cap: 5                       # gates cleared per sitting
    tap_out_phrase: "tap out"    # matched case-insensitively in any human reply to the coordinator
    skip_set_ttl: 24h            # age after which state/local/human-board.json is archived instead of restored
  disclosure:
    mode: append                 # append | off: AI-assistance line on the decision comment
    text: ""                     # empty = catalog default sentence
  domain_docs:
    glossary: ""                 # discovered when empty (docs/agents/domain.md, CONTEXT.md)
    adr_dir: ""                  # discovered when empty (docs/adr/)
  split:
    method: auto                 # auto | ticket-decomposition | sub-issues | manual

evidence:
  ui:
    type: screenshot             # screenshot | video | gif | none
    dir: docs/evidence
    embed: auto                  # auto (inline when public, link-only when private) | inline | link-only
    retention_days: 14           # walk-tier captures under state/local/evidence/ are pruned after this (#11)

browser:
  mode: attach                   # attach | launch | none
  auth:                          # one enumeration for every skill that drives a browser; dogfood.auth overrides per skill
    mode: attach-to-user-session # none | attach-to-user-session | provided-test-account | manual-login-handoff
    # In every mode the agent never sees, types, stores, or prints a raw credential; #11 defines each
    # mode's requirements and fallback. `boundary: user | none` (config v1 alias) reads as
    # attach-to-user-session | none and warns once; it is not written by setup.
    account_ref: ""              # provided-test-account: environment variable or runtime secret name, never a value
    bypass: none                 # none | header:<name>; treated as provided-test-account whose reference supplies the header

dogfood:                         # required section for dogfood
  target:
    url: https://staging.example.com          # required; asked during setup
    environment: staging         # staging | production; production is asked during setup
    allow_production: false      # must be true when environment is production, else preflight-failed
    readiness:                   # url + expect_status, or a command exiting 0; Tier 3 "Dogfood session start"
      url: https://staging.example.com/health
      expect_status: 200
      command: ""
    revision_source: none        # none | header | endpoint: how the deployed revision is proven
    revision_header: ""          # with header: response header carrying the revision
    revision_url: ""             # with endpoint: URL returning the revision
    revision_json_path: ""       # with endpoint: JSON path into the body; empty = whole body
  auth:                          # per-skill override of browser.auth; absent = inherit browser.auth
    mode: attach-to-user-session
    account_ref: ""
  journeys:                      # at least one; { id, name, entry, steps, expected, tags, routes, changed_paths }
    - id: onboarding
      name: Sign up and reach the welcome page
      entry: /signup
      steps: ["submit the sign-up form", "open the welcome page"]
      expected: The welcome page greets the new account by name
      tags: [core]               # auth | mutating | mobile | core | free text
      routes: ["/signup", "/welcome"]
      changed_paths: ["apps/web/src/onboarding/**"]   # recent-change -> journey mapping
  personas:                      # at least one; { id, stance, notes }
    - { id: first-time-user, stance: first-time-user, notes: "" }
  stances: []                    # custom [{ id, summary, notices, ignores }]; adds to the shipped library, never overrides a shipped id
  competitors: []                # [{ name, url, compare: [<what to compare>], journey: <id> }]
  recent_changes:
    since: last-artifact         # last-artifact | window
    lookback: 14d                # cap, or the window when since is window
  budget:
    max_journeys: 4
    max_stances_per_journey: 2
    max_minutes: 60
  data_policy: observe-only      # observe-only | scratch-account
  severity_rubric: {}            # { <level>: <one line> }; required when findings.severity_scale is customized

research:                        # required section for research-project
  areas_file: docs/research/areas.yaml   # research map (research_map v1, #12); exactly one of areas_file or areas
  # areas:                       # minimal inline form; maps to the research map as #12 specifies
  #   - id: ingestion            #   (id -> id; summary -> name and questions: [summary]; code -> landing_points; docs -> current_state.docs)
  #     summary: How raw inputs enter the system
  #     code: ["packages/ingest/**"]
  #     docs: ["docs/adr/0003-ingestion.md"]
  ledger_path: docs/research/LEDGER.md
  lens_count: 3                  # effective = min(this, budget.max_branches, runtime subagent capacity)
  lenses:
    add: []                      # [{ id, stance, sources, question, avoid }]
    disable: []                  # shipped lens ids to disable
  scoring:
    weights: { evidence: 0.4, applicability: 0.4, feasibility: 0.2 }   # must sum to 1
    bar: 3.5                     # weighted score a candidate must reach
    evidence_floor: 2            # minimum evidence axis regardless of score
  avoid_list_expiry: 180d        # ledger verdicts older than this no longer exclude a source
  budget:
    max_branches: 5
    max_sources_per_branch: 8
    max_minutes: 30              # wall clock for the branch phase
  persist: branch-pr             # branch-pr | commit | working-tree; asked during setup
  branch_pattern: "research/{area}-{date}"

findings:
  artifact_dir: docs/findings
  severity_scale: [critical, high, medium, low]   # an artifact may override per its frontmatter (#18)
  approval:
    mode: interactive            # REQUIRED, no default: interactive | pre-approved-severities | never
    # interactive: a human approves each Proposed Action in-session.
    # pre-approved-severities: file without prompting only severities listed below.
    # never: produce artifacts only; filing is refused.
    auto_file_severities: []
    auto_file_kinds: [simple]    # pre-approved-severities: action kinds eligible for unprompted filing
  dedupe:
    scope: [receipts, open_issues, closed_issues]
    closed_window_days: 90
  filing:
    create_missing_labels: false # true: create absent label strings with a neutral description and record it
    add_to_project: true         # effective only with tracker.project
    umbrella: off                # off | on: one umbrella issue per artifact with native sub-issues (YAML 1.1 parsers read these as false | true; both forms are accepted)
  persistence:                   # dogfood's docs commit; research-project uses research.persist instead
    commit: commit-on-branch     # commit-on-branch | leave-uncommitted; asked during setup
    branch_pattern: "findings/{artifact_id}"
    push: true                   # effective only with commit-on-branch

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
  grilling: auto                 # optional provider for the human-gate worker's questioning; baseline in-skill (#10)
  triage: auto                   # optional provider proposing the human-gate exit transition; baseline in-skill (#10)
  validation_max_age: 30d        # receipt age after which Tier 1 re-runs
  runtime_notes:
    codex: "Parallel subagents via the runtime task facility; cap 2."
    claude_code: "Browser control via the installed browser MCP server; attach to the running profile."
```

### Cross-key validation rules (Tier 1)

Beyond per-key types and enumerations, Tier 1 fails on: both `research.areas_file` and `research.areas` set, or neither, for `research-project`; `research.scoring.weights` not summing to 1; a customized `findings.severity_scale` without a `dogfood.severity_rubric` entry per level (dogfood only); `dogfood.target.environment: production` without `allow_production: true`; a `dogfood.stances[].id` or `research.lenses.add[].id` equal to a shipped id (custom entries add, never override; disable a lens with `research.lenses.disable`); `human.priority.order` other than `[leverage, direct, oldest]`; `browser.auth.boundary` and `browser.auth.mode` both set; a path key escaping the repository. `browser.auth.boundary` alone warns once and is read as `mode`. Unknown keys, including a leftover `tracker.labels.spec_gate`, warn and are ignored.

## Required keys by consuming skill

R = required, O = optional, D = discovered when absent, - = unused. Coordinators inherit their worker's requirements because they preflight it (ADR 0001).

| Key | pick-up-work | work-the-board | pick-up-human-task | work-the-human-board | dogfood | research-project | publish-findings |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `version` | R | R | R | R | R | R | R |
| `repository.*` | D | D | D | D | D | D | D |
| `tracker.project`, `fields.status`, `statuses.todo/in_progress/done` | O | R | R | R | - | - | O |
| `tracker.labels.ready_for_agent`, `needs_info` | R | R | O | O | - | - | O; `ready_for_agent` R when an action is `agent-doable` |
| `tracker.labels.needs_decision` | - | - | R | R | - | - | R when `provider` is not `none` or an action is `human-decision` |
| `tracker.labels.ready_for_human` | - | - | O (default exclusion) | O | - | - | R when an action is `human-execution` |
| `tracker.labels.bug`, `improvement` | - | - | - | - | - | - | R |
| `tracker.labels.wontfix`, `finding`, `in_progress`; `tracker.fields.priority`, `theme` | O | O | O | O | - | - | O |
| `capabilities.ticket_decomposition` | - | - | O | O | - | - | O |
| `tracker.milestones`, `dependencies`; `claims.*` | O | O | O | O | - | - | O |
| `work.eligibility`, `priority`, `verification`, `pr.state` | R | R | - | - | - | - | - |
| `work.concurrency.max_parallel` | - | R | - | - | - | - | - |
| other `work.*` | O | O | - | - | - | O (`base_branch`) | - |
| `human.eligibility` | - | - | R | R | - | - | - |
| other `human.*` | - | - | O | O | - | - | - |
| `evidence.ui` | O | O | - | - | R (`dir`) | - | O (`embed`) |
| `browser.*` | O | - | - | - | O (`auth` when `dogfood.auth` is absent) | - | - |
| `dogfood.target.url`, `journeys`, `personas` | - | - | - | - | R | - | - |
| `dogfood.target.allow_production` | - | - | - | - | R when `environment` is `production` | - | - |
| other `dogfood.*` | - | - | - | - | O | - | - |
| `research.areas_file` or `research.areas` (exactly one) | - | - | - | - | - | R | - |
| other `research.*` | - | - | - | - | - | O | - |
| `findings.approval.mode` | - | - | - | - | R | R | R |
| `findings.persistence` | - | - | - | - | O | - | - |
| `findings.artifact_dir`, `severity_scale`, `dedupe`, `filing`, `approval.auto_file_*`; `capabilities.*` | O | O | O | O | O | O | O |

Label roles the publisher applies are checked at filing time by its preflight ("every label string the run will apply exists"), so the conditional entries above are enforced per run, not at Tier 1.

`dogfood` and `research-project` require `findings.approval.mode` even without `publish-findings` installed so that the artifact they produce is publishable the moment the publisher is installed and setup asks the question once; they never file directly. Without the publisher they write the artifact, print the install command, and stop (#18 producer handoff contract, accepted by #11 and #12).

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
    { "id": "dogfood.target", "status": "skipped", "detail": "section absent" },
    { "id": "research.map", "status": "skipped", "detail": "section absent" }
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
| Dogfood session start | `environment: production` requires `allow_production: true` (else `preflight-failed`); readiness URL returns `expect_status`, or `readiness.command` exits 0, within 30 seconds, retried once (else `target-unhealthy`); browser capability available; the effective auth mode is servable or its documented fallback is recorded (#11) |
| Findings filing | `findings.approval.mode` present; `state/publications/` and `findings.artifact_dir` writable; dedupe sources reachable (`gh issue list` probe); every label string the run will apply exists, or `findings.filing.create_missing_labels: true` creates it and records the creation; Project and status field re-resolved when `filing.add_to_project` and a Project are configured; milestone exists when policy is `named` |
| Producer persistence (`findings.persistence.commit: commit-on-branch`; `research.persist: branch-pr` or `commit`) | as "Branch push, PR open"; `gh` available for `branch-pr`, else downgrade to `commit`, recorded (#12); a dirty checkout downgrades `branch-pr` to `working-tree`, recorded |
| Decomposition handoff (`provider: to-tickets`; `publish-findings` complex actions and the `pick-up-human-task` split path) | provider skill discoverable (below); upstream's tracker configuration names the same repository; `ready_for_agent`, `ready_for_human`, and `needs_decision` labels exist and the shared strings equal those in the repository's `docs/agents/triage-labels.md` (ADR 0003) |

**Sibling skill discovery.** A skill locates an installed sibling (its companion worker, or the decomposition provider) by checking, in order: the runtime's own skill listing when it exposes one; `<repo>/.agents/skills/<name>/SKILL.md` and `<repo>/.claude/skills/<name>/SKILL.md` (project scope, both install modes); the runtime's global skills directory (`~/.codex/skills/<name>` for Codex, the Claude configuration directory's `skills/<name>` for Claude Code); and, for Claude Code, an installed plugin that declares the skill. A found `SKILL.md` must parse with the expected `name`. Companion coordinators additionally read the worker's `metadata.contracts`, a space-separated `name=major` list the generator writes (for example `"config=1 claim=1 receipt.work=1"`), and stop before any claim when a major falls outside their accepted range (#9, #10). Nothing is read from the sibling beyond its frontmatter; ADR 0001 forbids runtime dependence on its files.

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

- **Claim.** Read comments; the *active claim* is the newest marker comment with `state: active` and `expires_at` absent or in the future. If none, post the claim comment, then (with the family's effective `mirror_status` and a Project) set status `in_progress` and, if `labels.in_progress` is set, add that label. If another owner holds the active claim, report `claim-held` (receipt outcome `skipped-claimed` in both families) and do nothing.
- **Timing.** Unattended paths claim immediately after selection: a coordinator claims each item before dispatching its worker, and a standalone worker running unattended claims as soon as it has selected. Interactive workers of both families claim after the human confirms the selection and before any other side effect (worktree, first question to the human, board status). The claim step re-reads the active claim, so a collision during the confirmation pause yields `claim-held` and a fresh selection rather than a posted-then-released comment. In every path the claim precedes the worktree, which closes the source skill's double-pick window.
- **Family overrides.** `human.claim.ttl` and `human.claim.mirror_status` replace `claims.ttl` and `claims.mirror_status` for the human-gate family; `claims.heartbeat` and `claims.recovery` apply to every family unchanged.
- **Idempotency.** If the active owner equals this run's owner id or its coordinator prefix, the claim is already held; never post a second comment. A crashed run that retries continues without duplicating.
- **Ownership verification.** Every mutation that assumes the claim (push, PR open, status change, release) re-reads the active claim and confirms the owner. Mismatch stops with `claim-lost`.
- **Heartbeat.** With `heartbeat` > 0 and a runtime that can edit comments, the worker extends `expires_at` by `ttl` every `heartbeat` interval. Otherwise heartbeat is skipped and the run receipt records `heartbeat: unsupported`; at claim time such a worker sets `expires_at` to `ttl` plus `work.pr.monitoring.max_duration` when it owns monitoring.
- **Release (owner only).** Edit the claim comment to `state: released` with `released_at` and `disposition`. The disposition is the releasing run's receipt outcome, or `abandoned` for a recovery: work family `pr-opened | needs-info | blocked | failed | merged | closed`; human family `cleared-for-agent | cleared-for-human | needs-info | split | deferred | closed | tapped-out | failed`; `abandoned` (with `reason`, for example `expired` or `session-interrupted`) in either. Dispositions are informational and additive; `state` is what closes the claim, so a reader never stops on an unknown disposition. Mirror status (`done` after `merged`, `in_review` after a PR, `todo` otherwise) and remove the mirror label. If editing is unavailable, post `<!-- agent-skills:claim-release v1 -->` with the owner; a release marker from the same owner closes the claim. A coordinator may release claims whose owner carries its own id as prefix.
- **Abandonment.** A claim is abandoned when `expires_at` has passed, or when it has no `expires_at`, is older than `2 × ttl`, and no open PR matches `branch_pattern`. A Project item in `in_progress` with no active claim comment is a *stale mirror*, not a claim.
- **Stranded fast path.** At startup a coordinator reads `state/local/runs/` for its own host's coordinator receipts left `status: running`; child claims carrying those owner ids as prefix that are still active with no open PR matching `branch_pattern` are abandoned immediately, without waiting for expiry, because the local ledger proves the owning process ended (#9). `work-the-human-board` applies the same rule through the `in_flight` entry of `state/local/human-board.json` and, being interactive, always asks whether to resume or release (#10). Recoveries taken this way are recorded in the receipt like any other.
- **Recovery.** `reclaim`: edit the old comment to `state: superseded` (or post a release marker), then post a new claim with `supersedes: <old owner>` and `reason: expired`. `release`: reset the mirror and leave the issue unclaimed. `stop`: report only. Interactive runs with `confirm_interactive` ask per recovery; unattended runs apply the policy and record every recovery in the receipt. Stale mirrors are reset to `todo` under `reclaim` or `release`.

### Why a comment

Labels carry no owner or time and cannot distinguish two concurrent runs sharing one GitHub account. Assignee is an account, not a run, and the source human-gate worker already overloads it. Project status needs the `project` scope, is absent in repositories without a Project, and per ADR 0002 may reflect a claim but is not one. A comment carries structured data, is visible where humans already look, needs only `repo` scope, works through `gh issue comment`, and survives Project deletion. The alternative considered, assignee plus an `in-progress` label as authoritative with the owner recorded only locally, is simpler and more visible in issue lists but leaves the owner unrecoverable after a crash on another machine and cannot expire; it survives only as the optional `labels.in_progress` mirror.

## Protocol contract versioning

| Contract | Owner | v1 defined by |
| --- | --- | --- |
| `config`, `claim`, `receipt.validation` | this document | the schema, comment format, and receipt above |
| `receipt.work` | #9 | worker result block and run receipt; outcomes `pr-opened`, `needs-info`, `blocked`, `failed`, `none-ready`, `skipped-claimed`, `claim-lost`, `setup-required`, `merged`, `closed`, `checks-fixed`, `retries-exhausted`, `monitoring-expired` |
| `receipt.human` | #10 | human-gate result; outcomes `cleared-for-agent`, `cleared-for-human`, `needs-info`, `split`, `deferred`, `closed`, `tapped-out`, `none-ready`, `failed`, `skipped-claimed`, `setup-required` |
| `receipt.dogfood` | #11 | dogfood run receipt; outcomes `completed`, `partial`, `target-unhealthy`, `setup-required`, `preflight-failed`, `aborted` |
| `receipt.research` | #12 | research run receipt; outcomes `researched`, `researched-empty`, `setup-required`, `failed` |
| `findings`, `publication` | #18 | findings artifact and publication receipt |
| `session_state` (`state/local/human-board.json`) | #10 | local coordinator session file; never read by another skill, so not a fingerprint input |
| `research_map`, `ledger` (`research.areas_file`, `research.ledger_path`) | #12 | repository-owned file formats versioned by their own headers; not fingerprint inputs |

**Common receipt envelope.** Every `receipt.*` family shares one envelope: required `receipt` (major), `family`, `skill`, `owner`, `outcome`, `protocols`; `issue` and `claim` (`held | released | none | lost`) whenever the family mutates a tracker item; `skill_version`, `reason`, `link`, `downgrades`, `recoveries`, `started_at`, `finished_at` optional and additive. The block is emitted as `<!-- agent-skills:receipt <family> v1 -->` followed by one fenced YAML block at the end of the run's output, and written identically as JSON to `state/local/runs/<owner-id>.json` (`/` in the owner id becomes `_`). `receipt.dogfood` and `receipt.research` stay separate families under this envelope; no merged `receipt.producer` family is registered, because their outcome vocabularies and required fields differ and a generic reader needs only the envelope. A consumer stops on an unknown family, major, or outcome.

**Shared finding fingerprint.** Producers (`dogfood`, `research-project`) and the publisher compute the same value from one rule, quoted from #18: "**Finding fingerprint** = first 16 hex of `sha256(normalize(title) + "\n" + subject + "\n" + disposition)`; `normalize` lowercases, strips a leading `bug:`/`fix:`-style prefix and `#<n>` references, removes punctuation, collapses whitespace. Stable across artifacts, so the same defect found on two days matches." Two readings the producer specs asked #18 to confirm are pinned here so that all three implementations agree: `subject` is the finding's *effective* subject (its own `- subject:` line when present, else the artifact's frontmatter `subject`), and "removes punctuation" deletes punctuation characters without substituting whitespace before whitespace is collapsed. This text is generated into `references/findings-artifact.md` of both producers and the publisher from one canonical source and drift-tested (ADR 0002); the validator script #18 defers to implementation is its reference implementation.

- Each contract has an integer major. Additive optional fields do not bump it; renamed, removed, or re-typed fields do.
- Every published skill embeds a generated `references/protocol-contracts.md` listing the majors it produces and accepts; drift tests compare it with the canonical source (ADR 0002).
- A consumer meeting a *lower* supported major applies documented upgrade rules; a *higher* major stops with guidance naming the producer and the skill update required. Unknown ownership or outcome semantics are never guessed.
- `config.version` is bumped only by this document. A `version: 2` config with a v1-only skill stops; a v1 config with a skill that also accepts v2 proceeds and warns once.
- The fingerprint includes every contract major the validating skill embeds, so a skill upgrade re-validates automatically.
- A claim with an unknown major is treated as *held by unknown*: not claimed, released, or recovered.

## Setup flow

When `config.yaml` is absent or invalid, every skill runs the same embedded setup procedure before any side effect. There is no separate setup skill (ADR 0002 fixes seven jobs); the procedure text is generated into each skill's `references/setup.md`.

1. Discover: `origin`, default branch, visibility, `gh` auth and scopes, existing labels, visible Projects, and verification commands hinted by `package.json`, `Makefile`, `pyproject.toml`, or CI workflow files.
2. Ask only for the keys the invoking skill requires (matrix), plus `findings.approval.mode` for findings-producing skills. Mutation-affecting keys (`tracker.labels.*`, `tracker.statuses.*`, `tracker.milestones.policy` for both coordinators, `work.pr.state`, `work.verification`, `work.concurrency.max_parallel`, `dogfood.target.url`, `dogfood.target.environment` and `allow_production` when production, `findings.approval.mode`) are always asked; the prompt may suggest the canonical or discovered value but never writes it unasked. Persistence keys (`findings.persistence.commit` for `dogfood`, `research.persist` for `research-project`) keep their catalog defaults because a branch push is reversible and invisible to the default branch, but guided setup asks them anyway since they push. When `tracker.labels.needs_decision` is asked and the label does not exist, setup offers `gh label create` and then the one-time migration of open `ready_for_human` issues that #10 specifies; both are interactive only, logged in the setup receipt, and never run unattended.
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
| Status-as-lock claim after worktree creation and confirmation | Replaced by the comment claim taken after confirmation and before the worktree; status is a mirror |
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
14. **Given** an interactive `pick-up-work` or `pick-up-human-task` selection and another owner claiming during the confirmation pause, **when** the human confirms, **then** the claim step yields `claim-held`, no worktree or question exists, no release comment was posted, and selection restarts.
15. **Given** `dogfood.target.environment: production` with `allow_production` absent, **when** `dogfood` runs, **then** Tier 1 (or the Tier 3 dogfood recheck) stops with `preflight-failed` naming the key before any browser session.
16. **Given** both `research.areas_file` and `research.areas` set, **when** `research-project` runs, **then** Tier 1 fails naming both keys and nothing is mutated.
17. **Given** `browser.auth.boundary: user` in a config, **when** any skill validates, **then** it is read as `browser.auth.mode: attach-to-user-session` with one deprecation warning and the run proceeds.

## Risks

- Comment-based claims cost one API call per candidate during frontier computation. Mitigation: coordinators fetch comments only for issues passing label and status filters and cache claim state per run.
- Runtimes that cannot edit comments rely on `ttl` alone, so a long task can expire while alive. Mitigation: the extended initial `expires_at` above, and `ttl` is per repository.
- Visibility flips after validation can produce link-only evidence in a now-public repository; corrected on the next Tier 1.
- Committed publication receipts land only when their PR merges; two machines can file the same finding in that window. The open-issue dedupe scope closes most of it.

## Open items

- `claims.representation` supports only `comment`; an `assignee` or `project-field` value would need its own recovery semantics. No ticket owns it.
- The minimum `gh` release carrying native dependency and sub-issue flags could not be verified from this environment (ADR 0003 open item 3); once confirmed, Tier 1 adds a `gh.version` check and #9 pins the probe command.
- ADR 0004's hygiene scan must allowlist `example.com`, `acme/widgets`, and `<...>` placeholders so this document's examples pass; #15 confirms.
- Two persistence key families exist (`findings.persistence.*` for `dogfood`; `research.persist` and `research.branch_pattern` for `research-project`) with deliberately different vocabularies, because only the research skill opens a docs PR. Folding both into one `findings.persistence.mode` enumeration is deferred to the naming pass; until then a repository may set both.
- `receipt.dogfood` is described by #11 as an `AGENT_SKILLS_RECEIPT` one-line JSON form; the common envelope above pins the marker block used by every other family. #11 aligns its emission at implementation; maintainer ratifies.

Maintainer ratifies: the `state/local/` gitignore split; 4h TTL and 30m heartbeat; `reclaim` as the default expired-claim policy; the owner id format; case-insensitive status matching; and the decisions in the reconciliation log below (in particular the `browser.auth.mode` enumeration with the `boundary` alias, `dogfood.auth` inheriting from `browser.auth` when absent, persistence keys being asked in setup while keeping defaults, and interactive claiming after confirmation for both families).

## Reconciliation log (2026-09-02)

The five later specs were written against the first version of this document and each flagged additions or tensions for #8. This pass absorbs them; every decision below is applied in the body above.

| Item | Source spec | Decision | Rationale |
| --- | --- | --- | --- |
| Producers filing directly when the publisher is absent | findings-publication (open item 1), dogfooding, project-research | Producers never file. The sentence under the matrix now says `findings.approval.mode` is required so the artifact is publishable and setup asks once; without the publisher they write the artifact, print the install command, and stop. | Both producers chose this; a second filing path would duplicate dedupe and receipt logic and the Decision bullet already names one filing path. |
| Interactive claim timing | work-execution (claim after confirmation, before worktree), human-gate (Decision text says before confirmation; its worker flow claims at step 4 after the step 3 confirmation) | Both families: claim after the human confirms the selection and before any other side effect (worktree, first question, board status). Unattended paths claim immediately after selection; coordinators claim before dispatch. Recorded under "Timing" in the claim protocol; the Decision bullet and the source-disposition table are reworded. | A claim before confirmation posts and releases a comment every time the human declines; a claim after confirmation re-reads the active claim, so a race yields `claim-held` and a fresh selection. The human-gate worker flow already claims after confirmation, so only its prose changes. |
| `spec_gate` placeholder role | human-gate | Retired: removed from the schema, the matrix, the consequences, and the open items. Decision and specification gates are one `needs_decision` queue. A leftover key warns as unknown and is ignored. | The human-gate pair selects on one role; a second gate label would split the frontier and the leverage calculation for no consumer. |
| Split outcome name: `split` (human-gate) vs `decomposed` (ADR 0003) | human-gate, ADR 0003 | `split` is the `receipt.human` outcome and release disposition; the capability that produced it is recorded in the receipt (`capabilities.ticket_decomposition: to-tickets`, `sub-issues`, or `manual`). ADR 0003's `decomposed` is read as `split` with that capability. | One outcome regardless of provider keeps the consumer's outcome table closed; the provider is a fact about the run, not a different result. |
| `dogfood.auth.mode` superseding `browser.auth.boundary` | dogfooding | One `browser.auth` enumeration for every browser-driving skill: `mode: none \| attach-to-user-session \| provided-test-account \| manual-login-handoff` plus `account_ref`; `boundary: user \| none` stays a config v1 alias (read as `attach-to-user-session \| none`, warns once). `dogfood.auth.{mode, account_ref}` overrides per skill and inherits `browser.auth` when absent. `pick-up-work`'s UI-evidence step uses `browser.auth.mode` with the same value set. `bypass: header:<name>` is `provided-test-account` whose reference supplies the header. | Two enumerations for the same boundary would drift; the alias keeps the previously documented config valid and `version` at 1. Inheritance is what "override" means; #11 listed `none` as its standalone default, and the maintainer ratifies the inherited default instead. |
| Finding-fingerprint normalization | findings-publication (definition), dogfooding (effective subject), project-research (punctuation removal) | The #18 definition is quoted verbatim in the protocol-contract section, with two pinned readings: effective subject (per-finding override, else artifact subject) and punctuation deleted without substitution. Generated into both producers and the publisher from one canonical source; drift-tested. | Three implementations must produce one value or dedupe silently fails; the producer specs' examples already computed fingerprints under these readings. |
| `receipt.dogfood` and `receipt.research` vs a merged `receipt.producer` | dogfooding (open item 3), project-research | Separate families sharing the common envelope, which is now defined here (required fields, marker block, JSON run file). | Outcome vocabularies and required fields differ; a generic reader needs only the envelope. |
| Receipt emission form | dogfooding (`AGENT_SKILLS_RECEIPT` one-line JSON) vs work-execution, human-gate, project-research (marker block) | The marker block is canonical for every family; #11 aligns at implementation. Listed under open items for ratification. | One parser for #15 and for any future reconciler. |
| `claim-held` vs `skipped-claimed` | work-execution, human-gate | `claim-held` is the claim-protocol result; both families report it as receipt outcome `skipped-claimed`. Recorded in the claim operation and the #9 consequence. | The protocol names a condition; the receipt names a run outcome. Both specs already use `skipped-claimed`. |
| Persistence keys | dogfooding (`findings.persistence.{commit, branch_pattern, push}`), project-research (`research.persist`, `research.branch_pattern`; asks `persist` in setup) | Both registered as specified. Both are asked in guided setup while keeping their catalog defaults (a third class beside "no default" and "optional"), recorded in the setup flow. Unification deferred (open item). | Each producer depends on its own key and vocabulary; a branch push is reversible, so a default is acceptable, but the user should still see it. |
| Publisher's disposition labels vs the matrix | findings-publication | `ready_for_agent`, `needs_decision`, `ready_for_human` are required for `publish-findings` whenever an action carries the matching disposition (not only when the provider is set); enforced by the publisher's per-run label preflight. Matrix updated. | The matrix previously tied `needs_decision` to the decomposition provider, but simple `human-decision` actions also receive it. |
| `dogfood.stances` overriding shipped ids | dogfooding (open item 4) | Add only; a custom id equal to a shipped id is a Tier 1 error. The same rule applies to `research.lenses.add`, with `research.lenses.disable` as the way to drop a default. | Overriding a shipped stance silently changes the meaning of every prior artifact that names it. |
| Inline `research.areas` vs `research.areas_file` | project-research | Exactly one of the two; declaring both, or neither, is a Tier 1 error; the inline form maps as #12 specifies (comment in the schema). | Confirms #12's rule so the skill and the validator agree. |
| Tier 3 dogfood row | dogfooding | Extended with `allow_production`, `readiness.command`, the 30-second retry, and the auth-mode fallback. | Requested by #11; keeps the readiness rule in one place. |
| Extended release `disposition` enumeration | work-execution (`merged`, `closed`), human-gate (its outcomes) | Disposition = the releasing run's receipt outcome, or `abandoned`; both families' lists recorded; dispositions are informational so unknown values never stop a reader. | Keeps the claim protocol at v1 while letting each family add outcomes. |
| Stranded-claim fast path | work-execution | Added to the abandonment rules, extended to the human board's `in_flight` entry. | The local run ledger proves the owning process ended; waiting for TTL only delays recovery. |
| Coordinator setup asks `tracker.milestones.policy` | work-execution | Added to the always-ask list for both coordinators. | Fixes the source bug where a prose default never reached the planner. |

**Keys added in this pass.** `work.pr.monitoring.max_retries`; the whole `human.*` section (`eligibility`, `priority.{order, leverage_scope}`, `candidates.accelerator`, `timezone`, `claim.{ttl, mirror_status}`, `session.{cap, tap_out_phrase, skip_set_ttl}`, `disclosure.{mode, text}`, `domain_docs.{glossary, adr_dir}`, `split.method`); `evidence.ui.retention_days`; `browser.auth.{mode, account_ref}` (with `boundary` as alias); `dogfood.target.{environment, allow_production, readiness.command, revision_header, revision_url, revision_json_path}`, `dogfood.auth.{mode, account_ref}`, `dogfood.journeys[].{name, entry, steps, expected, tags}`, `dogfood.personas[].notes`, `dogfood.stances`, `dogfood.competitors[].{compare, journey}`, `dogfood.recent_changes.{since, lookback}`, `dogfood.budget.{max_journeys, max_stances_per_journey, max_minutes}`, `dogfood.severity_rubric`; `research.areas_file`, `research.lenses.{add, disable}`, `research.scoring.{weights, bar, evidence_floor}`, `research.avoid_list_expiry`, `research.budget.{max_branches, max_sources_per_branch, max_minutes}`, `research.persist`, `research.branch_pattern`; `findings.severity_scale`, `findings.approval.auto_file_kinds`, `findings.filing.{create_missing_labels, add_to_project, umbrella}`, `findings.persistence.{commit, branch_pattern, push}`; `capabilities.grilling`, `capabilities.triage`. Removed: `tracker.labels.spec_gate`.

**Changelog.** `config` stays at major 1: every change is additive, the removed `spec_gate` key was a placeholder no skill consumed, and `browser.auth.boundary` remains readable as an alias, so the previously documented minimal config and every excerpt in the later specs validate unchanged. Contract table gains `receipt.dogfood` 1 and `receipt.research` 1; the claim protocol stays at 1 (dispositions are informational).

## Consequences

- **#9 (work-execution)**: consumes `work.*` (including `pr.monitoring.max_retries`), `claims.*`, `evidence.*`, `tracker.dependencies`, `capabilities.*`; defines `receipt.work` v1 with the outcomes listed in the contract table (the protocol's `claim-held` reported as `skipped-claimed`); interactive claims after confirmation and before the worktree, coordinator claims before dispatch; applies the Tier 3 table before push and PR open; the stranded fast path and the `merged`/`closed` dispositions are absorbed here.
- **#10 (human-gate)**: consumes `tracker.project`, `tracker.labels.needs_decision`, `tracker.milestones`, `claims.*` with the `human.claim` overrides (human sessions claim with the same comment format and `skill: pick-up-human-task`, after confirmation), the `human.*` section, `capabilities.{grilling, triage, ticket_decomposition}`, and `state/local/human-board.json`; defines `receipt.human` v1 with `split` as the decomposition outcome; `spec_gate` is retired.
- **#11 (dogfooding)** and **#12 (project research)**: consume `dogfood.*` or `research.*`, `browser.*` (through `dogfood.auth` for dogfood), `evidence.*`, `findings.*` (`findings.persistence` for dogfood, `research.persist` for research); define `receipt.dogfood` and `receipt.research` v1 under the common envelope; stop with guidance on a missing `findings.approval.mode` while preserving the artifact; never file directly; embed the shared fingerprint rule from the canonical source.
- **#18 (findings publication)**: consumes `findings.*` (now including `severity_scale`, `approval.auto_file_kinds`, `filing.*`), `capabilities.ticket_decomposition`, the findings label roles plus the three disposition roles, `tracker.fields.theme`, `tracker.milestones`; owns the `findings` and `publication` contracts, the committed `state/publications/` receipt format, and the canonical source of the shared fingerprint rule.
- **ADR 0003 (#13)**: its `labels.roles` and `capabilities.ticket-decomposition` keys are absorbed here as `tracker.labels.*` and `capabilities.ticket_decomposition.*`; the semantics (provider values, `needs-decision` gate, string agreement with upstream, preflight order) stand unchanged; its `decomposed` result is `receipt.human` outcome `split` with the capability recorded.
- **#15 (validation strategy)**: tests fingerprint stability, the fast path, every Tier 3 recheck, all seventeen scenarios, the cross-key validation rules, the `boundary` alias, the `state/.gitignore` behavior, one fingerprint fixture shared by both producers and the publisher, and drift between each skill's generated `references/protocol-contracts.md`, `references/setup.md`, and `references/findings-artifact.md` and their canonical sources.

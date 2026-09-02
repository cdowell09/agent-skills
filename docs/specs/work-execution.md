# Work-execution wave: `pick-up-work` and `work-the-board`

Date: 2026-09-02
Question: What should the public single-item and batch work-execution skills preserve, remove, configure, and delegate from the source project's `pick-up-work` and `work-the-board`, and what acceptance scenarios define their public contracts?
Resolves: [#9 Specify the work-execution release wave](https://github.com/cdowell09/agent-skills/issues/9)
Inputs: [ADR 0001](../adr/0001-npx-skills-distribution-contract.md), [ADR 0002](../adr/0002-skill-boundaries-and-shared-contracts.md), [ADR 0003](../adr/0003-ticket-decomposition-provider.md), [ADR 0004](../adr/0004-licensing-attribution-and-contribution.md), [Capability configuration contract](capability-configuration.md), [Source skill inventory](../research/2026-07-12-source-skill-inventory.md), [Source provenance audit](../research/2026-07-12-source-provenance.md)

Skill names used here (`pick-up-work`, `work-the-board`, `pick-up-human-task`, `work-the-human-board`, `dogfood`, `research-project`, `publish-findings`) are working names; a later naming pass may rename them. The human-gate pair is specified concurrently in `human-gate.md`; the two documents keep their frontier, claim, and receipt contracts parallel in shape.

## Decision

- `pick-up-work` owns one issue end to end, from claim verification through an isolated worktree, test-first implementation, configured verification, optional UI evidence, and self-review to a **verified open pull request**, its synchronous completion boundary. `work-the-board` never implements: it preflights the worker, computes the frontier, previews a wave, claims on the worker's behalf, dispatches, reconciles receipts, re-evaluates, and recovers.
- One generated frontier contract serves both: eligibility from `work.eligibility`, `tracker.milestones`, native blocked-by edges, and the claim protocol; priority from `work.priority.order` with issue number as the final tie-break. No filter exists outside `config.yaml`; the effective filters are printed before selection and recorded in every receipt.
- Claims use the capability spec's comment protocol unchanged. The coordinator claims as `<coordinator-owner>/w<n>` before dispatch; the worker verifies ownership before its first mutation; a standalone interactive worker claims after confirmation and before creating a worktree.
- Post-PR lifecycle: the worker owns it only when `work.pr.monitoring.owner: worker` **and** the `pr_monitoring` capability is available; otherwise the next run of either skill reconciles PR state. The claim is held through merge or closure; retries are bounded (3 pushes per PR, 1 fix per failing-check episode, 2h window).
- `receipt.work` v1 is a marker block in the worker's output plus a run-receipt file; consumers ignore additive fields and stop on an unknown major, outcome, or owner.
- Effective concurrency = min(`work.concurrency.max_parallel`, runtime subagent capacity, `--max`); without isolated subagents the coordinator runs workers one at a time and says so.

## Purpose and boundary

| | `pick-up-work` | `work-the-board` |
| --- | --- | --- |
| Owns | Standalone selection, claim verification, implementation, verification, evidence, PR, receipt, monitoring when effective owner | Worker preflight, frontier, wave preview, claim handoff, dispatch, receipt reconciliation, queue re-evaluation, stranded-claim and PR-state recovery, final report |
| Never | Dispatches other workers; merges PRs; rewrites others' branches; enters setup unattended | Implements, pushes, or opens PRs; claims before worker preflight passes; re-dispatches an issue that failed in the same run |
| Done when | PR is open, targets `work.base_branch`, references the issue, receipt emitted | Frontier empty or only merge-boundary-blocked issues remain; report written |

## Source disposition

### `pick-up-work`

| Preserve | Remove | Configure | Delegate |
| --- | --- | --- | --- |
| Select one ready issue; read the whole thread; fidelity check before coding; confirm unless unattended; isolated worktree; test-first for behavioral change; verify before PR; UI evidence when UI changed; simplify and self-review; PR linked to issue; machine-readable result; PR as completion boundary | Repository and Project identity; opaque IDs; MVP milestone prose default; hard-coded label strings; local board-planning script; npm-workspace commands; frontend path; auth bypass; Playwright device and image conventions; private-repo image behavior; FFmpeg install; runtime-specific review commands; indefinite monitoring; the policy breadth that made one skill a style guide (each cross-cutting rule becomes a capability baseline or a config key) | `work.eligibility`, `work.priority`, `tracker.milestones`, `work.base_branch`, `work.branch_pattern`, `work.worktrees`, `work.verification`, `work.tests`, `work.review`, `work.pr`, `evidence.ui`, `browser`, `claims` | Worktree creation (required capability; git-worktree baseline in-skill); TDD, verification, simplify, issue-fidelity and diff review (optional providers such as Superpowers; baselines in `references/implementation-baselines.md`); browser evidence (optional; fallback none, disclosed); PR monitoring (optional; fallback coordinator reconcile) |

### `work-the-board`

| Preserve | Remove | Configure | Delegate |
| --- | --- | --- | --- |
| Unblocked frontier; wave preview; claim before dispatch; `--max` bounded fan-out of unattended workers; collect results; release failures; re-evaluate; report merge-boundary blocks instead of draining across them | Fixed Project fields; status-as-lock; local board planner; "multiple Agent calls" runtime language; children-inherit-runtime assumption; post-PR same-branch fix rules (moved to the lifecycle section); issue-number-only priority (now the last tie-break) | `work.concurrency.max_parallel`; the shared eligibility, priority, milestone keys; `claims.recovery`, `claims.mirror_status`; Project statuses | Parallel isolated subagents (optional; sequential invocation fallback, disclosed); worker discovery (capability spec's sibling-discovery rule); PR-state reconciliation (shared lifecycle rules) |

## Shared frontier contract (v1)

Generated into both skills as `references/frontier-rules.md` and drift-tested (ADR 0002).

**Eligible** iff all hold: (1) an open issue, not a PR; (2) carries every label mapped by `work.eligibility.label_roles` (default `[ready_for_agent]`, the only role ADR 0003 leaves agent-eligible); (3) carries none of `exclude_label_roles`; (4) with `tracker.project`, is an item whose status is in `status_roles`, a stale mirror (`in_progress` with no active claim) being reset per the claim protocol and treated as `todo`; (5) has no active claim, except one owned by this run's owner id or coordinator prefix, which counts as eligible-and-held; (6) with `require_no_open_blockers`, every native blocked-by issue is closed (an open PR for a blocker does not clear it; that is the merge boundary); (7) milestone per `tracker.milestones.policy`: `none` no filter, `named` equals `active`, `any` some milestone set.

**Priority.** Apply `work.priority.order` left to right (capability spec default `[priority_field, milestone, oldest]`), then issue number ascending unconditionally. `priority_field`: option order of the single-select field mapped by `tracker.fields.priority`, first option highest, unset last, skipped without a Project or field. `milestone`: earliest due date first, no due date next, no milestone last. `oldest`: lowest issue number first.

**Computation with `gh`.** (1) `gh issue list --state open --label <ready_for_agent> [--milestone <active>] --json number,title,labels,milestone,url --limit 200`; exclusions and `any` filtered client-side. (2) With a Project, `gh project item-list <number> --owner <owner> --format json`, joined on issue number, kept when status is in `status_roles`. (3) Blocked-by per `tracker.dependencies.transport` order: `gh issue view N --json` dependency fields when the installed `gh` exposes them, else `gh api repos/{owner}/{repo}/issues/N/dependencies/blocked_by`, else the GraphQL `blockedBy` connection; the `Blocked by: #N` body convention only when the transport resolves to `body`, recorded as a downgrade. (4) Claim comments fetched only for survivors of 1–3. (5) Sort. Reads are cached per run.

**Filter rule (fixes the source bug).** The source project stated an MVP default in prose that never reached its planner, which filtered by milestone only when a value was passed and otherwise selected every open issue. Here the only milestone filter is `tracker.milestones`; an absent key means `none` because the capability spec declares that default, and both skills print the effective filters (labels, exclusions, statuses, Project, milestone) in the preview or selection summary and record them under `frontier.filters` in the receipt. Guided setup for `work-the-board` asks `tracker.milestones.policy` explicitly (addition to the capability spec's always-ask list; #8 absorbs).

## Claim behavior for the pair

Comment format, owner ids, operations, and expiry recovery are the capability spec's claim protocol v1; this section fixes who claims when.

- **Coordinator claims for the worker.** After worker preflight and wave preview, it posts each wave item's claim with `owner: <coordinator-owner>/w<n>`, `skill: pick-up-work`, `expires_at = claimed_at + claims.ttl` (plus `work.pr.monitoring.max_duration` when the worker will own monitoring and cannot heartbeat), mirrors status, then dispatches `pick-up-work --issue N --owner <id> --unattended`. `claim-held` on an item skips it and pulls the next frontier item.
- **Worker verifies on start.** With `--owner`, it re-reads the active claim before any mutation; a mismatch ends the run with `claim-lost` and no side effects. Push, PR open, status change, and release each re-verify per the protocol.
- **Standalone worker.** Unattended: claim immediately after selection. Interactive: select, print the summary, confirm, **then** claim, **then** create the worktree. This adjusts the capability spec's "before any post-selection confirmation": claiming first posts and releases a comment whenever the human declines, whereas the claim step after confirmation re-reads the active claim, so a collision during the pause yields `claim-held` and a fresh selection, never duplicate work. (The human-gate family claims before confirmation because there the conversation is the work.) Maintainer ratifies; #8 aligns its wording.
- **Release by outcome.** `pr-opened`: claim stays active, mirror `in_review`. `needs-info`, `blocked`, `failed`: worker releases with that disposition, mirror `todo`. `none-ready`, `skipped-claimed`, `claim-lost`, `setup-required`: nothing to release. A worker that exits without releasing is released by the coordinator under the prefix rule with disposition `failed`.
- **Recovery.** Expired claims follow `claims.recovery.expired`. Stranded fast path: at startup a coordinator reads `state/local/runs/` for its own host's coordinator receipts left `status: running`; their child claims that are still active with no open PR matching `branch_pattern` are abandoned immediately, because the local ledger proves the owning process ended (addition to the protocol's abandonment rule; #8 absorbs).

## Worker flow (`pick-up-work`)

Arguments: `--issue N`, `--owner <id>`, `--unattended`, `--dry-run`, `--fix-pr N` (lifecycle fix mode).

1. **Preflight.** Parse config, run the validation tiers, resolve capabilities, list downgrades. Missing required keys stop with the setup block (`setup-required`); interactive runs may enter guided setup.
2. **Select or accept.** With `--issue`, verify eligibility (held-by-this-owner counts); otherwise take the frontier head; an empty frontier ends with `none-ready` plus the effective filters.
3. **Claim** per the rules above.
4. **Read the thread:** body, every comment, sub-issues, blocked-by set, linked PRs, repository conventions (`AGENTS.md`/`CLAUDE.md`, `docs/agents/`, contributing docs).
5. **Issue-fidelity check.** Derive acceptance criteria and touched surfaces; isolated subreview when `work.review.issue_fidelity: required` and subagents exist, else the in-context checklist. Thin spec: interactive asks the human and may record answers as a comment; unattended posts one comment listing the questions, adds the `needs_info` label role (`ready_for_agent` stays; exclusion removes the issue from the frontier), releases, ends `needs-info`. An unmet external prerequisite ends `blocked` with a comment.
6. **Worktree.** Baseline `git fetch` then `git worktree add <work.worktrees.root>/<branch> -b <branch> <base_branch>`; `required` stops on failure, `preferred` falls back to the checkout with a disclosed downgrade, `off` uses the checkout. Branch from `work.branch_pattern`.
7. **Implement and verify.** `work.tests.policy` governs test-first (`behavioral`: failing test first for behavior changes; `always`; `never`). Run every `work.verification.commands` entry, once per `matrix` entry when set; `none: true` skips with disclosure. A failing command is fixed or ends `failed`.
8. **UI evidence.** When the diff touches a UI surface and `evidence.ui.type` is not `none`: with browser capability, capture into `evidence.ui.dir` and embed per `embed` (`auto` uses discovered visibility); without it, record `evidence: none` with the reason in the PR body and receipt.
9. **Simplify and self-review.** Diff review capability when present, else the baseline checklist (dead code, scope creep, missing tests, secrets, unrelated files); `self_review: required` blocks the PR on unresolved findings.
10. **PR.** Tier 3 rechecks for push and PR open; push; `gh pr create --base <base_branch>`, `--draft` when `work.pr.state: draft`; body from `work.pr.template` or the discovered `.github` template with `Closes #N`, verification summary, evidence or its absence, downgrades.
11. **Verify completion.** `gh pr view --json state,isDraft,baseRefName,closingIssuesReferences,url` must show open, correct base, and the issue among closing references; otherwise repair the body or end `failed`. Mirror `in_review`.
12. **Monitoring disposition** per the lifecycle below; emit the receipt block and run receipt.

## Post-PR lifecycle

**Effective owner.** `work.pr.monitoring.owner: worker` makes the worker the owner only when the `pr_monitoring` capability is available (a runtime facility that wakes the run on PR events, such as Claude Code's PR subscription); otherwise the effective owner is `coordinator`, recorded in the receipt with the reason. `coordinator` means the next `work-the-board` or standalone `pick-up-work` run in the repository reconciles every active claim with an open PR at startup; the worker never waits. `none` releases the claim with `pr-opened` at PR time and leaves the rest to humans. The receipt names the effective owner and, for `worker`, the `until` timestamp.

| PR event | Claim | Status / labels | Action |
| --- | --- | --- | --- |
| Merged | Release, `merged` | `done` | Confirm the issue closed (close as completed if `Closes` did not fire); remove the local worktree; never delete the remote branch |
| Closed unmerged or rejected | Release, `closed` | `todo` + `needs_info` label | One comment: PR link, visible closure reason, what a human must decide before re-dispatch; the exclusion rule keeps the issue out of the frontier until a human removes the label |
| Failed checks | Held | unchanged | `on_failed_checks: fix-once`: one fix attempt per failing-check episode on the same branch and worktree; `report`: comment naming the failing checks, no code change |
| Merge conflict | Held | unchanged | Update from `base_branch` by merge (`gh pr update-branch` or `git merge`); rebase only when every commit is this owner's; a branch with another author's commits is never rewritten: comment, release `failed` |
| Retries exhausted | Release, `failed` | `in_review`, PR stays open | Comment on PR and issue summarizing attempts; no automatic re-dispatch |
| `max_duration` elapsed, PR open | Held | unchanged | Worker ends `monitoring-expired`; the coordinator path takes over next run |

**Defaults.** `max_duration: 2h`; `on_failed_checks: fix-once`; `work.pr.monitoring.max_retries: 3` total pushes per PR across fix and conflict attempts (addition; #8 absorbs), persisted in the run receipt so a reconciling coordinator continues the same budget.

**Same-branch rules (relocated from the source coordinator).** Follow-up work on an open PR happens on that PR's branch and worktree, by the effective owner, never as a new issue, branch, or claim. A fix run is `pick-up-work --fix-pr N --owner <id>`: reuse or recreate the worktree from the remote branch, address the failure, re-run verification, push, and emit `checks-fixed` or `failed`. A coordinator that is the effective owner dispatches fix runs inside a wave under the same cap.

## Receipt `receipt.work` v1

Emitted as a marker block at the end of the worker's output and, identically, as JSON in `state/local/runs/<owner-id>.json` (`/` in the owner id becomes `_`). Required: `receipt`, `family`, `skill`, `owner`, `issue`, `outcome`, `claim`, `protocols`; all else optional and additive.

````markdown
<!-- agent-skills:receipt work v1 -->
```yaml
receipt: 1
family: work
skill: pick-up-work
skill_version: 0.1.0
owner: claude-code@buildbox/20260902T140512Z-3f9a1c2e/w2
issue: 142
outcome: pr-opened
claim: held                      # held | released | none | lost
pr: { number: 187, url: https://github.com/acme/widgets/pull/187, state: open, draft: false, base: main }
branch: issue-142-export-csv
worktree: .worktrees/issue-142-export-csv
verification: [{ name: lint, status: pass }, { name: test, status: pass }]
evidence: { type: none, reason: browser capability unavailable }
monitoring: { owner: coordinator, reason: pr_monitoring unavailable, retries_used: 0 }
frontier: { filters: { labels: [ready-for-agent], exclude: [needs-info, wontfix], statuses: [Todo], project: acme/7, milestone: none } }
downgrades: ["evidence: none"]
recoveries: []
reason: null
started_at: 2026-09-02T14:05:40Z
finished_at: 2026-09-02T14:52:03Z
protocols: { config: 1, claim: 1, receipt.work: 1 }
```
````

**Outcomes.** Worker runs: `pr-opened`, `needs-info`, `blocked`, `failed`, `none-ready`, `skipped-claimed` (the protocol's `claim-held` result), `claim-lost`, `setup-required`. Lifecycle runs and reconciliation: `merged`, `closed`, `checks-fixed`, `retries-exhausted`, `monitoring-expired`. `reason` is required for `needs-info`, `blocked`, `failed`, `claim-lost`, `retries-exhausted`; `pr` for `pr-opened` and every lifecycle outcome. The claim protocol's release `disposition` gains `merged` and `closed` as additive values (#8 absorbs).

**Compatibility.** Consumers accept the `receipt.work` majors listed in their `references/protocol-contracts.md` (v1 at first release), ignore unknown fields, and stop with guidance on a higher major, an outcome or `claim` value outside the tables, or an `owner` that is not the dispatched id. A missing required field, or a worker exit with no marker block, is reconciled as `failed` with reason `malformed-receipt` or `no-receipt` and the claim released by prefix. Renaming, removing, or re-typing a field bumps the major.

## Coordinator flow (`work-the-board`)

Arguments: `--max N` (lowers the cap only), `--dry-run`, `--issue N` (repeatable; restricts the frontier).

1. **Preflight.** Config and validation tiers for the coordinator's required keys. Discover the worker by the capability spec's sibling-discovery rule; its frontmatter must carry `name: pick-up-work` and `metadata.contracts`, a space-separated `name=major` list the generator writes (`"config=1 claim=1 receipt.work=1"`) whose majors fall inside the coordinator's accepted ranges; absent or incompatible values stop before any claim, printing the paired update command. The convention exists because ADR 0001 forbids reading anything but a sibling's frontmatter.
2. **Capacity.** Effective concurrency = min(`work.concurrency.max_parallel`, runtime subagent capacity from `capabilities.subagents` and `runtime_notes`, `--max`). No isolated-subagent capability sets capacity 1 with "sequential fallback" printed. `work.worktrees.policy: off` stops the coordinator: parallel workers must be isolated.
3. **Reconcile and recover.** Apply the lifecycle table to every active claim with an open PR whose effective owner is `coordinator`; run stranded and expired recovery per `claims.recovery`; reset stale mirrors.
4. **Frontier.** Compute per the shared contract; set aside the merge-boundary set (eligible except for blockers whose only remaining work is an open PR).
5. **Wave preview.** Print effective filters, ordered frontier, the planned wave, the merge-boundary set, downgrades. Interactive runs confirm; `--dry-run` stops here with a receipt.
6. **Claim, then dispatch.** Claim each wave item as `<owner>/w<n>`; run each worker in an isolated subagent with its own worktree, or sequentially under the fallback.
7. **Collect** all receipts; validate per the compatibility rules.
8. **Reconcile.** `pr-opened`: keep the claim, record the PR. `needs-info`, `blocked`, `failed`, `claim-lost`, no receipt: ensure release; exclude the issue for the rest of the run. `setup-required`: surface unchanged and stop.
9. **Re-evaluate** and loop until: empty frontier; only merge-boundary or excluded issues remain; every remaining item is held by another owner; or an interactive stop.
10. **Report and receipt.** Per-issue outcomes with PR links, releases, recoveries, downgrades, and the merge-boundary list ("blocked on PRs #187, #190; rerun after merge"); write the coordinator receipt (`skill: work-the-board`, `children: [...]`, `status: complete`) to `state/local/runs/`.

## Configuration consumed

Every key is the capability spec's except the marked additions. Coordinator-relevant excerpt:

```yaml
version: 1
tracker:
  project: { owner: acme, number: 7 }        # required by work-the-board
  fields: { status: Status, priority: Priority }
  statuses: { todo: Todo, in_progress: In Progress, in_review: In Review, done: Done }
  labels: { ready_for_agent: ready-for-agent, needs_info: needs-info, wontfix: wontfix }
  milestones: { policy: none }                # explicit; coordinator setup asks (addition)
  dependencies: { transport: auto }
claims: { ttl: 4h, heartbeat: 30m, recovery: { expired: reclaim, confirm_interactive: true }, mirror_status: true }
work:
  eligibility: { label_roles: [ready_for_agent], status_roles: [todo], exclude_label_roles: [needs_info, wontfix], require_no_open_blockers: true }
  priority: { order: [priority_field, milestone, oldest] }
  concurrency: { max_parallel: 3 }
  worktrees: { policy: required, root: .worktrees }
  base_branch: main
  branch_pattern: "issue-{number}-{slug}"
  verification: { commands: [{ name: lint, run: npm run lint }, { name: test, run: npm test }] }
  tests: { policy: behavioral }
  review: { self_review: required, issue_fidelity: required }
  pr:
    state: ready
    monitoring: { owner: worker, max_duration: 2h, on_failed_checks: fix-once, max_retries: 3 }  # max_retries: addition
evidence: { ui: { type: screenshot, dir: docs/evidence, embed: auto } }
browser: { mode: attach, auth: { boundary: user } }
capabilities: { subagents: auto, worktrees: auto, browser: auto, diff_review: auto, pr_monitoring: auto }
```

Additions for #8 to absorb: `work.pr.monitoring.max_retries` (default 3); release dispositions `merged` and `closed`; the stranded-claim fast path; asking `tracker.milestones.policy` in coordinator setup. Adjacent, not config: the `metadata.contracts` frontmatter convention shared with `human-gate.md`.

## Capabilities

| Capability | Worker | Coordinator | Fallback |
| --- | --- | --- | --- |
| GitHub issue, PR, Project operations via `gh` | required | required | none; preflight stops |
| Worktrees | required (git-worktree baseline in-skill) | required (`policy: off` stops) | `preferred` uses the checkout, disclosed |
| Isolated subagents | optional (fidelity subreview) | optional (parallel dispatch) | in-context review; sequential invocation, disclosed |
| Sequential skill invocation | - | required when subagents are absent | none |
| Browser control | optional (UI evidence) | - | `evidence: none`, disclosed in PR and receipt |
| Diff review | optional | - | in-skill checklist |
| PR monitoring | optional (lifecycle ownership) | - | coordinator reconciliation next run |
| TDD, verification, simplify providers (for example Superpowers) | optional providers | - | in-skill baselines; ADR 0004 disclosure table, "Bundled here: No" |

Runtime notes only where the runtimes differ: Claude Code can run background subagents with their own worktrees and offers PR-event subscription, satisfying `subagents` and `pr_monitoring`; Codex runs parallel work through its task facility under the cap in `runtime_notes.codex` and has no PR subscription, so its effective monitoring owner is always `coordinator`. Browser attachment mechanics live in `runtime_notes`.

## `SKILL.md` outlines

### `pick-up-work`

```yaml
---
name: pick-up-work
description: Take one ready-for-agent GitHub issue from claim to a verified open pull request in an isolated worktree, with configured verification, evidence, and a versioned receipt.
license: MIT
compatibility: GitHub repository with Issues; gh CLI; git worktrees; .agent-skills/config.yaml. Optional: isolated subagents, browser control, diff review, PR monitoring; Superpowers worktree/TDD/verification skills as providers.
metadata: { author: Christian Dowell, version: "0.1.0", provenance: original, contracts: "config=1 claim=1 receipt.work=1" }
---
```

Sections: Purpose and boundary (one issue; PR is the boundary) · Arguments · Preflight (config tiers, capabilities, downgrades) · Select or accept · Claim (timing; `skipped-claimed`, `claim-lost`) · Read the thread · Issue-fidelity check (questions or `needs-info`) · Worktree (baseline command, policy) · Implement and verify (tests policy, commands, matrix) · UI evidence (capture or disclosed none) · Simplify and self-review · Open and verify the PR (state, template, `Closes`, Tier 3) · Monitoring disposition · Fix mode (`--fix-pr`) · Receipt · Safety rules (never rewrite others' branches; never mutate before claim verification; never enter setup unattended).

`references/`: `protocol-contracts.md`, `claim-protocol.md`, `receipt-work.md`, `frontier-rules.md`, `post-pr-lifecycle.md`, `setup.md` (all generated); `implementation-baselines.md` (TDD, verification, simplify, review checklists); `ui-evidence.md` (capture and embedding by visibility).

### `work-the-board`

```yaml
---
name: work-the-board
description: Drain the unblocked ready-for-agent frontier in bounded parallel waves by claiming issues and dispatching pick-up-work workers, then reconciling their receipts and pull-request state.
license: MIT
compatibility: GitHub repository with Issues and a Project; gh CLI; git worktrees; pick-up-work installed (companion); .agent-skills/config.yaml. Optional: isolated subagents (else sequential invocation).
metadata: { author: Christian Dowell, version: "0.1.0", provenance: original, contracts: "config=1 claim=1 receipt.work=1" }
---
```

Sections: Purpose and boundary (coordinate, never implement) · Arguments · Preflight (config, worker discovery, `metadata.contracts`) · Capacity · Reconcile and recover · Frontier and merge boundaries · Wave preview · Claim then dispatch (owner suffixes; subagent or sequential) · Collect and validate receipts · Reconcile outcomes · Re-evaluate and stop · Report and receipt · Safety rules (claim only after preflight; release only by prefix; never re-dispatch a failed issue in-run).

`references/`: `protocol-contracts.md`, `claim-protocol.md`, `receipt-work.md`, `frontier-rules.md`, `post-pr-lifecycle.md`, `setup.md` (all generated); `dispatch-adapters.md` (subagent versus sequential, per-runtime notes); `report-format.md`.

## Acceptance scenarios

1. **No config.** Given no `config.yaml`, when `work-the-board` runs unattended, then it prints the setup block, claims nothing, and records `setup-required`.
2. **Stale validation receipt.** Given a receipt older than `validation_max_age`, when `pick-up-work` starts, then Tier 1 re-runs before selection and no claim is posted until it passes.
3. **Claim collision.** Given #142 claimed by another owner, when the coordinator reaches it, then it records `skipped-claimed`, posts nothing, and dispatches the next frontier item.
4. **Worker version mismatch.** Given a worker whose `metadata.contracts` says `receipt.work=2`, when the coordinator preflights, then it stops before any claim naming the worker, the accepted range, and the paired update command.
5. **PR merged between runs.** Given a held claim whose PR merged since the last run, when either skill starts, then reconciliation releases with `merged`, confirms the issue closed, sets `done`, and removes the worktree.
6. **Retry exhaustion.** Given `fix-once` and `max_retries: 3`, when checks still fail after the budget, then the owner comments on PR and issue, releases with `failed`, leaves `in_review`, and the coordinator does not re-dispatch.
7. **Browser absent.** Given `evidence.ui.type: screenshot` and no browser capability, when the worker changes a UI surface, then PR body and receipt record `evidence: none` with the reason and the run ends `pr-opened`.
8. **Concurrency cap.** Given `max_parallel: 3`, capacity 2, and `--max 5`, when a wave is planned, then exactly two workers run and the preview states the effective cap and why.
9. **Filter unset.** Given `tracker.milestones` absent, when the frontier is computed, then the preview shows `milestone: none` as a declared default and no issue is excluded by milestone.
10. **Unattended thin spec.** Given an unattended worker and an issue without acceptance criteria, when the fidelity check runs, then it posts one questions comment, adds `needs-info`, releases with `needs-info`, and creates no worktree or branch.
11. **Interactive double-pick.** Given another agent claims the issue during the confirmation pause, when the human confirms, then the claim step yields `skipped-claimed`, no worktree exists, and selection restarts.
12. **Stranded claims.** Given a coordinator crashed mid-wave on this host, when the next run starts, then its child claims without open PRs are recovered via the local run ledger and listed under `recoveries`.
13. **Merge boundary.** Given #150 blocked only by #142 whose PR is open, when the frontier is computed, then #150 is reported as merge-boundary, never dispatched, and the run ends with "rerun after merge".
14. **No receipt.** Given a worker that exits without a marker block, when the coordinator collects, then it reconciles `failed` with reason `no-receipt` and releases the claim by prefix.
15. **Closed unmerged.** Given a reviewer closes the PR, when reconciliation runs, then the claim is released with `closed`, status returns to `todo`, `needs-info` is applied, and one comment explains.
16. **Sequential fallback.** Given no isolated-subagent capability, when the coordinator runs, then workers run one at a time and the receipt lists the downgrade.
17. **Foreign commits on a conflicted branch.** Given another author's commit on the branch, when the owner handles the conflict, then no history is rewritten, a comment explains, and the claim is released with `failed`.

## Risks

- Comment-based claims plus dependency reads cost several API calls per candidate. Mitigation: label and status filters first, claim and dependency reads only for survivors, per-run cache, `--limit 200`.
- Dependency fields in `gh issue view --json` were not verifiable from this session (ADR 0003 open item 3); the probe order degrades to the documented REST endpoint.
- Coordinator-owned reconciliation is only as timely as the next run; a merged PR can hold a claim for hours. Acceptable: claims expire and `Closes` closes the issue regardless.
- Same-branch fix runs can loop on flaky checks; the per-PR budget and no-re-dispatch rule bound it.
- `metadata.contracts` is a catalog convention, not an Agent Skills field; validators that strip unknown metadata would break preflight. The installer copies frontmatter verbatim, so exposure is limited.

## Open items

1. Maintainer ratifies: interactive claim after confirmation (differs from the capability spec's wording and the human-gate family); `max_retries: 3`; `needs_info` on closed-unmerged PRs; sequential fallback rather than a stop when subagents are absent.
2. An `unblocks` priority strategy (issues blocking the most open work first) for coordinators; deferred to the pilot.
3. Whether standalone `pick-up-work` reconciles all open-PR claims or only its host's; this spec says all, with `confirm_interactive` prompting.
4. Addressing reviewer comments in `--fix-pr` mode is out of scope for the first release; `report` is the safe policy for repositories with active human review.

## Consequences

- **#10 (human-gate wave):** shares `frontier-rules.md` in shape with `needs_decision` in place of `ready_for_agent`, the `metadata.contracts` convention, the receipt marker envelope and scalar `claim` field, and the `skipped-claimed` outcome; `ready-for-human` stays out of both frontiers.
- **#15 (validation and pilot strategy):** tests the seventeen scenarios, the `metadata.contracts` preflight, receipt parsing including malformed blocks, drift between each skill's generated references and the canonical sources, the sequential fallback, and the filter echo; adds the #8 additions to the schema tests once absorbed.
- **#16 (pilot consumer and acceptance scenarios):** the pilot repository needs a Project with status and priority fields, native blocked-by edges, CI checks that can be made to fail, both runtimes, and one UI surface, so scenarios 5–8 and 13–17 run for real.
- **#17 (release sequencing and readiness gates):** ships the pair together with paired install and update commands; documents `metadata.contracts`, the effective monitoring owner per runtime, and the merge-boundary rerun message; lists Superpowers providers as optional in the External skills table.

# Work-execution wave: `pick-up-work` and `work-the-board`

Date: 2026-09-02
Question: What should the public single-item and batch work-execution skills preserve, remove, configure, and delegate from the source project's `pick-up-work` and `work-the-board`, and what acceptance scenarios define their public contracts?
Resolves: [#9 Specify the work-execution release wave](https://github.com/cdowell09/agent-skills/issues/9)
Inputs: [ADR 0001](../adr/0001-npx-skills-distribution-contract.md), [ADR 0002](../adr/0002-skill-boundaries-and-shared-contracts.md), [ADR 0003](../adr/0003-ticket-decomposition-provider.md), [ADR 0004](../adr/0004-licensing-attribution-and-contribution.md), [Capability configuration contract](capability-configuration.md), [Source skill inventory](../research/2026-07-12-source-skill-inventory.md), [Source provenance audit](../research/2026-07-12-source-provenance.md)

Skill names used here (`pick-up-work`, `work-the-board`, `pick-up-human-task`, `work-the-human-board`, `dogfood`, `research-project`, `publish-findings`) are working names; a later naming pass may rename them.

## Decision

- `pick-up-work` owns one issue end to end: claim verification, full-thread read, fidelity check, isolated worktree, test-first implementation, configured verification, optional UI evidence, self-review, and a **verified open pull request**, which is its synchronous completion boundary. `work-the-board` never implements; it preflights the worker, computes the frontier, previews a wave, claims on the worker's behalf, dispatches, reconciles receipts, re-evaluates, and recovers.
- Both skills share one generated frontier contract: eligible = open issue, carries every `work.eligibility.label_roles` label, none of `exclude_label_roles`, status in `status_roles` when a Project is configured, no active claim, every native blocked-by issue closed, milestone per `tracker.milestones.policy`. Priority follows `work.priority.order` with issue number ascending as the final tie-break. There is no prose default anywhere; the effective filter set is printed before selection and recorded in every receipt.
- Claims use the capability spec's comment protocol unchanged. The coordinator claims as `<coordinator-owner>/w<n>` before dispatch; the worker verifies ownership before its first mutation. A standalone interactive worker claims after confirmation and before creating a worktree.
- Post-PR lifecycle ownership: the worker owns it only when `work.pr.monitoring.owner: worker` **and** the `pr_monitoring` capability is available; otherwise ownership falls to reconciliation at the start of the next run of either skill. The claim is held through merge or closure by whichever owner is effective; retries are bounded (default 3 pushes per PR, 1 fix attempt per failing-check episode, 2h window).
- `receipt.work` v1 is a single JSON result line plus a run-receipt file. Coordinators ignore additive fields and stop on an unknown major, outcome, or owner.
- Effective concurrency = min(`work.concurrency.max_parallel`, runtime subagent capacity, `--max`). Without isolated subagents the coordinator runs workers one at a time and says so.

## Purpose and boundary

| | `pick-up-work` (worker) | `work-the-board` (coordinator) |
| --- | --- | --- |
| Purpose | Take one `ready-for-agent` issue to a verified open PR, interactively or unattended | Drain the unblocked agent frontier in bounded parallel waves |
| Owns | Selection (standalone), claim verification, implementation, verification, evidence, PR, receipt, monitoring when effective owner | Worker preflight, frontier, wave preview, claim handoff, dispatch, receipt reconciliation, queue re-evaluation, stranded-claim and PR-state recovery, final report |
| Never does | Dispatch other workers; merge PRs; rewrite other people's branches; enter setup unattended | Implement, push, or open PRs; claim before worker preflight passes; re-dispatch an issue that failed in the same run |
| Completes when | PR is open, targets `work.base_branch`, references the issue, and its receipt is emitted | Frontier is empty or only merge-boundary-blocked issues remain, and the final report is written |

## Source disposition

### `pick-up-work`

| Preserve | Remove | Configure | Delegate |
| --- | --- | --- | --- |
| Select one ready issue; read the whole thread; issue-fidelity check before coding; confirm interactively unless unattended; isolated worktree; test-first for behavioral change; verify before PR; UI evidence when UI changed; simplify and self-review; PR linked to issue; machine-readable result; PR as completion boundary | Repository and Project identity; opaque Project, field, and status IDs; MVP milestone prose default; hard-coded label strings; local board-planning script; npm-workspace commands; frontend path; auth bypass; Playwright device and image conventions; private-repo image behavior; FFmpeg install; runtime-specific review commands; indefinite monitoring; the policy breadth that made one skill also a style guide (each cross-cutting rule becomes a capability baseline or a config key) | Eligibility and priority (`work.eligibility`, `work.priority`); milestone (`tracker.milestones`); base branch and branch pattern; worktree policy and root; verification commands and matrix; tests policy; review policy; PR state, template, monitoring; evidence type, dir, embedding; browser mode and auth; claim TTL, heartbeat, recovery | Worktree creation (required capability; git-worktree baseline in-skill); TDD, verification, simplify, issue-fidelity and diff review (optional providers such as Superpowers; self-contained baselines in `references/implementation-baselines.md`); browser evidence (optional; fallback none, disclosed); PR monitoring (optional; fallback coordinator reconcile) |

### `work-the-board`

| Preserve | Remove | Configure | Delegate |
| --- | --- | --- | --- |
| Frontier of unblocked agent issues; wave preview; claim before dispatch; `--max` bounded fan-out of unattended workers; collect results; release failures; re-evaluate; report issues blocked at merge boundaries rather than draining across them | Fixed Project fields; status-as-lock; local board planner; "multiple Agent calls" runtime language; children-inherit-runtime assumption; post-PR same-branch fix rules (moved to the post-PR lifecycle below); issue-number-only priority (now the last tie-break) | `work.concurrency.max_parallel`; shared eligibility, priority, milestone keys; `claims.recovery`; `claims.mirror_status`; Project statuses | Parallel isolated subagents (optional; sequential invocation fallback, disclosed); worker discovery (sibling-discovery rule from the capability spec); PR-state reconciliation (shared lifecycle rules) |

## Shared frontier contract (v1)

Generated into both skills as `references/frontier-rules.md`; drift-tested (ADR 0002).

**Eligibility.** An issue is eligible when all hold: (1) it is an issue, not a PR, and open; (2) it carries every label mapped by `work.eligibility.label_roles` (default `[ready_for_agent]`, the only role ADR 0003 leaves eligible); (3) it carries none of `exclude_label_roles`; (4) with `tracker.project` configured, it is an item of that Project whose status is one of `status_roles`, a stale mirror (`in_progress` with no active claim) being reset per the claim protocol and treated as `todo`; (5) it has no active claim, except that a claim owned by this run's owner id or coordinator prefix counts as eligible-and-held; (6) with `require_no_open_blockers`, every native blocked-by issue is closed (an open PR for a blocker does not clear it; that is the merge boundary); (7) milestone per `tracker.milestones.policy`: `none` no filter, `named` milestone equals `active`, `any` some milestone set.

**Priority.** Apply `work.priority.order` left to right (default `[priority_field, milestone, oldest]` from the capability spec), then issue number ascending as the unconditional final tie-break:

- `priority_field`: rank by option order of the single-select field mapped by `tracker.fields.priority` (first option = highest); unset sorts last; skipped when no Project or field is configured.
- `milestone`: earliest milestone due date first; milestones without a due date next; no milestone last.
- `oldest`: lowest issue number first.

**Computation with `gh`.** (1) `gh issue list --state open --label <ready_for_agent> [--milestone <active>] --json number,title,labels,milestone,url,createdAt --limit 200`; exclusion labels and `any` policy filtered client-side. (2) With a Project: `gh project item-list <number> --owner <owner> --format json`, joined on issue number; keep items in `status_roles`. (3) Dependencies, per `tracker.dependencies.transport` in the capability spec's order: `gh issue view N --json` dependency fields when the installed `gh` exposes them, else `gh api repos/{owner}/{repo}/issues/N/dependencies/blocked_by`, else the GraphQL `blockedBy` connection; each blocker's state must be `closed`. The `Blocked by: #N` body convention is read only when the transport resolves to `body`, and the receipt records the downgrade. (4) Claims: fetch comments only for issues surviving 1–3 and derive the active claim. (5) Sort. Coordinators cache claim and dependency reads per run.

**Filter rule (fixes the source bug).** The source project stated an MVP default in prose that never reached its planner, which filtered by milestone only when passed and otherwise selected every open issue. Here no filter exists outside `config.yaml`: `tracker.milestones.policy` absent means `none` because the capability spec declares that default, and both skills print the effective filters (`labels`, `exclude`, `statuses`, `project`, `milestone`) in the wave preview or selection summary and record them under `frontier.filters` in the receipt. Guided setup for `work-the-board` asks `tracker.milestones.policy` explicitly (an addition to the capability spec's always-ask list; #8 absorbs).

## Claim behavior for the pair

The claim comment, owner-id format, operations, and recovery are those of the capability spec's claim protocol v1; this section fixes only who claims when.

- **Coordinator claims for the worker.** After worker preflight and wave preview, for each wave item the coordinator posts the claim with `owner: <coordinator-owner>/w<n>`, `skill: pick-up-work`, and `expires_at = claimed_at + claims.ttl` (plus `work.pr.monitoring.max_duration` when the worker will own monitoring and cannot heartbeat), mirrors status, then dispatches `pick-up-work --issue N --owner <id> --unattended`. `claim-held` on any item skips it and pulls the next frontier item.
- **Worker verifies on start.** With `--owner`, the worker re-reads the active claim before any mutation; owner mismatch ends the run with `claim-lost` and no side effects. Every later mutation (push, PR open, status, release) re-verifies per the protocol.
- **Standalone worker.** Unattended: claim immediately after selection. Interactive: select, print the summary, confirm, **then** claim, **then** create the worktree. The capability spec's sentence "before any post-selection confirmation" is adjusted here: claiming before confirmation posts and then releases a comment whenever the human declines; claiming right after confirmation keeps the comment thread clean, and the claim step re-reads the active claim so a collision during the confirmation pause yields `claim-held` and a fresh selection, never duplicate work. Maintainer ratifies; #8 aligns its wording.
- **Release rules by outcome.** `pr-opened`: keep the claim active, mirror `in_review`. `needs-info`, `blocked`, `failed`: worker releases with the matching disposition, mirror `todo`. `none-ready`, `claim-held`, `claim-lost`, `setup-required`: nothing to release. If a worker exits without releasing (crash, no receipt), the coordinator releases using the prefix rule with disposition `failed`.
- **Recovery.** Expired claims follow `claims.recovery.expired`. Stranded-claim fast path: at startup a coordinator reads `state/local/runs/` for its own host's coordinator receipts left `status: running`; child claims of those runs that are still active and have no open PR matching `branch_pattern` are treated as abandoned immediately, because the local ledger proves the owning process ended. This is an addition to the protocol's abandonment rule; #8 absorbs.

## Worker flow (`pick-up-work`)

Arguments: `--issue N`, `--owner <id>`, `--unattended`, `--dry-run`, `--fix-pr N` (lifecycle fix mode, below).

1. **Preflight.** Parse config, run the capability spec's validation tiers, resolve capabilities, list downgrades. Missing required keys stop with the setup block (`setup-required`); interactive runs may enter guided setup.
2. **Select or accept.** With `--issue`, verify eligibility (held-by-this-owner counts). Without it, compute the frontier and take the head; empty frontier ends with `none-ready` and the effective filters.
3. **Claim** per the rules above.
4. **Read the thread.** Body, every comment, sub-issues, blocked-by set, linked PRs, and repository conventions (`AGENTS.md`/`CLAUDE.md`, `docs/agents/`, contributing docs).
5. **Issue-fidelity check.** Derive acceptance criteria and touched surfaces. Optional isolated subreview when `work.review.issue_fidelity: required` and subagents exist; baseline is the in-context checklist. Thin spec: interactive asks the human and may record answers as an issue comment; unattended posts one comment listing the questions, applies the `needs_info` label role (the `ready_for_agent` label stays; exclusion keeps the issue out of the frontier), releases, and ends with `needs-info`. An unmet external prerequisite (credentials, environment, a human-execution blocker) ends with `blocked` and a comment.
6. **Worktree.** Baseline `git worktree add <work.worktrees.root>/<branch> -b <branch> <base_branch>` after `git fetch`; `policy: required` stops if creation fails, `preferred` falls back to the checkout with a disclosed downgrade, `off` uses the checkout. Branch name from `work.branch_pattern`.
7. **Implement.** `work.tests.policy` governs test-first (`behavioral`: failing test first for behavior changes; `always`; `never`). Run every `work.verification.commands` entry, once per `matrix` entry when set; `none: true` skips with disclosure. Any failing command is fixed or ends the run with `failed`.
8. **UI evidence.** When the diff touches a configured UI surface and `evidence.ui.type` is not `none`: with browser capability, capture per type into `evidence.ui.dir` and embed per `embed` (`auto` uses discovered visibility); without it, record `evidence: none, reason: browser capability unavailable` in the PR body and receipt.
9. **Simplify and self-review.** Diff review capability when available, else the baseline checklist (dead code, scope creep, missing tests, secrets, unrelated files); `work.review.self_review: required` blocks the PR on unresolved findings.
10. **PR.** Tier 3 rechecks for push and PR open; push; `gh pr create --base <base_branch>` with `--draft` when `work.pr.state: draft`; body from `work.pr.template` or the discovered `.github` template, containing `Closes #N`, verification summary, evidence or its absence, and downgrades.
11. **Verify completion.** `gh pr view --json state,isDraft,baseRefName,headRefName,closingIssuesReferences,url` must show open, correct base, and the issue in closing references; otherwise repair (edit body) or end with `failed`. Mirror `in_review`.
12. **Monitoring disposition** per the post-PR lifecycle; then emit the result line and run receipt.

## Post-PR lifecycle

**Effective owner.** `work.pr.monitoring.owner: worker` makes the worker the owner only when the `pr_monitoring` capability is available (a runtime facility that wakes the run on PR events, such as Claude Code's PR subscription); otherwise the effective owner is `coordinator`, and the receipt records `monitoring: {owner: coordinator, reason: pr_monitoring unavailable}`. `coordinator` means the start of the next `work-the-board` run, or of the next standalone `pick-up-work` run in the same repository, reconciles every active claim that has an open PR; the worker never waits. `none` means the catalog does nothing after the PR opens: the claim is released with disposition `pr-opened` at PR time and humans own the rest. The worker's receipt always names the effective owner and, for `worker`, the `until` timestamp.

**Transitions.** Whoever is the effective owner applies these; each mutation re-verifies claim ownership.

| PR event | Claim | Status / labels | Action |
| --- | --- | --- | --- |
| Merged | Release, disposition `merged` | `done` | Confirm the issue closed (close with reason completed if `Closes` did not fire); remove the local worktree; never delete the remote branch (repository setting decides) |
| Closed unmerged, or changes rejected | Release, disposition `closed` | `todo`; add `needs_info` label role | Post one comment: PR link, closure reason if visible, what a human must decide before re-dispatch. The exclusion rule keeps the issue out of the frontier until a human removes the label |
| Failed checks | Held | unchanged | `on_failed_checks: fix-once`: one fix attempt per failing-check episode, on the same branch in the same worktree; `report`: comment on the PR naming the failing checks, no code change |
| Merge conflict | Held | unchanged | Update the branch from `base_branch` by merge (`gh pr update-branch` or `git merge`); rebase only when every commit on the branch is this owner's; a branch containing another author's commits is never rewritten: comment and release with `failed` |
| Retries exhausted | Release, disposition `failed` | `in_review` (PR stays open) | Comment on PR and issue summarizing attempts; the issue is not re-dispatched automatically |
| Window expired (`max_duration`) with PR open | Held | unchanged | Worker ends with `monitoring-expired`; the coordinator path takes over on the next run |

**Defaults.** `max_duration: 2h`; `on_failed_checks: fix-once`; `work.pr.monitoring.max_retries: 3` total pushes to one PR across fix and conflict attempts (addition; #8 absorbs). Counts are per PR, persisted in the run receipt so a reconciling coordinator continues the same budget.

**Same-branch rules (from the source coordinator, relocated).** Follow-up work on an open PR is always done on that PR's branch and worktree by the effective owner, never as a new issue, branch, or claim. A fix run is `pick-up-work --fix-pr N --owner <id>`: it checks out the existing worktree (recreating it from the remote branch if missing), addresses the failing checks or conflict, re-runs verification, pushes, and emits a receipt with outcome `checks-fixed` or `failed`. The coordinator, when it is the effective owner, dispatches fix runs inside a wave under the same concurrency cap.

## Receipt `receipt.work` v1

Emitted twice with identical content: the last line of worker output, `AGENT_SKILLS_RECEIPT ` followed by one-line JSON, and `state/local/runs/<owner-id>.json` with `/` in the owner id replaced by `_`. Required fields: `receipt`, `family`, `skill`, `owner`, `issue`, `outcome`, `claim.state`, `protocols`. Everything else is optional and additive.

```json
{"receipt": 1, "family": "work", "skill": "pick-up-work", "skill_version": "0.1.0",
 "owner": "claude-code@buildbox/20260902T140512Z-3f9a1c2e/w2",
 "issue": 142, "outcome": "pr-opened",
 "claim": {"state": "active", "disposition": null},
 "pr": {"number": 187, "url": "https://github.com/acme/widgets/pull/187", "state": "open", "draft": false, "base": "main"},
 "branch": "issue-142-export-csv", "worktree": ".worktrees/issue-142-export-csv",
 "verification": [{"name": "lint", "status": "pass"}, {"name": "test", "status": "pass"}],
 "evidence": {"type": "none", "reason": "browser capability unavailable"},
 "monitoring": {"owner": "coordinator", "reason": "pr_monitoring unavailable", "retries_used": 0},
 "frontier": {"filters": {"labels": ["ready-for-agent"], "exclude": ["needs-info", "wontfix"], "statuses": ["Todo"], "project": "acme/7", "milestone": "none"}},
 "downgrades": ["worktrees: preferred fallback not needed", "evidence: none"], "recoveries": [],
 "reason": null, "started_at": "2026-09-02T14:05:40Z", "finished_at": "2026-09-02T14:52:03Z",
 "protocols": {"config": 1, "claim": 1, "receipt.work": 1}}
```

**Outcome vocabulary.** Worker runs: `pr-opened`, `needs-info`, `blocked`, `failed`, `none-ready`, `claim-held` (the brief's `skipped-claimed`), `claim-lost`, `setup-required`. Lifecycle runs and reconciliation: `merged`, `closed`, `checks-fixed`, `retries-exhausted`, `monitoring-expired`. `reason` is required for `needs-info`, `blocked`, `failed`, `claim-lost`, `retries-exhausted`; `pr` is required for `pr-opened` and every lifecycle outcome.

**Compatibility (tolerant but safe).** Consumers accept `receipt.work` majors they list in `references/protocol-contracts.md` (v1 only at first release); ignore unknown fields; stop with guidance on a higher major, an outcome outside the vocabulary, an `owner` that is not the dispatched id, or a missing required field. A worker exit with no parseable result line is reconciled as `failed` with reason `no-receipt` and the claim released by prefix. Renaming, removing, or re-typing any field bumps the major.

## Coordinator flow (`work-the-board`)

Arguments: `--max N` (lowers the cap only), `--dry-run`, `--issue N` (repeatable; restrict the frontier).

1. **Preflight.** Config and validation tiers for the coordinator's required keys. Worker discovery per the capability spec's sibling-discovery rule; the worker's frontmatter must have `name: pick-up-work` and `metadata.contracts` (for example `"config=1;claim=1;receipt.work=1"`) whose majors fall inside the coordinator's accepted ranges; absent or incompatible values stop before any claim with the paired update command. `metadata.contracts` is a frontmatter convention this spec introduces because ADR 0001 forbids reading anything but frontmatter from a sibling; #15 tests it, #17 documents it.
2. **Capacity.** Effective concurrency = min(`work.concurrency.max_parallel`, runtime subagent capacity from `capabilities.subagents` and `runtime_notes`, `--max`). `capabilities.subagents: unavailable` or no isolated-subagent capability sets capacity 1 and prints "sequential fallback". `work.worktrees.policy: off` stops the coordinator: parallel workers must be isolated.
3. **Reconcile and recover.** Apply the post-PR lifecycle to every active claim with an open PR whose effective owner is `coordinator`; run stranded-claim and expired-claim recovery per `claims.recovery`; reset stale mirrors.
4. **Frontier.** Compute per the shared contract; separate the merge-boundary set (eligible except for blockers whose only remaining work is an open PR).
5. **Wave preview.** Print effective filters, ordered frontier, the planned wave of `effective` items, the merge-boundary set, and downgrades. Interactive runs confirm; `--dry-run` stops here with a receipt.
6. **Claim, then dispatch.** Claim each wave item as `<owner>/w<n>`; dispatch each worker in an isolated subagent with its own worktree; sequential fallback runs them one after another.
7. **Collect.** Wait for all workers; parse each result line; validate per the compatibility rules.
8. **Reconcile receipts.** `pr-opened`: keep the claim, record the PR in the coordinator receipt. `needs-info`, `blocked`, `failed`, `claim-lost`, no receipt: ensure the claim is released and exclude the issue for the rest of this run. `setup-required`: surface the block unchanged and stop.
9. **Re-evaluate and loop.** Recompute the frontier and dispatch the next wave until a stop condition: empty frontier; only merge-boundary or excluded issues remain; every remaining item is `claim-held` by another owner; or the human stops an interactive run.
10. **Final report and receipt.** Per-issue outcomes with PR links, claims released, recoveries, downgrades, the merge-boundary list ("blocked on PRs #187, #190; rerun after merge"), and the coordinator receipt (`family: work`, `skill: work-the-board`, `children: [...]`, `status: complete`) written to `state/local/runs/`.

## Configuration consumed

All keys are the capability spec's; the additions this document proposes are marked. Coordinator-relevant excerpt:

```yaml
version: 1
tracker:
  project: { owner: acme, number: 7 }        # required by work-the-board
  fields: { status: Status, priority: Priority }
  statuses: { todo: Todo, in_progress: In Progress, in_review: In Review, done: Done }
  labels: { ready_for_agent: ready-for-agent, needs_info: needs-info, wontfix: wontfix }
  milestones: { policy: none }                # explicit; coordinator setup asks for it (addition)
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
    monitoring: { owner: worker, max_duration: 2h, on_failed_checks: fix-once, max_retries: 3 }  # max_retries is an addition
evidence: { ui: { type: screenshot, dir: docs/evidence, embed: auto } }
browser: { mode: attach, auth: { boundary: user } }
capabilities: { subagents: auto, worktrees: auto, browser: auto, diff_review: auto, pr_monitoring: auto }
```

Additions for #8 to absorb: `work.pr.monitoring.max_retries` (default 3); claim release dispositions `merged` and `closed`; the stranded-claim fast path; asking `tracker.milestones.policy` in coordinator setup. Not config, but adjacent: the `metadata.contracts` frontmatter convention.

## Capabilities

| Capability | Worker | Coordinator | Fallback |
| --- | --- | --- | --- |
| GitHub issue, PR, Project operations via `gh` | required | required | none; preflight stops |
| Worktrees | required (git-worktree baseline ships in-skill) | required (`policy: off` stops) | `preferred` falls back to the checkout, disclosed |
| Isolated subagents | optional (fidelity subreview) | optional (parallel dispatch) | in-context review; sequential worker invocation, disclosed |
| Sequential interactive skill invocation | - | required when subagents are absent | none |
| Browser control | optional (UI evidence) | - | `evidence: none`, disclosed in PR and receipt |
| Diff review | optional | - | in-skill checklist |
| PR monitoring | optional (lifecycle ownership) | - | coordinator reconciliation on the next run |
| TDD, verification, simplify providers (for example Superpowers) | optional providers | - | in-skill baselines; ADR 0004 disclosure table lists them, "Bundled here: No" |

Runtime notes, only where the runtimes genuinely differ: Claude Code can dispatch background subagents with their own worktrees and offers PR-event subscription, so it can satisfy both `subagents` and `pr_monitoring`; Codex runs parallel work through its task facility under the cap recorded in `runtime_notes.codex` and has no PR subscription, so its effective monitoring owner is always `coordinator`. Browser attachment mechanics differ by runtime and live in `runtime_notes`, not in the skill.

## `SKILL.md` outlines

### `pick-up-work`

```yaml
---
name: pick-up-work
description: Take one ready-for-agent GitHub issue from claim to a verified open pull request in an isolated worktree, with configured verification, evidence, and a versioned receipt.
license: MIT
compatibility: GitHub repository with Issues; gh CLI; git worktrees; .agent-skills/config.yaml. Optional: isolated subagents, browser control, diff review, PR monitoring; Superpowers worktree/TDD/verification skills as providers.
metadata:
  author: Christian Dowell
  version: "0.1.0"
  provenance: original
  contracts: "config=1;claim=1;receipt.work=1"
---
```

Sections: Purpose and boundary (one issue, PR is the boundary) · Arguments · Preflight (config tiers, capabilities, downgrades) · Select or accept (frontier head or `--issue`) · Claim (timing rules; `claim-held`, `claim-lost`) · Read the thread · Issue-fidelity check (interactive questions; unattended `needs-info`) · Worktree (baseline command; policy) · Implement and verify (tests policy; commands; matrix) · UI evidence (capture or disclosed none) · Simplify and self-review · Open and verify the PR (state, template, `Closes`, Tier 3) · Monitoring disposition · Fix mode (`--fix-pr`) · Receipt · Safety rules (never rewrite others' branches; never mutate before claim verification; never enter setup unattended).

`references/`: `protocol-contracts.md` (generated), `claim-protocol.md` (generated), `receipt-work.md` (generated schema and vocabulary), `frontier-rules.md` (generated), `post-pr-lifecycle.md` (generated), `setup.md` (generated), `implementation-baselines.md` (TDD, verification, simplify, review checklists), `ui-evidence.md` (capture and embedding rules by visibility).

### `work-the-board`

```yaml
---
name: work-the-board
description: Drain the unblocked ready-for-agent frontier in bounded parallel waves by claiming issues and dispatching pick-up-work workers, reconciling their receipts and PR state.
license: MIT
compatibility: GitHub repository with Issues and a Project; gh CLI; git worktrees; pick-up-work installed (companion); .agent-skills/config.yaml. Optional: isolated subagents (else sequential invocation).
metadata:
  author: Christian Dowell
  version: "0.1.0"
  provenance: original
  contracts: "config=1;claim=1;receipt.work=1"
---
```

Sections: Purpose and boundary (coordinate, never implement) · Arguments · Preflight (config, worker discovery, `metadata.contracts` check) · Capacity · Reconcile and recover (open PRs, stranded and expired claims, stale mirrors) · Frontier and merge boundaries · Wave preview and confirmation · Claim then dispatch (owner suffixes; subagent or sequential) · Collect and validate receipts · Reconcile outcomes · Re-evaluate and stop conditions · Final report and receipt · Safety rules (claim only after preflight; release only by prefix; never re-dispatch a failed issue in-run).

`references/`: `protocol-contracts.md`, `claim-protocol.md`, `receipt-work.md`, `frontier-rules.md`, `post-pr-lifecycle.md`, `setup.md` (all generated), `dispatch-adapters.md` (subagent versus sequential invocation, per-runtime notes), `report-format.md`.

## Acceptance scenarios

1. **Given** no `config.yaml`, **when** `work-the-board` runs unattended, **then** it prints the setup block, claims nothing, discovers no worker, and records `setup-required`.
2. **Given** a validation receipt older than `validation_max_age`, **when** `pick-up-work` starts, **then** Tier 1 re-runs before selection and no claim is posted until it passes.
3. **Given** issue 142 claimed by another owner, **when** the coordinator reaches it in a wave, **then** it records `claim-held`, posts no comment, and dispatches the next frontier item instead.
4. **Given** an installed worker whose `metadata.contracts` says `receipt.work=2`, **when** the coordinator preflights, **then** it stops before any claim, naming the worker, the accepted range, and the paired update command.
5. **Given** a claim held with an open PR that merged since the last run, **when** either skill starts, **then** reconciliation releases the claim with `merged`, confirms the issue is closed, sets status `done`, and removes the worktree.
6. **Given** `on_failed_checks: fix-once` and `max_retries: 3`, **when** checks fail after the fix attempt and the retry budget is spent, **then** the owner comments on PR and issue, releases with `failed`, leaves status `in_review`, and the coordinator does not re-dispatch the issue.
7. **Given** `evidence.ui.type: screenshot` and no browser capability, **when** the worker changes a UI surface, **then** the PR body and receipt record `evidence: none` with the reason and the run still completes with `pr-opened`.
8. **Given** `max_parallel: 3`, runtime capacity 2, and `--max 5`, **when** a wave is planned, **then** exactly 2 workers run and the preview states the effective cap and why.
9. **Given** `tracker.milestones` absent, **when** the frontier is computed, **then** the preview shows `milestone: none` as a declared default and no issue is excluded by milestone.
10. **Given** an unattended worker and an issue with no acceptance criteria, **when** the fidelity check runs, **then** it posts one comment with the questions, adds `needs-info`, releases the claim with `needs-info`, and creates no worktree or branch.
11. **Given** an interactive worker and another agent claiming the same issue during the confirmation pause, **when** the human confirms, **then** the claim step reports `claim-held`, no worktree is created, and selection restarts.
12. **Given** a coordinator that crashed mid-wave on this host, **when** the next coordinator run starts, **then** its child claims without open PRs are recovered immediately via the local run ledger and recorded under `recoveries`.
13. **Given** issue 150 blocked only by issue 142 whose PR is open, **when** the frontier is computed, **then** 150 appears in the merge-boundary report, is never dispatched, and the run ends with "rerun after merge".
14. **Given** a worker that exits without a result line, **when** the coordinator collects, **then** it reconciles `failed` with reason `no-receipt` and releases the claim by prefix.
15. **Given** a PR closed unmerged by a reviewer, **when** reconciliation runs, **then** the claim is released with `closed`, status returns to `todo`, `needs-info` is applied, and one explanatory comment is posted.
16. **Given** `capabilities.subagents: unavailable`, **when** the coordinator runs, **then** it dispatches workers sequentially, prints the fallback, and the receipt lists it under `downgrades`.
17. **Given** a merge conflict on a branch containing another author's commit, **when** the owner attempts to update it, **then** no history is rewritten, a comment explains, and the claim is released with `failed`.
18. **Given** `--dry-run`, **when** the coordinator runs, **then** it prints preview and merge-boundary set, writes a receipt, and posts no claim.

## Risks

- Comment-based claims plus dependency reads cost several API calls per candidate; large frontiers may hit rate limits. Mitigation: label and status filters first, claim and dependency reads only for survivors, per-run cache, `--limit 200` on the initial list.
- Dependency read fields in `gh issue view --json` could not be verified from this session (ADR 0003 open item 3); the transport probe order degrades to REST, which is documented.
- Coordinator-owned reconciliation is only as timely as the next run; a merged PR can hold a claim for hours. Acceptable because claims expire and the issue is closed by `Closes` regardless.
- Fix runs on the same branch can loop on flaky checks; the per-PR retry budget and "no re-dispatch in-run" rule bound it.
- `metadata.contracts` is a catalog convention, not an Agent Skills field; tooling that strips unknown metadata would break preflight. The installer copies frontmatter verbatim, so the risk is limited to third-party validators.

## Open items

1. Maintainer ratifies: interactive claim after confirmation (this spec) versus before (capability spec wording); `max_retries: 3`; `needs_info` on closed-unmerged PRs; sequential fallback rather than a stop when subagents are absent.
2. Whether `work.priority.order` should gain an `unblocks` strategy (issues blocking the most open work first) for coordinators; deferred to the pilot.
3. Whether standalone `pick-up-work` runs should reconcile all open-PR claims or only their own host's; this spec says all, with `confirm_interactive` prompting.
4. The `--fix-pr` mode's interaction with review comments (address reviewer feedback) is out of scope for the first release; `report` remains the safe policy for repositories with active human review.

## Consequences

- **#10 (human-gate wave):** reuse `references/frontier-rules.md` with `needs_decision` in place of `ready_for_agent` and the same priority order; reuse the claim timing rule for the interactive worker; define `receipt.human` with the same result-line marker, required fields, and compatibility rules so a future coordinator can parse both families; `ready-for-human` items stay out of both frontiers.
- **#15 (validation and pilot strategy):** test the eighteen scenarios, the `metadata.contracts` preflight, receipt parsing including malformed lines, drift between each skill's generated references and the canonical sources, the sequential fallback, and the filter echo; add the additions above to the config schema tests once #8 absorbs them.
- **#16 (pilot consumer and acceptance scenarios):** the pilot repository needs a Project with status and priority fields, native blocked-by edges, CI checks that can be made to fail, both runtimes, and at least one UI surface, so scenarios 5–8 and 13–17 can run for real.
- **#17 (release sequencing and readiness gates):** ship the pair together with paired install and update commands; document `metadata.contracts`, the effective monitoring owner per runtime, and the merge-boundary rerun message; the External skills table lists Superpowers providers as optional.

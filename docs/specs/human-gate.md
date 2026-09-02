# Human-gate release wave

Date: 2026-09-02
Question: What should the public single-item and batch human-gate skills preserve, remove, configure, and delegate from the source `pick-up-human-task` and `work-the-human-board`, including their relationship to the work-execution pair?
Resolves: [#10 Specify the human-gate release wave](https://github.com/cdowell09/agent-skills/issues/10)
Inputs: [ADR 0001](../adr/0001-npx-skills-distribution-contract.md), [ADR 0002](../adr/0002-skill-boundaries-and-shared-contracts.md), [ADR 0003](../adr/0003-ticket-decomposition-provider.md), [ADR 0004](../adr/0004-licensing-attribution-and-contribution.md), [Capability configuration spec](capability-configuration.md), [source skill inventory](../research/2026-07-12-source-skill-inventory.md), [source provenance audit](../research/2026-07-12-source-provenance.md), [triage label roles](../agents/triage-labels.md)

Skill names used here (`pick-up-work`, `work-the-board`, `pick-up-human-task`, `work-the-human-board`, `dogfood`, `research-project`, `publish-findings`) are working names; a later naming pass may rename them. The agent pair is specified concurrently in `work-execution.md`; this document keeps its contracts parallel in shape (frontier, claim, receipt v1) and refers to that pair as "the work-execution pair (see `work-execution.md`)" without depending on its text.

## Decision

- **Two skills, one contract.** `pick-up-human-task` clears exactly one human gate with a live human; `work-the-human-board` preflights that worker, computes the gate frontier live, claims one gate at a time, invokes the worker sequentially, reconciles its receipt, keeps a durable skip set, and recovers stranded claims. Neither duplicates the other; both embed the same generated leverage rules, claim protocol, receipt schema, and disposition policy.
- **A human gate is a `needs-decision` issue.** The source project's overloaded `ready-for-human` label is split per ADR 0003: `needs-decision` (a person must decide or specify before agent work can proceed) is the only role the pair selects; `ready-for-human` (a person must do the work) is never selected. The `spec_gate` placeholder role in the capability-configuration spec is retired; decision and specification gates are one queue.
- **Priority is transitive leverage, computed live with `gh` from native blocked-by edges**, with deterministic tie-breaks: transitive count, direct count, oldest, lowest number. A Project status update is an optional accelerator, never the truth.
- **The interactive worker claims.** Both skills use claim protocol v1 with a short human TTL (1h), no Project status mirror by default, and coordinator-prefixed owner ids. The coordinator's skip set lives in `.agent-skills/state/local/human-board.json` and survives interruption.
- **`receipt.human` v1** carries eleven outcomes: `cleared-for-agent`, `cleared-for-human`, `needs-info`, `split`, `deferred`, `closed`, `tapped-out`, `none-ready`, `failed`, `skipped-claimed`, `setup-required`. Consumers tolerate additive fields and stop on unknown majors, outcomes, or claim states.
- **Triage, grilling with domain-doc writes, and ticket decomposition are optional capabilities** with self-contained baselines; the coordinator's interactive-invocation capability is required but has a human-driven form. Inlining the worker's instructions into the coordinator is forbidden.
- **Handoff to the work-execution pair is a printed command, never an automatic invocation.**

## Vocabulary

| Term | Meaning | Label role | Selected by |
| --- | --- | --- | --- |
| Human gate | An open issue that needs a human decision or specification before agent work can proceed: a product choice, a domain-model call, an acceptance boundary, an approval, or a spec that does not yet exist. Clearing it is conversation and tracker mutation, never implementation. | `needs_decision` | this pair only |
| Human execution | An open issue a person must perform: provisioning, credentials, sign-ups, manual checks. | `ready_for_human` | no catalog skill |
| Agent work | A fully specified issue an unattended agent can take to a verified PR. | `ready_for_agent` | the work-execution pair |
| Gate frontier | Eligible gates with no open blockers, ranked by leverage. | | |
| Leverage | Number of open issues a gate transitively blocks through native edges. | | |
| Sitting | One coordinator session: preview, sequential gates, stop on cap, tap-out, or empty frontier. | | |

### Why the split, and migration for a legacy repository

The source project used one `ready-for-human` label for both meanings (acknowledged design debt). A gate is cleared by talking; an execution item cannot be cleared in a sitting and would only inflate the frontier, the cap, and the skip set. ADR 0003 fixes `needs-decision` as an addition to upstream's vocabulary, not a redefinition. A repository that only has the old label is handled without guessing:

1. Setup (capability-configuration spec, Setup flow) asks for `tracker.labels.needs_decision`; if the label does not exist on the repository, setup offers to create it (`gh label create`), interactively only.
2. Immediately after creation, if open issues carry `ready_for_human`, setup offers a one-time interactive migration: for each such issue it shows title and first paragraph and asks "decision gate, human execution, or leave it"; a gate gets `needs_decision` added and `ready_for_human` removed. Unattended runs never migrate.
3. Until migration, the pair selects nothing labelled `ready_for_human`. The coordinator preview prints a one-line hint: "N open `ready-for-human` issues are not gates; rerun setup to classify them if some are decisions."
4. `ready_for_human` is a default member of `human.eligibility.exclude_label_roles`, so an issue mistakenly carrying both labels is excluded and reported, never cleared.

## Preserve, remove, configure, delegate

### `pick-up-human-task` (worker)

| Disposition | Item |
| --- | --- |
| Preserve | One gate per run; full-thread read before any question; present the top candidate with the reason it ranks first; human confirms or skips; decision recorded as an issue comment; label transition on exit; normal outcome is `ready-for-agent`; splitting a gate into smaller issues; gate stays out of In Progress. |
| Remove | Single-owner assumption (claims now carry a run owner); MVP milestone default; fixed Project status-update sections as the candidate source; path-based runtime routing; assignee as claim; hard-coded label strings; the AI disclaimer as unconditional prose. |
| Configure | Candidate source and accelerator (`human.candidates`); milestone and Project filters (`tracker.milestones`, `tracker.project`, `human.eligibility.status_roles`); eligibility and exclusion roles; leverage scope; timezone; claim TTL and mirror; disclosure text and mode; domain-doc locations; split method. |
| Delegate | Triage-style classification of the exit transition → optional `triage` capability (baseline: the worker's own transition table). Grilling with domain-doc writes → optional `grilling` capability (baseline: `references/grilling-baseline.md`, comment-only record). Ticket decomposition → optional `ticket-decomposition` capability per ADR 0003 (baseline: native sub-issues created by the worker, or manual). |

### `work-the-human-board` (coordinator)

| Disposition | Item |
| --- | --- |
| Preserve | One initial preview; live frontier recomputed before every gate; highest-leverage root first; exactly one gate in flight; skip set; stop on cap or tap-out; handoff of agent-ready output to the agent coordinator. |
| Remove | In-memory-only skip set; unclaimed gates (concurrent sessions could duplicate a gate); fixed board query limit; MVP milestone; status-update sections as truth; duplicated candidate/root logic (now generated shared rules); runtime-specific skill-call syntax. |
| Configure | Cap, tap-out phrase, skip-set TTL (`human.session`); everything the worker configures (inherited; coordinators inherit worker requirements per the required-keys matrix). |
| Delegate | Gate clearing → the worker, always. Skill invocation → `interactive-skill-invocation` capability (runtime-driven or human-driven form). Agent dispatch → the work-execution pair, by printed command only. |

## Eligibility and priority (shared contract, `references/leverage-rules.md`)

**Eligible** iff all hold: open; carries at least one `human.eligibility.label_roles` label (default `[needs_decision]`); carries no `exclude_label_roles` label (default `[needs_info, wontfix, ready_for_human]`); passes `tracker.milestones.policy`; when a Project is configured, its item status is in `human.eligibility.status_roles` (default `[todo]`); has no active claim by another owner; and, with `require_no_open_blockers: true` (default), no open native blocker. Blocked gates are shown in the preview as "waiting on #N" but are not candidates. This mirrors `work.eligibility` in shape.

**Leverage.** Let `E` be the eligible set and `V` the set of open issues in the repository (`human.priority.leverage_scope: repository`, default) or only those passing the same milestone/Project filters (`filters`). For a gate `g`:

1. Fetch `g`'s native *blocking* list (issues that `g` blocks) using `tracker.dependencies.transport` order: `gh issue view --json` dependency fields when the installed `gh` exposes them, else the REST `dependencies/blocking` endpoint, else GraphQL. Body text is never read.
2. Breadth-first expand: for each fetched issue that is open and in `V`, add it to `reach(g)` and fetch its blocking list; closed issues are not traversed (a closed blocker no longer blocks, so nothing behind it is blocked by `g`). Maintain a visited set; cycles terminate. Cache each issue's blocking list per run.
3. `transitive(g) = |reach(g)|`, `direct(g)` = number of open issues `g` blocks directly.

**Order:** `transitive` descending, then `direct` descending, then `created_at` ascending (oldest first), then issue number ascending. The order is total, so two sessions over the same graph produce the same list. `human.priority.order` exists for shape parity with `work.priority.order` but accepts only this sequence in v1.

**Accelerator.** With `human.candidates.accelerator: status-update` and a configured Project, the newest Project status update posted "today" is read once; issues it references are verified first so the top candidate can be shown quickly, and its text is quoted as context in the preview. It never adds, removes, or reorders candidates. "Today" is the calendar day in `human.timezone` (IANA name; default `UTC`); the same zone formats every human-facing timestamp and dates the session file.

## Claim behavior (claim protocol v1, `references/claim-protocol.md`)

- **The interactive worker claims** before confirmation and before any question, using the capability-configuration comment format with `skill: pick-up-human-task`. This closes the source gap where two concurrent sessions could clear the same gate.
- **TTL.** `human.claim.ttl` (default `1h`) overrides `claims.ttl` for this family; `claims.heartbeat` applies unchanged. A gate is a conversation, so an hour of silence is a stronger abandonment signal than for agent work.
- **Mirror.** `human.claim.mirror_status` defaults to `false`: a cleared gate must not appear In Progress (it is not being worked; it is about to be worked by an agent or a person). With `true`, the mirror follows the capability-configuration spec.
- **Coordinator claims per gate** as `<coordinator-owner>/w<n>` before handing off, passes that owner id to the worker (idempotency: the worker sees its own owner), and releases on skip, tap-out, or a receipt that leaves `claim: held`. Release `disposition` carries the receipt outcome.
- **Held by another owner** → the worker returns `skipped-claimed`; the coordinator adds the issue to the skip set with `until: <expires_at>`.
- **Durable skip set.** `.agent-skills/state/local/human-board.json` (gitignored with the rest of `state/local/`):

```json
{
  "session_state": 1,
  "owner": "claude-code@buildbox/20260902T140512Z-3f9a1c2e",
  "started_at": "2026-09-02T14:05:12Z",
  "timezone": "UTC",
  "cap": 5,
  "cleared": [{ "issue": 142, "outcome": "cleared-for-agent" }],
  "skipped": [{ "issue": 150, "reason": "deferred", "until": null }, { "issue": 151, "reason": "claimed", "until": "2026-09-02T15:10:00Z" }],
  "in_flight": { "issue": 158, "owner": "claude-code@buildbox/20260902T140512Z-3f9a1c2e/w3", "claimed_at": "2026-09-02T14:40:01Z" },
  "updated_at": "2026-09-02T14:40:01Z"
}
```

- **Recovery on startup.** If the file's `updated_at` is within `human.session.skip_set_ttl` (default `24h`), the coordinator restores `skipped` and `cleared` and reports them. If `in_flight` names an issue whose active claim carries this file's owner as prefix, it asks (always interactive) whether to resume that gate first or release it (`disposition: abandoned`, reason `session-interrupted`). Other active claims with this machine's coordinator prefix follow `claims.recovery.expired` once expired; unexpired ones are listed and left alone. An older file is archived as `human-board.<started_at>.json`.

## Worker flow: `pick-up-human-task`

1. **Preflight.** Parse config, fingerprint, Tier 2 fast path or Tier 1; stop with guidance (`setup-required`) on missing keys; detect optional capabilities via sibling-skill discovery and report each downgrade once.
2. **Candidates.** Compute the frontier and leverage live; optional accelerator. If an issue number or owner id was passed by the coordinator, verify it is still eligible instead of ranking.
3. **Present.** Show the top candidate: title, transitive/direct counts, first three blocked issues by title, age, and why it ranks first. Ask confirm / skip / stop. Skip → next candidate (`deferred` receipt only when the run ends without a gate); stop → `deferred` with reason `human-declined`.
4. **Claim.** Post the claim comment (Tier 3 recheck first). `claim-held` → `skipped-claimed`.
5. **Read.** Whole issue: body, every comment, sub-issues, blocked-by and blocking lists, linked PRs. Summarize in five lines before asking anything.
6. **Grill.** With the `grilling` capability, invoke the provider by name and let it drive; otherwise run the baseline: list the open questions the gate poses, ask them one at a time with concrete options, propose a decision, ask for confirmation, and stop when the human says the gate is decided. Runtime note: use the runtime's structured question prompt where one exists, else numbered plain-text options.
7. **Record.** Post one decision comment: decision, rationale, consequences for blocked issues, then the disclosure line per `human.disclosure` (default `append`, catalog default text "Recorded with AI assistance during a human-gate session; the decision is the human's."; `off` omits it; `text` replaces it).
8. **Domain docs.** If the repository has a glossary or ADR convention (configured in `human.domain_docs`, else discovered: `docs/agents/domain.md`, `CONTEXT.md`, `docs/adr/`), offer to write or update the term or decision record; changes go on a branch and are announced, never merged silently. Without a convention, the comment is the record. The receipt lists `docs_written`.
9. **Split (optional).** Method per `human.split.method`: `ticket-decomposition` runs the ADR 0003 path (preflight provider, emit overlay, prompt `/to-tickets #<n>`, verify and repair native sub-issue and blocked-by edges); `sub-issues` creates native sub-issues with one disposition label each from the disposition policy; `manual` records the intended slices in the comment and returns `deferred`; `auto` picks `ticket-decomposition` when available, else `sub-issues`. After a split the parent loses `needs_decision`, its outgoing blocking edges are re-pointed to the sub-issues the human names (default: all), and it stays open unlabelled as a tracking issue unless the human says the decision is complete, in which case it is closed.
10. **Transition.** Exactly one: remove `needs_decision` and add `ready_for_agent` (gate becomes agent work); add `ready_for_human` (a person must execute); add `needs_info` (unanswerable now; comment says what is needed); close with `state_reason` (`decided`, `not_planned`, or `duplicate`, no work remains; closing releases its native blocked-by edges). With the `triage` capability, the provider proposes the transition; the worker applies only transitions in this table.
11. **Release** the claim with `disposition: <outcome>`. Never set In Progress unless `human.claim.mirror_status: true`.
12. **Receipt.** Emit the receipt block in the conversation and write `state/local/runs/<owner-id>.json`.

## Receipt v1 (`receipt.human`, `references/receipt-human.md`)

Envelope, parallel to the work-execution pair's `receipt.work`:

````markdown
<!-- agent-skills:receipt human v1 -->
```yaml
receipt: 1
family: human
skill: pick-up-human-task
issue: 142
owner: claude-code@buildbox/20260902T140512Z-3f9a1c2e/w1
outcome: split
claim: released            # released | held | none | lost
transition: { from: needs-decision, to: null, closed: false }
link: https://github.com/acme/widgets/issues/142#issuecomment-2210001
reason: ""
leverage: { transitive: 7, direct: 2 }
created: [{ issue: 160, disposition: agent-doable }, { issue: 161, disposition: human-execution }]
edges_repointed: [{ from: 142, to: 160, blocked: 143 }]
docs_written: [docs/adr/0007-export-format.md]
capabilities: { grilling: provider, triage: baseline, ticket_decomposition: to-tickets }
started_at: 2026-09-02T14:06:00Z
finished_at: 2026-09-02T14:31:12Z
```
````

| Outcome | Meaning | Transition | Claim |
| --- | --- | --- | --- |
| `cleared-for-agent` | Decided or specified; this issue is now agent work | `needs_decision` → `ready_for_agent` | released |
| `cleared-for-human` | Decided; a person must execute this issue | `needs_decision` → `ready_for_human` | released |
| `needs-info` | Cannot be decided now; comment names the missing input | + `needs_info` | released |
| `split` | Decision produced sub-issues (`created`, `via` in `capabilities`) | parent unlabelled or closed | released |
| `deferred` | Human skipped or declined; nothing changed | none | released or none |
| `closed` | Decided; no work remains on this issue; blockers released | closed with `state_reason` | released |
| `tapped-out` | Human ended the session mid-gate; a progress comment records where it stopped | none | released |
| `none-ready` | No eligible gate | none | none |
| `failed` | Error; `reason` required (`claim-lost` is a reason, not an outcome) | unspecified | released if held, else lost |
| `skipped-claimed` | Selected gate held by another owner | none | none |
| `setup-required` | Preflight stop; nothing mutated | none | none |

ADR 0003's result name `decomposed` is expressed as `split` with `capabilities.ticket_decomposition: to-tickets`; maintainer ratifies the single name.

**Compatibility (tolerant-but-safe).** Consumers ignore unknown fields; accept `receipt: 1` and any lower major with documented upgrade rules (none yet); stop on a higher major naming the producer and the update needed; stop on an outcome or `claim` value outside the tables (never guess semantics); treat a receipt whose `owner` lacks the coordinator's prefix as foreign and leave its claim alone; treat a missing `issue`, `outcome`, or `claim` as `failed` with reason `malformed-receipt` and stop the loop. The claim protocol's release `disposition` gains the outcome names above as additive values (#8 absorbs).

## Coordinator flow: `work-the-human-board`

1. **Preflight.** Config and validation as the worker; discover `pick-up-human-task` by sibling-skill order and read only its frontmatter; compare its embedded `receipt.human` and `claim` majors (from `metadata.contracts`, a `name=major` list the generator writes) with its own: equal → proceed; worker higher → stop naming the coordinator update; worker lower with upgrade rules → proceed and warn. Verify the `interactive-skill-invocation` capability form (below). Recover the session file. Claim nothing before this step completes.
2. **Preview.** Frontier with leverage, blocked gates, unmigrated `ready_for_human` count, restored skip set, cap, tap-out phrase; ask to begin.
3. **Loop.** Recompute the frontier live; pick the highest-leverage root not in the skip set (expired `until` entries re-enter); claim as `/w<n>`; write `in_flight`; invoke the worker with the issue number and owner id; reconcile the receipt: `cleared-*`, `closed`, `split`, `needs-info` → count toward the cap and record; `deferred`, `skipped-claimed` → skip set; `tapped-out` → stop; `none-ready` → stop; `failed` → release the claim if still held, record, ask whether to continue; `held` in any receipt → release. Stop when `cleared` reaches `human.session.cap` (default 5), on the tap-out phrase (`human.session.tap_out_phrase`, default `tap out`, matched case-insensitively in any human reply to the coordinator) or an explicit stop, or when the frontier is empty.
4. **Report.** Per-gate outcomes and links; the new agent frontier; and, when any outcome was `cleared-for-agent`, `closed`, or `split` with agent-doable children, a ready-made handoff: "Now unblocked for agents: #143, #144, #160. Run `/work-the-board` to dispatch them." If that pair is not discoverable, print ADR 0001's paired install command instead. The coordinator never invokes it.

**Invocation forms.** `interactive-skill-invocation` is required. *Runtime-driven*: the coordinator invokes the worker in the same interactive session. *Human-driven*: the coordinator prints `Run: /pick-up-human-task #158 --owner <id>`, waits for `state/local/runs/<owner-id>.json`, then reconciles it. Both satisfy the capability; `capabilities.interactive_skill_invocation: unavailable` forces the human-driven form. **Inlining the worker's instructions is not allowed**: ADR 0001 forbids reading a sibling's files beyond frontmatter, ADR 0002 forbids a coordinator duplicating worker behavior, and an inlined copy would drift from the worker's generated contracts. A runtime supporting neither form stops at preflight.

## Relationship to the work-execution pair (see `work-execution.md`)

- **Shared keys and roles.** `tracker.*`, `claims.*`, `capabilities.*`, `tracker.dependencies.transport`, and the label roles `ready_for_agent`, `needs_info`, `wontfix`. The families' eligibility sets are disjoint by construction: `needs_decision` here, `ready_for_agent` there; `ready_for_human` in neither.
- **Same graph, same algorithm shape.** Both compute a frontier over native blocked-by edges with `require_no_open_blockers`; they differ in candidate roles, priority (leverage versus `work.priority.order`), and cadence (one live human versus `max_parallel` unattended workers).
- **How a cleared gate enters the agent frontier.** (a) `cleared-for-agent`: the same issue now carries `ready_for_agent`; it is agent-eligible immediately if it has no open blockers. (b) `closed`: its native edges release, so the issues it blocked become agent-eligible when they carry `ready_for_agent`. (c) `split`: sub-issues labelled `ready_for_agent` with no open blockers are eligible; re-pointed edges keep downstream ordering. In all cases the agent coordinator sees the change on its next frontier recomputation; nothing is pushed.
- **Concurrent coordinators are safe.** Disjoint eligibility means no issue is a candidate of both; the shared claim comment guards accidental overlap (an issue carrying both roles is excluded by both families and reported); the human family does not mirror status by default, so the agent coordinator's stale-mirror reset never touches a gate. The only interaction is beneficial: gates cleared during an agent wave are dispatched in that wave's next reevaluation.

## Configuration

Reused capability-configuration keys are shown once; the `human` section is new and mirrors `work` in shape. Additions for #8 are marked.

```yaml
version: 1
tracker:
  project: { owner: acme, number: 7 }      # required for both human skills
  fields: { status: Status }
  statuses: { todo: Todo, in_progress: In Progress, done: Done }
  labels:
    needs_decision: needs-decision         # required; spec_gate placeholder retired (#8)
    ready_for_agent: ready-for-agent
    ready_for_human: ready-for-human       # optional; excluded, never selected
    needs_info: needs-info
    wontfix: wontfix
  milestones: { policy: none, active: "" }
  dependencies: { transport: auto, sub_issues: native }
claims: { ttl: 4h, heartbeat: 30m, recovery: { expired: reclaim, confirm_interactive: true } }

human:                                      # NEW section (#8)
  eligibility:
    label_roles: [needs_decision]
    status_roles: [todo]
    exclude_label_roles: [needs_info, wontfix, ready_for_human]
    require_no_open_blockers: true
  priority:
    order: [leverage, direct, oldest]       # v1 accepts only this sequence
    leverage_scope: repository              # repository | filters
  candidates:
    accelerator: none                       # none | status-update
  timezone: UTC                             # IANA zone; defines "today"
  claim:
    ttl: 1h                                 # overrides claims.ttl for this family
    mirror_status: false
  session:
    cap: 5
    tap_out_phrase: "tap out"
    skip_set_ttl: 24h
  disclosure:
    mode: append                            # append | off
    text: ""                                # empty = catalog default sentence
  domain_docs:
    glossary: ""                            # discovered when empty
    adr_dir: ""
  split:
    method: auto                            # auto | ticket-decomposition | sub-issues | manual

capabilities:
  ticket_decomposition: { provider: to-tickets, user_story_traceability: auto }
  interactive_skill_invocation: auto
  grilling: auto                            # NEW: auto | available | unavailable (#8)
  triage: auto                              # NEW (#8)
```

Required-keys matrix changes for #8: `tracker.labels.spec_gate` row removed; `human.eligibility` R for both human skills; `human.session.cap` O (default 5); `state/local/human-board.json` added to the layout; release `disposition` enumeration extended; `capabilities.grilling` and `capabilities.triage` added.

## Capabilities

| Capability | Worker | Coordinator | Fallback when absent |
| --- | --- | --- | --- |
| GitHub via `gh` (issues, labels, comments, dependencies, sub-issues, Projects) | required | required | stop at preflight |
| Live human in session | required | required | stop; these skills never run unattended |
| Interactive skill invocation (runtime- or human-driven) | - | required | human-driven form; stop if neither |
| `grilling` (provider example: upstream `grill-with-docs`) | optional | inherited | `references/grilling-baseline.md`; comment-only record; downgrade reported |
| `triage` (provider example: upstream `triage`) | optional | inherited | worker's own transition table |
| `ticket-decomposition` (`to-tickets`, ADR 0003) | optional | inherited | native sub-issues by the worker, or `manual` |
| Comment editing (heartbeat, release edit) | optional | optional | release marker comment; `heartbeat: unsupported` |
| Repository writes (domain docs) | optional | - | comment-only record |

External providers are listed in the README `## External skills` table and in `compatibility` (ADR 0004); none is the only way to satisfy a capability.

**Runtime notes (only where they differ).** *Claude Code*: runtime-driven invocation through the skill tool; structured question prompts (AskUserQuestion-style) for grilling options and confirm/skip/stop. *Codex*: runtime-driven invocation only when the worker's `agents/openai.yaml` allows implicit invocation (the pilot confirms, ADR 0003 open item 4); otherwise human-driven; questions as numbered plain-text options. Everything else is identical.

## `SKILL.md` outlines

### `skills/pick-up-human-task/SKILL.md`

```yaml
---
name: pick-up-human-task
description: Select the highest-leverage human decision gate from a GitHub board, claim it, work through the decision with the human, record it, and hand the issue on as agent work, human work, or sub-issues.
license: MIT
compatibility: GitHub repository with Issues (Projects optional but recommended); gh CLI with dependency and sub-issue support; a live human in the session. Optional providers: grill-with-docs, triage, to-tickets (mattpocock/skills).
metadata:
  author: Christian Dowell
  version: "0.1.0"
  provenance: original
  contracts: "config=1 claim=1 receipt.human=1"
---
```

- **Purpose and boundary** — one gate, one human, no implementation, no In Progress.
- **Preflight** — config, validation tiers, capability detection, downgrade report.
- **Find the gate** — frontier and leverage per `references/leverage-rules.md`; accelerator.
- **Confirm and claim** — present with reasons; confirm/skip/stop; claim per `references/claim-protocol.md`.
- **Understand the thread** — full read; five-line summary.
- **Decide with the human** — provider or `references/grilling-baseline.md`; one question at a time.
- **Record** — decision comment with disclosure; domain docs when a convention exists.
- **Split when needed** — `references/disposition-policy.md`; decomposition, sub-issues, or manual; edge repair and re-pointing.
- **Transition and release** — exactly one transition; release with disposition.
- **Receipt** — emit and write per `references/receipt-human.md`.
- **Safety rules** — never select `ready-for-human`; never guess an unclear decision; never merge doc changes; never mutate after `claim-lost`.

`references/`: `protocol-contracts.md` (generated), `setup.md` (generated), `claim-protocol.md` (generated), `receipt-human.md` (generated), `leverage-rules.md` (generated), `disposition-policy.md` (generated, ADR 0003), `grilling-baseline.md`, `runtime-notes.md`.

### `skills/work-the-human-board/SKILL.md`

```yaml
---
name: work-the-human-board
description: Clear human decision gates one at a time in a single sitting by ordering them by leverage, claiming each, invoking pick-up-human-task, and reconciling results until a cap or tap-out.
license: MIT
compatibility: Requires pick-up-human-task installed in the same runtime (install and update the pair together); GitHub repository with Issues and a Project; gh CLI; a live human; a runtime that can invoke a skill interactively or a human willing to run the printed command.
metadata:
  author: Christian Dowell
  version: "0.1.0"
  provenance: original
  contracts: "config=1 claim=1 receipt.human=1"
---
```

- **Purpose and boundary** — sequencing and reconciliation only; never clears a gate itself.
- **Preflight** — config; worker discovery and contract compatibility; invocation form; session recovery.
- **Preview** — frontier, blocked gates, migration hint, restored skip set, cap.
- **Loop** — recompute; select; claim `/w<n>`; invoke; reconcile per `references/receipt-human.md`; update `references/session-state.md` file; stop conditions.
- **Recovery** — stranded claims and in-flight gate on startup; failed receipts.
- **Report and handoff** — outcomes, new agent frontier, printed work-execution command per `references/handoff.md`.
- **Safety rules** — never inline the worker; never invoke the agent coordinator; never claim before preflight completes.

`references/`: `protocol-contracts.md`, `setup.md`, `claim-protocol.md`, `receipt-human.md`, `leverage-rules.md` (all generated), `session-state.md`, `handoff.md`, `runtime-notes.md`.

## Acceptance scenarios

1. **Legacy label overload.** Given a repository whose only human label is `ready-for-human` on six open issues, when `pick-up-human-task` runs interactively, then setup offers to create `needs-decision`, walks the six issues asking gate/execution/leave, relabels confirmed gates, and selects nothing labelled `ready-for-human` before or after.
2. **Two sessions racing.** Given #142 is the top gate and two worker sessions confirm it within seconds, when both claim, then exactly one claim comment exists, the second returns `skipped-claimed` with the holder's owner id, and no duplicate decision comment is posted.
3. **Tap-out mid-gate.** Given the coordinator is on gate 3 of 5 and the human replies "tap out" during grilling, when the worker returns `tapped-out`, then a progress comment is posted, the claim is released with `disposition: tapped-out`, the session file records two cleared and one skipped, and the coordinator reports without selecting another gate.
4. **Split into sub-issues with decomposition present.** Given `provider: to-tickets` preflights clean, when the human chooses to split #142, then the overlay is emitted, `/to-tickets #142` is run by the human, native sub-issue and blocked-by edges are verified and repaired, #142 loses `needs-decision`, its edge to #143 is re-pointed, and the receipt is `split` with `created` and `edges_repointed`.
5. **Decomposition capability absent.** Given `provider: none`, when the human chooses to split, then the worker creates native sub-issues itself with one disposition label each, or with `method: manual` records the slices and returns `deferred` with reason `ticket-decomposition unavailable`; it never files the parent as agent work.
6. **Skip persistence across interruption.** Given the coordinator skipped #150 and #151 and was killed with #158 in flight, when it restarts within 24h, then the preview shows both skips restored, asks whether to resume or release #158, and does not re-offer #150 or #151.
7. **Leverage tie-break.** Given gates A (created March, blocks 2 directly, 4 transitively), B (created January, blocks 3 directly, 4 transitively), and C (5 transitively), when the frontier is computed, then the order is C, B, A, and a second run over the same graph yields the same order.
8. **Cap reached.** Given `cap: 2`, when two gates are cleared, then the coordinator stops, does not claim a third, and prints the handoff command listing the unblocked agent issues.
9. **No gates.** Given no open `needs-decision` issue, when either skill runs, then it returns `none-ready` with `claim: none`, mutates nothing, and prints the unmigrated `ready-for-human` count if any.
10. **Grilling capability absent.** Given no grilling provider is discoverable, when a gate is worked, then the baseline asks questions one at a time with options, records the decision comment with the disclosure line, reports `grilling: baseline`, and writes no domain doc unless a convention is discovered and the human agrees.
11. **Closed gate releases the agent frontier.** Given #142 blocks `ready-for-agent` issues #143 and #144, when the worker records the decision and closes #142 with `state_reason: decided`, then both appear in the work-execution frontier on its next recomputation and the handoff command names them.
12. **Receipt major mismatch.** Given a worker whose frontmatter declares `receipt.human=2`, when the coordinator preflights, then it stops before claiming, naming the worker and the coordinator update required.
13. **Human-driven invocation.** Given `interactive_skill_invocation: unavailable`, when the loop selects #158, then the coordinator claims, prints the exact worker command with the owner id, waits for the run receipt file, and reconciles it as if invoked directly.
14. **No mirror by default.** Given a Project with `mirror_status` unset for the family, when a gate is claimed and cleared, then its item status never changes from Todo.

## Risks

- **Leverage cost.** One dependency read per reachable open issue; a large graph makes the preview slow. Mitigation: per-run cache, accelerator for a fast first candidate, `leverage_scope: filters`.
- **`gh` dependency support varies.** Transport order plus a Tier 1 `gh.version` check once the minimum release is confirmed (capability-configuration open item).
- **Grilling providers may mutate docs beyond the worker's disclosure.** The worker announces every file a provider changed and never merges.
- **Splitting can grow the frontier faster than a sitting clears it.** The cap counts cleared gates, not created issues; the report shows net growth.
- **Human-driven invocation depends on the human typing the command correctly.** The coordinator validates the receipt's `issue` and `owner` before reconciling.
- **Migration relabels by human judgement.** It is interactive-only and logged in the setup receipt.

## Open items

1. Maintainer ratifies: `spec_gate` retirement; `split` as the single outcome name (ADR 0003 says `decomposed`); 1h human TTL; `mirror_status: false` default; cap 5; the default disclosure sentence and `append` mode; `ready_for_human` as a default exclusion.
2. Whether `human.candidates.accelerator` is worth keeping at all if the pilot shows leverage computation is fast enough; drop it in v1.1 if unused.
3. Whether `leverage_scope: filters` should also exclude `wontfix` issues from `V`; defaulted to counting every open issue.
4. Codex implicit invocation for the worker (ADR 0003 open item 4) decides which invocation form is the Codex default.

## Consequences

- **#8 (capability configuration):** absorb the `human` section, `capabilities.grilling`/`triage`, the session-state file in the layout, the extended release `disposition` enumeration, retirement of `spec_gate`, and the matrix changes above; `receipt.human=1` is now defined.
- **#9 (work-execution):** nothing to change; this document assumes `receipt.work` uses the same envelope shape (`receipt`, `family`, `skill`, `issue`, `owner`, `outcome`, `claim`, `link`, `reason`) so a generic reconciler can read both.
- **#15 (validation and pilot strategy):** test all fourteen scenarios; drift tests for the five generated references in both skills; the coordinator-worker contract-compatibility check; the session-file recovery paths; the leverage total order against a fixture graph with cycles and closed intermediates; hygiene scan on both skill directories.
- **#16 (pilot consumer and acceptance scenarios):** the pilot repository needs a Project, the three labels, at least one legacy `ready-for-human` issue for scenario 1, a dependency graph with a tie for scenario 7, and both invocation forms exercised on Codex and Claude Code.
- **#17 (release sequencing and readiness gates):** the human-gate pair ships after the capability-configuration keys above are absorbed and after the work-execution pair, because its handoff report references that coordinator by name; document paired install and update commands for both pairs and the optional providers' install commands in the README.
- **#18 (findings publication):** the parent proposal issue it files with `needs-decision` is a first-class gate here; its `handed-off` receipt links to the gate, and this pair's `split` receipt closes the loop.

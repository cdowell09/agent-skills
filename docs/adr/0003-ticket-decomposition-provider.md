# ADR 0003: Ticket decomposition provider and disposition policy

Date: 2026-09-02
Status: Proposed (maintainer ratifies)
Resolves: [#13 Decide the public disposition of to-issues](https://github.com/cdowell09/agent-skills/issues/13)
Inputs: [`to-issues` overlap audit](../research/2026-07-12-to-issues-overlap.md), [Source provenance audit](../research/2026-07-12-source-provenance.md), [Source skill inventory](../research/2026-07-12-source-skill-inventory.md), [ADR 0001](0001-npx-skills-distribution-contract.md), [ADR 0002](0002-skill-boundaries-and-shared-contracts.md), [ADR 0004](0004-licensing-attribution-and-contribution.md), [Capability configuration spec](../specs/capability-configuration.md)

Question: Given that the source project's `to-issues` is excluded as a first-party skill, which external provider or thin attributed extension supplies optional ticket decomposition, and which differentiated behavior must survive?

Skill names used here (`pick-up-work`, `work-the-board`, `pick-up-human-task`, `work-the-human-board`, `dogfood`, `research-project`, `publish-findings`) are working names; a later naming pass may rename them.

## Decision

1. **Provider.** Ticket decomposition is an optional capability named `ticket-decomposition`, supplied by upstream [`to-tickets`](https://github.com/mattpocock/skills/blob/6654f6b60cd9d5be8b54c6fafe44346dabeb3b76/skills/engineering/to-tickets/SKILL.md) from `mattpocock/skills`, installed separately by the consuming repository. The catalog never vendors it, never ships a decomposition skill of its own, and never publishes the source project's `to-issues`.
2. **Shape.** Primary path is **pure composition**: a clean-room `references/disposition-policy.md` generated into the catalog skills that consume decomposition output, plus a short prompt overlay those skills emit into the conversation before the user invokes `/to-tickets`. Fallback is a **thin attributed extension skill** only if the pilot shows the overlay cannot reliably produce correct dispositions. **Upstream contribution** is pursued opportunistically in parallel and never blocks a release.
3. **Surviving behavior** is expressed as a versioned **disposition policy** contract: every slice is `agent-doable`, `human-decision`, or `human-execution`; dispositions map through capability configuration to the label roles `ready-for-agent`, `needs-decision`, and `ready-for-human`; user-story traceability is an optional presentation; native GitHub blocked-by and sub-issue edges are authoritative and are verified after publication.
4. **Attribution.** Composition copies no upstream text, so no MIT notice arises and `THIRD_PARTY_NOTICES.md` carries no `to-tickets` row; the dependency is disclosed in the README `## External skills` table per ADR 0004. The fallback extension is a derivative and ships `skills/<name>/NOTICE.md` plus a register row if it retains any upstream wording or template.
5. **Retirement.** `to-issues` is not published under any name. Catalog documentation mentions it only as history.

## Context

ADR 0002 fixed seven public jobs and left this question open: the findings publisher "may hand complex actions to configured ticket decomposition" and must preserve the proposal when decomposition cannot complete; human decision gates stay distinct from human execution; native GitHub relationships are authoritative. The overlap audit established that the source project's `to-issues` is a lightly adapted MIT derivative of upstream's deleted `to-issues`, that upstream's `to-tickets` covers everything the copy did except the human-gate disposition and user-story presentation, and that native GitHub edges are a tracker-adapter rule rather than a differentiator.

### Upstream state verified on 2026-09-02

Verified against a shallow clone of `mattpocock/skills` at `main` = [`6654f6b`](https://github.com/mattpocock/skills/commit/6654f6b60cd9d5be8b54c6fafe44346dabeb3b76) (committed 2026-08-24, `package.json` version 1.2.3). The GitHub API and MCP server refused this repository from this session, and `docs.github.com` and `cli.github.com` are blocked by the egress proxy, so GitHub's dependency and sub-issue documentation is cited through the overlap audit rather than re-fetched.

| Check | Finding | Source |
| --- | --- | --- |
| Human-gate disposition and labels | **Absent.** The only label applied is the `ready-for-agent` role, "unless instructed otherwise; the tickets are agent-grabbable by construction." No per-slice HITL/AFK classification. | [`to-tickets/SKILL.md`](https://github.com/mattpocock/skills/blob/6654f6b60cd9d5be8b54c6fafe44346dabeb3b76/skills/engineering/to-tickets/SKILL.md) |
| Tracker configuration | Read at run time from `docs/agents/issue-tracker.md` and `docs/agents/triage-labels.md` (five roles: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`, each mapped to a label string), written by `/setup-matt-pocock-skills` and located via an `## Agent skills` block in `CLAUDE.md`/`AGENTS.md`. Not `.agent-skills/config.yaml`. | [`setup-matt-pocock-skills/SKILL.md`](https://github.com/mattpocock/skills/blob/6654f6b60cd9d5be8b54c6fafe44346dabeb3b76/skills/engineering/setup-matt-pocock-skills/SKILL.md), [`triage-labels.md`](https://github.com/mattpocock/skills/blob/6654f6b60cd9d5be8b54c6fafe44346dabeb3b76/skills/engineering/setup-matt-pocock-skills/triage-labels.md) |
| `ready-for-human` meaning | "Requires human implementation." Decisions are not a label; `triage` handles them through `needs-triage` plus grilling. | [`triage/SKILL.md`](https://github.com/mattpocock/skills/blob/6654f6b60cd9d5be8b54c6fafe44346dabeb3b76/skills/engineering/triage/SKILL.md) |
| Native blocked-by / sub-issue | "Use the platform's native blocking / sub-issue relationship where it has one; otherwise set each ticket's 'Blocked by'." The GitHub adapter uses `gh api` against the REST `dependencies/blocked_by` and sub-issues endpoints, with a `Blocked by: #n` body fallback. The docs page admits two open failure modes ([#554](https://github.com/mattpocock/skills/issues/554) sub-issues not created; [#513](https://github.com/mattpocock/skills/issues/513) edges as body text) and recommends `gh issue create --parent`/`--blocked-by` and `gh issue edit --add-sub-issue` as the repair, "since v2.94". | [`issue-tracker-github.md`](https://github.com/mattpocock/skills/blob/6654f6b60cd9d5be8b54c6fafe44346dabeb3b76/skills/engineering/setup-matt-pocock-skills/issue-tracker-github.md), [`docs/engineering/to-tickets.md`](https://github.com/mattpocock/skills/blob/6654f6b60cd9d5be8b54c6fafe44346dabeb3b76/docs/engineering/to-tickets.md) |
| `gh` flags | Confirmed in GitHub CLI source on `trunk`: `issue create --parent/--blocked-by/--blocking`; `issue edit --parent/--remove-parent/--add-sub-issue/--remove-sub-issue/--add-blocked-by/--remove-blocked-by/--add-blocking/--remove-blocking`. Minimum release version not verifiable from this session. | [`cli/cli` create](https://github.com/cli/cli/blob/trunk/pkg/cmd/issue/create/create.go), [edit](https://github.com/cli/cli/blob/trunk/pkg/cmd/issue/edit/edit.go) |
| Invocation mode | **User-invoked only**: frontmatter `disable-model-invocation: true`; `agents/openai.yaml` sets `allow_implicit_invocation: false`. | `to-tickets/SKILL.md`, `to-tickets/agents/openai.yaml` |
| Install command | `npx skills@latest add mattpocock/skills` with interactive selection, or `/plugin install mattpocock-skills` in Claude Code; `setup-matt-pocock-skills` must be installed alongside. | [README](https://github.com/mattpocock/skills/blob/6654f6b60cd9d5be8b54c6fafe44346dabeb3b76/README.md) |
| Upstream appetite | Open issue [#815](https://github.com/mattpocock/skills/issues/815) (2026-08-08) asks `wayfinder` to record each ticket's HITL/AFK verdict as a label; no maintainer response. No issue asks `to-tickets` for a human-gated disposition. | GitHub search, 2026-09-02 |
| License | MIT, "Copyright (c) 2026 Matt Pocock". | [LICENSE](https://github.com/mattpocock/skills/blob/6654f6b60cd9d5be8b54c6fafe44346dabeb3b76/LICENSE) |

Three findings shape the decision more than the audit anticipated. `to-tickets` is user-invoked only, so "automatic handoff" cannot be a skill-to-skill call; it means preparing the input, emitting the overlay, and telling the user the exact command. Upstream reads its own configuration files, so label strings must agree across both files and the catalog's setup validation must check that. And "unless instructed otherwise" is an explicit extension point: a prompt overlay is a supported way to change labels, not a hack.

## Provider contract

### Declaring the capability

A consuming repository declares the provider in `.agent-skills/config.yaml`, using the key names the capability-configuration spec owns; this ADR fixes the semantics. Roles are written here by their default label strings; the spec's keys are the snake_case forms.

```yaml
version: 1
tracker:
  labels:
    ready_for_agent: ready-for-agent   # agent-doable slices; consumed by pick-up-work / work-the-board
    needs_decision: needs-decision     # human-decision gates; consumed by pick-up-human-task / work-the-human-board
    ready_for_human: ready-for-human   # human-execution work; consumed by no catalog skill
  dependencies:
    transport: auto                    # gh flags -> REST -> GraphQL -> body (downgrade)
    sub_issues: native
capabilities:
  ticket_decomposition:
    provider: to-tickets               # none | to-tickets | manual
    user_story_traceability: auto      # auto | off
```

- `provider: none` (default when absent): complex proposals are deferred, never filed.
- `provider: to-tickets`: upstream `to-tickets` is installed and configured for GitHub in this repository.
- `provider: manual`: no tool decomposes; the publisher files the parent proposal as a `needs-decision` gate and a human decomposes by hand through the human-gate worker. An explicit opt-in, so the "never silently file as a single issue" rule holds: the parent is labelled as a gate, not as work.

The catalog adds one role beyond upstream's five, `needs-decision`, and reuses `ready-for-agent` / `ready-for-human` with upstream's meanings. The shared strings must equal those in upstream's `docs/agents/triage-labels.md`; the spec's Tier 3 "decomposition handoff" recheck verifies this and that all three labels exist, since upstream's setup does not create labels.

### Installing the provider

Documented in the catalog README next to the companion-pair commands, following ADR 0001's canonical form:

```bash
npx skills add mattpocock/skills \
  --skill to-tickets \
  --skill setup-matt-pocock-skills \
  --agent codex \
  --agent claude-code \
  --yes
```

Then run `/setup-matt-pocock-skills` once, choose GitHub, and keep label names equal to those in `.agent-skills/config.yaml`. Claude Code users may use `/plugin install mattpocock-skills` instead. Both are documented as "install separately," never "included here." Pinning via `mattpocock/skills#<ref>` is the consumer's choice; the catalog tests against upstream's default branch and records the tested revision in release notes.

### Preflight and absent-provider behavior

`publish-findings` and `pick-up-human-task` preflight the capability before any side effect that depends on it, in this order:

1. `capabilities.ticket_decomposition.provider` is declared and is not `none`.
2. For `to-tickets`: the skill is found by the spec's sibling-skill discovery order, and upstream's tracker configuration resolves to GitHub for the same repository the catalog configuration names.
3. The three label roles exist on the repository and the shared strings agree with upstream's mapping.

Any failed step marks the capability **unavailable** and the skill reports the downgrade with the install command above. `publish-findings` then files simple actions as usual and records each complex Proposed Action as `deferred` with reason `ticket-decomposition unavailable`, leaving the proposal unchanged in the artifact or ledger and the receipt retryable; it never files a complex action as a single issue or reclassifies it as simple. `pick-up-human-task` still presents the gate but disables its "decompose into tickets" path, offering manual decomposition under the same policy or result `deferred`.

When available, the handoff is:

1. The publisher files (or, on retry, reuses) one **parent proposal issue** carrying the finding ID, evidence links, acceptance intent, and any user stories, labelled `needs-decision` and nothing work-eligible, and records outcome `handed-off` with the parent URL. The human-gate pair surfaces it; neither agent-board skill does.
2. In a live session the human-gate worker emits the overlay below and the human runs `/to-tickets #<parent>`. Upstream reads "whatever is already in the conversation context," so no file path or config plumbing is needed.
3. After publication the worker verifies native edges (below) and records result `decomposed` with the created issue URLs.

## Shape: why composition, when extension, and upstream in parallel

| Shape | What it is | Verdict |
| --- | --- | --- |
| (a) Pure composition | Clean-room `references/disposition-policy.md` generated into `publish-findings` and `pick-up-human-task` (drift-tested per ADR 0002), plus a prompt overlay those skills print before the user runs `/to-tickets`. | **Primary.** Fits ADR 0001 (no vendoring, no dependency field), uses upstream's own "unless instructed otherwise" seam, tracks upstream improvements for free, carries no notice obligation, adds no skill to maintain. |
| (b) Thin attributed extension | A catalog skill (working name `decompose-with-dispositions`) invoked instead of `/to-tickets`, re-implementing quiz-and-publish with dispositions built in. | **Fallback only.** Upstream is user-invoked only, so a wrapper cannot call it; a real extension must reproduce the decomposition workflow, which is the duplication the audit rejected, and inherits MIT obligations if any upstream wording is kept. |
| (c) Upstream contribution | Propose an optional disposition step in `to-tickets`, off by default, roles supplied by `triage-labels.md`. Issue #815 shows adjacent demand. | **Parallel track, not a dependency.** If accepted, the overlay shrinks to "enable dispositions with these roles"; the policy file remains the consumer-side contract. |

Switching criteria, measured by the pilot (#15): (a) → (b) if, after one round of overlay tuning, more than one in ten decomposition runs per runtime applies `ready-for-agent` to a human slice or omits the disposition question, or if two consecutive upstream releases force the overlay to be reworded; (b) → (a) when upstream exposes a disposition or label hook; (c) becomes primary when an upstream release ships disposition support, with the policy file kept as the consumer contract.

## Disposition policy contract (v1)

This is the differentiated behavior that survives. It lives as `references/disposition-policy.md` in each consuming skill, with one canonical authoring source and drift tests. Its normative content:

### Classification

Each proposed slice receives exactly one disposition:

| Disposition | Meaning | Label role | Who acts | Typical position in the graph |
| --- | --- | --- | --- | --- |
| `agent-doable` | Fully specified; an unattended agent can complete it to a verified PR. Upstream's default. | `ready-for-agent` | `pick-up-work` via `work-the-board` | Leaf or mid-chain |
| `human-decision` | Cannot start until a human decides or specifies something (product choice, domain-model call, acceptance boundary, approval). No implementation here. | `needs-decision` | `pick-up-human-task` via `work-the-human-board`, which resolves it and normally re-labels the unblocked work `ready-for-agent` | Blocker of agent slices; usually near the root |
| `human-execution` | A human must do the work itself: provisioning, credentials, vendor sign-up, account actions, manual verification no agent can do. | `ready-for-human` | No catalog skill; a human closes it | Blocker of downstream slices |

Rules:

- Default is `agent-doable`; the overlay changes only the exceptions, leaving upstream's granularity, blocking-edge, and wide-refactor behavior untouched.
- A `human-decision` slice records the question and the slices it gates, not the work; when the decision lands, the human-gate worker re-labels the gated slices rather than creating new ones.
- A slice is never both `needs-decision` and `ready-for-human`; split it if a human must both decide and execute.
- The quiz shows a **Disposition** line per slice and confirms each non-default one; publication applies exactly one disposition label per slice and never `ready-for-agent` to a non-agent slice.

### Why `ready-for-human` must be split

The source project used one `ready-for-human` label for both decision gates and human execution; the inventory records the overload as acknowledged design debt, and upstream `triage` defines `ready-for-human` strictly as "needs human implementation." The two meanings have different consumers: the human-gate pair's frontier and leverage calculation must contain only gates it can clear in a sitting, and an execution item there is un-clearable by the worker and distorts the sequential coordinator's cap and skip-set. Adding `needs-decision` keeps upstream's vocabulary intact and makes the human-gate role a clean addition rather than a redefinition.

### User-story traceability (optional)

When `user_story_traceability` is `auto` and the source or parent issue has a user-stories section (a `User stories` heading or `US-<n>` items), the quiz adds a **Stories covered** line per slice and a closing **Uncovered stories** line, and each issue body gets a `Covers:` line. Without stories nothing changes. Traceability is presentation and body text only; it never creates edges or labels.

### Native GitHub relationships

- Every published slice is a native **sub-issue** of the parent; every blocking edge is a native **blocked-by** relationship. This graph is authoritative for both board pairs, which never parse body text. Upstream's `## Blocked by` / `## Parent` body sections are informational; the catalog neither removes nor reads them.
- **Transport order** is `tracker.dependencies.transport: auto` from the capability-configuration spec: `gh issue create --parent/--blocked-by` and `gh issue edit --add-sub-issue/--add-blocked-by`; then the REST sub-issues and `dependencies/blocked_by` endpoints (the upstream adapter's recipe); then GraphQL `addBlockedBy`/`addSubIssue`; then the body convention only when every native transport fails, reported as a downgrade. Native edges are never dual-written to the body.
- **Post-publication verification** (the source project's single-source-of-truth intent, kept as a check rather than a rewrite of the provider): after `/to-tickets` returns, the human-gate worker lists the parent's sub-issues and each child's blocked-by set, compares them with the approved breakdown, repairs gaps with `gh issue edit --add-sub-issue` / `--add-blocked-by`, and records repairs in its receipt. This mitigates upstream's #513/#554 failure modes without forking.

### The overlay (clean-room wording)

Emitted verbatim into the conversation, with role names replaced by the repository's configured label strings, immediately before the user runs `/to-tickets #<parent>`:

```text
Decomposition policy for this repository:
- Classify every ticket as agent-doable (default), human-decision, or human-execution.
  human-decision = a person must decide or specify before the gated work can start.
  human-execution = a person must do the work (access, provisioning, sign-up, manual checks).
- In the review list, add a "Disposition" line to each ticket and ask me to confirm each non-default one.
- On publish, label agent-doable tickets "<ready-for-agent>", human-decision tickets "<needs-decision>",
  human-execution tickets "<ready-for-human>". Apply exactly one of these per ticket. Do not label
  human tickets "<ready-for-agent>".
- Make every ticket a native sub-issue of #<parent> and record blocking edges as native blocked-by
  relationships (gh issue create --parent / --blocked-by). Body text is not a substitute.
- [If the source has user stories] Add "Stories covered" per ticket and an "Uncovered stories" line.
Do not close or modify #<parent>.
```

## Attribution obligations

ADR 0004 settled both `to-issues` branches; this ADR picks composition.

| Shape | Upstream text copied? | Obligation (ADR 0004) | `THIRD_PARTY_NOTICES.md` |
| --- | --- | --- | --- |
| (a) Composition (primary) | None; the policy file and overlay are written from requirements without consulting the upstream file, and role names are short phrases, not protectable expression. | No license notice. Dependency disclosure only: a `## External skills` row in `README.md` for `to-tickets` and `setup-matt-pocock-skills` (needed by `publish-findings` and `pick-up-human-task`, optional, install command above, MIT, not bundled), mirrored in those skills' `compatibility` field; `metadata.provenance: original`. | No row. |
| (b) Thin extension (fallback) | Only if it retains upstream prose, quiz wording, rule blocks, or the issue template. | Derivative: `skills/<name>/NOTICE.md` with Matt Pocock's copyright line and the verbatim MIT text, `metadata.provenance: derivative`, `metadata.third-party-notices: NOTICE.md`. Clean-room: same as (a). | Register row pinned to the upstream commit the extension was compared against, plus the verbatim notice. |
| (c) Upstream contribution | Contributed text is licensed under upstream's MIT on merge. | Contribute only maintainer-owned wording; no source-project identity in the PR. | No row. |

The source project's `to-issues` file is never copied in any shape, so its MIT-derivative status creates no catalog obligation; disposition routing is a method, not expression. The vendoring history is recorded in `docs/provenance.md` as history, not as a notice.

## Retiring `to-issues`

- No `skills/to-issues/` directory, alias, or redirect is ever published. The name appears in the public repository only in `docs/research/`, the ADRs, and the one-sentence history note in `docs/provenance.md` (ADR 0004).
- Catalog prose says "ticket decomposition" for the capability and "`to-tickets`" only when naming the provider.
- ADR 0004's release-hygiene scan is extended with the term `to-issues` for `skills/**` and `README.md`. The source project's local copy is outside this catalog.

## Alternatives considered

- **Publish an attributed derivative of the source project's `to-issues`.** Rejected by the overlap audit: duplicates a maintained skill while dropping prefactoring, context-window sizing, the wide-refactor exception, local-tracker mode, and native sub-issues, for a copy whose only original behavior is a routing policy.
- **Fork `to-tickets` into the catalog with dispositions built in.** Same objections, plus drift from upstream's adapter files and two tracker configurations to keep in sync.
- **Make `ticket-decomposition` required for `publish-findings`.** Rejected: ADR 0002 keeps the publisher complete for simple actions; an external requirement would break independent installability.
- **Keep one `ready-for-human` label with a body marker for decisions.** Rejected: frontier computation never parses body text, and the human-gate pair needs a label-level filter.
- **Let the publisher decompose "moderately complex" actions itself.** Rejected: recreates the duplicated workflow and blurs the simple/complex boundary the receipt outcomes depend on.
- **Rely on upstream `wayfinder` for HITL/AFK.** Rejected: it classifies decision-map tickets, not implementation slices, and its label recording (#815) is unmerged.

## Consequences

- **#8 (capability configuration spec):** already carries these semantics as `tracker.labels.{ready_for_agent,needs_decision,ready_for_human}`, `tracker.dependencies.transport`, `capabilities.ticket_decomposition.{provider,user_story_traceability}`, the sibling-skill discovery order, and the Tier 3 decomposition-handoff recheck. This ADR uses those names; any later divergence is resolved in the spec's favor.
- **#18 (findings publication):** implement the complex-action path as specified: preflight, parent proposal issue with `needs-decision`, outcomes `handed-off` (parent URL) and `deferred` (reason), idempotent parent reuse on retry, no single-issue fallback. Ship `references/disposition-policy.md` and the overlay renderer.
- **#10 (human-gate wave):** `pick-up-human-task` filters on `needs-decision`, gains the "decompose into tickets" path (emit overlay, prompt `/to-tickets #<parent>`, verify and repair native edges) and results `decomposed` and `deferred`; `ready-for-human` stays out of both human-gate frontiers. Ships the same generated policy file.
- **#9 (work-execution wave):** unchanged; `ready-for-agent` is the only eligible role.
- **ADR 0004 (#14):** already specifies both branches; composition is chosen, so no `NOTICE.md` or register row is needed unless the fallback is triggered.
- **#15 (validation and pilot):** scenarios for provider-absent deferral, `manual` gating, overlay effectiveness per runtime (the switching metric), label agreement across the two configuration files, and native-edge verification and repair; record the upstream revision each release was tested against; add `to-issues` to the hygiene scan.
- **#17 (documentation):** the README `## External skills` table lists `to-tickets` and `setup-matt-pocock-skills` with `publish-findings` and `pick-up-human-task` as consumers; the README explains the provider install command and the two-file configuration relationship; `docs/provenance.md` carries the history note.

## Open items

1. Maintainer ratifies the `needs-decision` role name and the `provider: manual` opt-in (alternatives: `needs-spec`, or no manual mode).
2. Maintainer ratifies whether to open the upstream proposal (c) now; if so, it should reference #815 and offer dispositions as an opt-in, roles supplied by `triage-labels.md`.
3. The minimum `gh` release carrying `--parent`/`--blocked-by` was not verifiable from this session; #8 records it as a preflight version check once confirmed.
4. Confirm in the pilot that Codex honors `allow_implicit_invocation: false` as Claude Code honors `disable-model-invocation`; if a runtime allows model invocation, #10 may add a runtime note letting the worker invoke `to-tickets` directly without changing this contract.
5. Whether the overlay should ask `to-tickets` to omit the informational `## Blocked by` body section. Deferred: harmless while non-authoritative.

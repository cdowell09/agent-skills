# Findings publication contract

Date: 2026-09-02
Question: How should a repository-agnostic findings-publication skill accept any conforming structured Markdown findings artifact, deduplicate its proposed actions, apply repository-configured approval policy, file simple actions as GitHub issues, hand complex actions to a configured ticket-decomposition capability when available, and write durable publication receipts?
Resolves: [#18 Specify generic findings publication](https://github.com/cdowell09/agent-skills/issues/18)
Inputs: [ADR 0001](../adr/0001-npx-skills-distribution-contract.md), [ADR 0002](../adr/0002-skill-boundaries-and-shared-contracts.md), [ADR 0003](../adr/0003-ticket-decomposition-provider.md), [ADR 0004](../adr/0004-licensing-attribution-and-contribution.md), [Capability configuration spec](capability-configuration.md), [source skill inventory](../research/2026-07-12-source-skill-inventory.md), [`to-issues` overlap audit](../research/2026-07-12-to-issues-overlap.md)

Skill names used here (`pick-up-work`, `work-the-board`, `pick-up-human-task`, `work-the-human-board`, `dogfood`, `research-project`, `publish-findings`) are working names; a later naming pass may rename them.

## Decision

- `publish-findings` is the single filing path for every findings producer. Its input is one **Findings Artifact** (contract `findings` v1): Markdown with YAML frontmatter, a stable `artifact_id`, finding IDs `F-<short>-<n>`, action IDs `A-<short>-<n>`, an integer `revision` that must change whenever any Proposed Action's content changes, and a per-finding dedupe fingerprint the publisher recomputes.
- Its output is one append-only **publication receipt** (contract `publication` v1) at `.agent-skills/state/publications/<artifact_id>.yaml`, one row per attempt per action, outcomes `filed | handed-off | duplicate | rejected | deferred | failed`. A row is written after each GitHub mutation and before the next, so a crash never loses a filed issue and a re-run touches only actions without a terminal row.
- A non-conforming artifact stops before any side effect. Deduplication has three layers: receipts, exact HTML markers in open and closed issue bodies, then fuzzy title search that never decides alone. Approval binds to `(artifact_id, revision, action_hash)` and is void when any of them changes. Approval mode is `findings.approval.mode` exactly as the capability spec defines it: no catalog default, and `never` refuses filing explicitly.
- Simple actions become one issue each with role-mapped labels and exactly one work-disposition label. Complex actions follow ADR 0003: one parent proposal issue labelled `needs_decision`, outcome `handed-off`; with no provider, outcome `deferred`, reason `decomposition-unavailable`, proposal untouched. No umbrella issue per artifact by default.
- Producers hand off through sibling-skill discovery, pass the artifact path, keep their own result, and report publication on a separate line. The publisher accepts any conforming artifact standalone and never commits.

## Purpose and boundary

`publish-findings` turns approved Proposed Actions into GitHub issues and durable receipts. It does not discover findings, judge severity, rewrite proposals, decompose work, commit, or open pull requests. It reads the artifact and `.agent-skills/config.yaml`; mutates issues and, optionally, Project items and milestones; and writes the receipt, the artifact's action `status` fields and `## Publication` section, and, when the frontmatter names one, a line in the producer's ledger.

The source is the filing overlay duplicated in the `dogfood` and `pipeline-research` source skills plus the `to-issues` handoff.

| Source behavior | Disposition |
| --- | --- |
| Dedupe against open and closed issues, ask approval, file selected findings with labels | **Preserved**, with exact-marker matching and an explicit approval binding added |
| Report is durable; issues point at it | **Preserved**: the artifact is committed under `findings.artifact_dir`; every issue links to it |
| Improvements go to `to-issues`; bugs file directly | **Preserved as `kind: simple \| complex`** chosen by the producer; complex goes to the `ticket-decomposition` capability (ADR 0003) |
| Fixed labels, milestone, Project, Theme field, repository coordinates, severity scale | **Configured**: `tracker.labels.*`, `tracker.milestones`, `tracker.project`, `tracker.fields.theme`, `findings.*` |
| Report PR auto-merge; committing the report; two copies of the filing rules | **Removed**: the publisher never commits; producers own their docs PR; one skill carries the rules |
| Overloaded `ready-for-human` | **Delegated** to ADR 0003's disposition policy, generated into `references/disposition-policy.md` |

## Findings Artifact contract (`findings` v1)

A UTF-8 Markdown file under `findings.artifact_dir` (default `docs/findings`) named `<artifact_id>.md`. Producers create it; humans may edit it; the publisher edits only action `status` fields and `## Publication`.

### Frontmatter

| Key | Required | Meaning |
| --- | --- | --- |
| `artifact_version` | yes | Contract major; `1` |
| `artifact_id` | yes | `<producer>-<YYYYMMDD>-<slug>`: producer skill name, UTC creation date, kebab-case slug of at most 32 characters (journey, research area, or topic). Assigned once, never changed; on collision with an existing artifact or receipt the producer appends `-2`, `-3`, and so on |
| `artifact_short_id` | yes | First 8 hex of `sha256(artifact_id)`, stored so IDs are greppable; the publisher verifies it |
| `producer` | yes | `{ skill, version }` |
| `created_at` | yes | RFC 3339 UTC |
| `subject` | yes | Stable identifier of what was examined, reused verbatim on later runs against the same target: `journey:<id>` or `url:<origin>` for dogfooding, `area:<id>` for research. Free text allowed but must be stable |
| `revision` | yes | Integer from 1; incremented on any edit to an action's `title`, `body`, `kind`, `disposition`, `labels`, or `findings`. Edits to `status`, finding prose, or `## Publication` do not bump it |
| `ledger` | no | Repository-relative path of a producer ledger the publisher appends to |
| `severity_scale` | no | Per-artifact override of `findings.severity_scale` |

### Body

Exactly these H2 sections in this order, all present even when empty:

1. `## Summary` — free Markdown.
2. `## Findings` — `### F-<artifact_short_id>-<n>: <title>` blocks, `n` dense from 1. Each begins with a field list, then free prose:
   - `- severity:` a value from the effective scale (default `critical | high | medium | low`)
   - `- disposition:` `bug | improvement | question | note | wontfix`
   - `- fingerprint:` `sha256:<first 16 hex>` (below)
   - `- evidence:` one or more sub-items: repository-relative paths (screenshots, console or network captures, briefs) or absolute `https://` URLs (sources); `none` only for `note`
   - `- subject:` optional per-finding override; other `- key: value` lines are producer extensions and are ignored
3. `## Proposed Actions` — `### A-<artifact_short_id>-<n>: <title>` blocks, `n` dense from 1. Field list, blank line, then the body as free Markdown up to the next heading:
   - `- findings:` one or more finding IDs from this artifact
   - `- kind:` `simple | complex`. Simple is one issue an implementer can act on as written; complex needs slicing into several tickets. The producer decides; the publisher never reclassifies
   - `- disposition:` `agent-doable | human-decision | human-execution` (ADR 0003), required; complex actions must be `human-decision`
   - `- labels:` zero or more **label roles** from `tracker.labels` (`bug`, `improvement`, `finding`, ...). Disposition roles are added by the publisher and are an error here
   - `- status:` `proposed | approved | rejected | filed | handed-off | deferred | duplicate`; producers write `proposed`
4. `## Publication` — append-only lines written by the publisher; producers create it empty.

### Fingerprint and hashes

- **Finding fingerprint** = first 16 hex of `sha256(normalize(title) + "\n" + subject + "\n" + disposition)`; `normalize` lowercases, strips a leading `bug:`/`fix:`-style prefix and `#<n>` references, removes punctuation, collapses whitespace. Stable across artifacts, so the same defect found on two days matches.
- **Action hash** = `sha256` of the canonical action (LF endings, trailing whitespace stripped, fields in the order `title, kind, disposition, labels, findings, body`). Computed by the publisher and stored in the receipt only.
- **Actions digest** = `sha256` of the sorted action hashes; stored in the receipt to catch content changes without a `revision` bump.

### Example

````markdown
---
artifact_version: 1
artifact_id: dogfood-20260902-onboarding
artifact_short_id: 3f9a1c2e
producer: { skill: dogfood, version: "0.1.0" }
created_at: 2026-09-02T14:05:12Z
subject: journey:onboarding
revision: 2
ledger: docs/findings/LEDGER.md
---

## Summary

Walked the onboarding journey on the staging target as a first-time user. Two defects, one improvement.

## Findings

### F-3f9a1c2e-1: Welcome page shows a blank name when the profile has no display name
- severity: high
- disposition: bug
- fingerprint: sha256:9f2c7b1e44a0d3c8
- evidence:
  - docs/evidence/dogfood-20260902-onboarding/welcome-blank-name.png
  - docs/evidence/dogfood-20260902-onboarding/console-welcome.txt

The greeting renders as "Hi, !" when `display_name` is null; the template interpolates an empty string.

### F-3f9a1c2e-2: Sign-up form accepts an email with trailing whitespace and then rejects it on submit
- severity: medium
- disposition: bug
- fingerprint: sha256:0b77e2a91c5d4f10
- evidence:
  - docs/evidence/dogfood-20260902-onboarding/network-signup.har

### F-3f9a1c2e-3: No way to resume onboarding after closing the tab
- severity: medium
- disposition: improvement
- fingerprint: sha256:6d1a9c3e2b80f4a7
- evidence:
  - docs/evidence/dogfood-20260902-onboarding/resume-missing.png

Returning users land on an empty dashboard with no pointer back to the unfinished steps.

## Proposed Actions

### A-3f9a1c2e-1: Fall back to the account email when display_name is null on the welcome page
- findings: [F-3f9a1c2e-1]
- kind: simple
- disposition: agent-doable
- labels: [bug, finding]
- status: proposed

Render `display_name`, else the local part of the email, else "there". Add a unit test for the null case.

### A-3f9a1c2e-2: Trim email input before validation
- findings: [F-3f9a1c2e-2]
- kind: simple
- disposition: agent-doable
- labels: [bug, finding]
- status: proposed

Trim on blur and before submit; server-side validation unchanged.

### A-3f9a1c2e-3: Resumable onboarding
- findings: [F-3f9a1c2e-3]
- kind: complex
- disposition: human-decision
- labels: [improvement, finding]
- status: proposed

Persist onboarding progress per account and surface a "Continue setup" entry point. Needs a product decision on whether progress expires, then slicing across storage, API, and UI.

## Publication
````

## Validation

The first step; no side effects. An artifact is **conforming** when no `E-*` fires; warnings never block.

| Code | Condition |
| --- | --- |
| `E-FRONTMATTER`, `E-VERSION-UNSUPPORTED`, `E-FIELD-MISSING <key>`, `E-FIELD-TYPE <key>` | Frontmatter absent or invalid; unsupported `artifact_version`; required key absent or mistyped |
| `E-ARTIFACT-ID`, `E-SHORT-ID` | `artifact_id` pattern or `producer.skill` mismatch; short id not derived from `artifact_id` |
| `E-SECTION-MISSING <name>`, `E-SECTION-ORDER` | One of the four H2 sections absent, duplicated, or out of order |
| `E-ID-MALFORMED <id>`, `E-ID-GAP`, `E-ID-DUPLICATE <id>` | Heading pattern, dense numbering, or uniqueness violated |
| `E-FINDING-FIELD <id> <field>`, `E-VOCAB <id> <field> <value>` | Required finding field missing; value outside a vocabulary |
| `E-FINGERPRINT <id>` | Stored fingerprint differs from the recomputed one |
| `E-ACTION-FINDING <id>`, `E-ACTION-KIND <id>`, `E-LABEL-ROLE <id> <role>` | Unknown finding reference; complex action not `human-decision`; unknown label role or a disposition role under `labels` |
| `E-STATUS-OWNER <id>` | A publisher-owned status (for example a hand-edited `filed`) with no receipt row behind it |
| `E-REVISION-STALE`, `E-REVISION-REGRESSED` | Actions digest changed while `revision` did not; `revision` lower than the receipt's `last_seen` |
| `W-UNKNOWN-FIELD`, `W-EVIDENCE-UNCOMMITTED <path>`, `W-NO-ACTIONS` | Extension field ignored; evidence path not in `git ls-files`; nothing to publish |

On any error the publisher prints one block listing every error and the artifact path, states that nothing was mutated, writes no receipt row, edits nothing, and returns `outcome: invalid-artifact`. Producers run the same validation before handing off, so a producer-side failure is a producer bug.

## Deduplication

Per action, after validation and before approval; the first exact hit decides.

1. **Receipts.** Every file under `state/publications/` is scanned for rows sharing any finding ID or fingerprint with this action whose effective outcome is `filed` or `handed-off`. Hit: `duplicate`, `matched: { source: receipt, issue_url }`.
2. **Exact markers.** Every issue the publisher creates ends with:

   ```html
   <!-- agent-skills:finding F-3f9a1c2e-1 -->
   <!-- agent-skills:fingerprint sha256:9f2c7b1e44a0d3c8 -->
   <!-- agent-skills:action dogfood-20260902-onboarding rev=2 A-3f9a1c2e-1 sha256:... -->
   ```

   The publisher runs `gh issue list --state all --search '"agent-skills:fingerprint sha256:9f2c7b1e44a0d3c8" in:body'` and the same per finding ID, restricted by `findings.dedupe.scope` and, for closed issues, to those closed within `closed_window_days`. Hit: `duplicate` with the URL and state (`closed_reason` for closed). An open hit on the **action** marker with this `artifact_id` is a **recovered filing** from a run that crashed after creating the issue: the row is `filed` with `recovered: true`.
3. **Fuzzy titles.** `gh issue list --state all --search '<normalized title words> in:title'` under the same scope and window, keeping candidates sharing at least 80 percent of normalized tokens. Advisory only: interactive mode shows them and the user picks `duplicate of <url>` or `proceed`; unattended or pre-approved runs record `deferred`, reason `possible-duplicate`, with the candidate URLs for the next interactive run.

A closed exact match is a duplicate by default: a fixed defect that reappears is a regression that deserves a fresh finding, not a re-filed proposal. Interactive mode may override with **file as regression**; the new issue body carries `Regression of <url>` and the row records `regression_of`. Unattended runs never override.

## Approval policy

`findings.approval.mode` is required with no catalog default (capability spec); absent, the publisher stops with setup guidance.

| Mode | Behavior |
| --- | --- |
| `interactive` | Each non-duplicate action is shown (title, kind, disposition, labels, findings, evidence, fuzzy candidates) and the user chooses `approve`, `reject`, `defer`, or `duplicate of <url>`. Edits happen outside the publisher, bump `revision`, and restart the run. Unattended: every action `deferred`, reason `approval-required` |
| `pre-approved-severities` | Auto-approved when every covered finding's severity is in `auto_file_severities`, the kind is in `auto_file_kinds` (proposed key, default `[simple]`), and no fuzzy candidate exists. Otherwise interactive handling in a live session, `deferred` with `approval-required` when unattended |
| `never` | Every action `deferred`, reason `approval-mode-never`; the run entry is still recorded so the refusal is visible |

An approval is recorded in the receipt row as `approval: { mode, by, at, revision, action_hash }` and reused on a later run only when `artifact_id`, `revision`, and `action_hash` all still match; otherwise the action returns to `proposed`. `by` is the `gh` login for interactive approvals and `policy:pre-approved-severities` for automatic ones. A human approved specific words, not a slot, so no approval survives an edit.

## Filing

Actions are processed in ID order; each GitHub mutation is followed by a receipt row before the next.

**Simple actions.** One `gh issue create` per action: title = action title; body = action body, a `## Findings` section quoting each finding's title, severity, and prose, an `## Evidence` list, a `## Source` line linking the artifact at `https://github.com/<owner>/<repo>/blob/<default_branch>/<artifact path>`, then the markers. Evidence follows `evidence.ui.embed`: `auto` embeds images inline when `repository.visibility` is `public` and links only when private; `inline` and `link-only` force either. Links are default-branch blob URLs of the repository-relative path and resolve once the producer's docs PR merges. Labels: the action's roles, exactly one work-disposition role (`agent-doable` to `ready_for_agent`, `human-decision` to `needs_decision`, `human-execution` to `ready_for_human`), and `finding` when configured; roles map to strings through `tracker.labels`. Milestone applied when `tracker.milestones.policy` is `named`. When `tracker.project` is configured and `findings.filing.add_to_project` (proposed key, default `true`) holds, the issue is added with status `todo` and, when `tracker.fields.theme` is configured and the action carries a `- theme:` extension field, that value. Outcome `filed` with the URL.

**Complex actions** (ADR 0003), after the `ticket-decomposition` preflight:

- Provider `to-tickets` or `manual`: create, or on retry reuse via the action marker, one **parent proposal issue** carrying the title, body, finding IDs, evidence links, acceptance intent, and any `## User stories` section, labelled `needs_decision` plus the action's non-disposition roles and never `ready_for_agent`. Outcome `handed-off` with the parent URL. In a live session with provider `to-tickets` the publisher additionally prints the overlay from `references/disposition-policy.md` and the command `/to-tickets #<parent>` so the user can continue at once; it never invokes the provider or waits for it, and native-edge verification stays with `pick-up-human-task` (#10).
- Provider `none`, or preflight failed: outcome `deferred`, reason `decomposition-unavailable` (ADR 0003's "ticket-decomposition unavailable"), the install command in the message, the action untouched. Never filed as a single issue, never reclassified.

**Umbrella issue.** None by default: the committed artifact is the durable parent, every issue links to it, the receipt lists every issue, and an umbrella is an issue no board skill consumes and someone must close. `findings.filing.umbrella: on` (proposed key, default `off`) creates one first, labelled `finding` only, makes each filed issue its native sub-issue through `tracker.dependencies` transport order, and records `umbrella_url` in the receipt.

## Publication receipt (`publication` v1)

File `.agent-skills/state/publications/<artifact_id>.yaml`, committed. Append-only: `runs` and `rows` only grow; an action's effective state is its newest row. The header is written on first use, a run entry when a run starts, a row immediately after each mutation or decision.

```yaml
publication_version: 1
artifact_id: dogfood-20260902-onboarding
artifact_path: docs/findings/dogfood-20260902-onboarding.md
producer: { skill: dogfood, version: "0.1.0" }
last_seen: { revision: 2, actions_digest: "sha256:5c0e...9a12" }
umbrella_url: null
runs:
  - run: claude-code@buildbox/20260902T151002Z-7be31d40
    started_at: 2026-09-02T15:10:02Z
    finished_at: 2026-09-02T15:11:40Z
    mode: interactive
    revision: 2
    summary: { filed: 1, handed-off: 0, duplicate: 1, rejected: 0, deferred: 1, failed: 0 }
rows:
  - action: A-3f9a1c2e-1
    findings: [F-3f9a1c2e-1]
    fingerprints: [sha256:9f2c7b1e44a0d3c8]
    action_hash: "sha256:e4b1...77c0"
    revision: 2
    outcome: filed
    issue_url: https://github.com/acme/widgets/issues/301
    labels: [bug, finding, ready-for-agent]
    at: 2026-09-02T15:10:31Z
    run: claude-code@buildbox/20260902T151002Z-7be31d40
    approval: { mode: interactive, by: octocat, at: 2026-09-02T15:10:20Z, revision: 2, action_hash: "sha256:e4b1...77c0" }
  - action: A-3f9a1c2e-2
    findings: [F-3f9a1c2e-2]
    fingerprints: [sha256:0b77e2a91c5d4f10]
    action_hash: "sha256:19aa...03fe"
    revision: 2
    outcome: duplicate
    matched: { source: issue-marker, issue_url: https://github.com/acme/widgets/issues/288, state: closed, closed_reason: completed }
    at: 2026-09-02T15:10:35Z
    run: claude-code@buildbox/20260902T151002Z-7be31d40
  - action: A-3f9a1c2e-3
    findings: [F-3f9a1c2e-3]
    fingerprints: [sha256:6d1a9c3e2b80f4a7]
    action_hash: "sha256:b02d...4e61"
    revision: 2
    outcome: deferred
    reason: decomposition-unavailable
    detail: "capabilities.ticket_decomposition.provider is none"
    attempt: 1
    at: 2026-09-02T15:11:02Z
    run: claude-code@buildbox/20260902T151002Z-7be31d40
    approval: { mode: interactive, by: octocat, at: 2026-09-02T15:10:58Z, revision: 2, action_hash: "sha256:b02d...4e61" }
```

Always present: `action`, `findings`, `fingerprints`, `action_hash`, `revision`, `outcome`, `at`, `run`. Conditional: `issue_url` (`filed`, `handed-off`); `matched` (`duplicate`); `reason` and optional `detail` (`rejected`, `deferred`, `failed`); `approval` when one was given; `attempt` for `failed` and retryable `deferred`; `recovered`, `regression_of`, `labels`, `project_item`, `milestone` as applicable. Run owner ids use the capability spec's claim-owner format. Reasons: `approval-required`, `approval-mode-never`, `possible-duplicate`, `decomposition-unavailable`, `label-missing`, `project-unavailable`, `gh-error`, `rate-limited`, `retry-exhausted`, `run-aborted`, `user-rejected`.

**Artifact and ledger updates.** After each row the action's `status` becomes the outcome (`failed` leaves `approved`). At run end one line is appended to `## Publication`:

```markdown
- 2026-09-02T15:11:40Z run claude-code@buildbox/20260902T151002Z-7be31d40 revision 2: 1 filed, 1 duplicate, 1 deferred -> .agent-skills/state/publications/dogfood-20260902-onboarding.yaml
```

When `ledger` is set, one line per terminal row (`filed`, `handed-off`, `duplicate`, `rejected`) is appended under a `## Publications` heading, created at the end of the file if absent: `- 2026-09-02 dogfood-20260902-onboarding A-3f9a1c2e-1 filed https://github.com/acme/widgets/issues/301`. Nothing else in the ledger is touched; #12 owns its format and keeps that heading append-only. The publisher never commits; its result names the paths to commit, which producers include in their docs PR.

## Idempotency and recovery

- **Re-run on the same artifact and revision**: validate, load the receipt, skip every action whose newest row is `filed`, `handed-off`, `duplicate`, or `rejected`; process the rest (`deferred` other than `approval-mode-never`, `failed`, or no row), reusing a stored approval when its binding still matches.
- **Write order per action**: dedupe, decide approval, create the issue or parent, append the row, then Project and milestone placement, then the artifact status. A crash after creation and before the row is caught by the action-marker search (`recovered: true`). A placement failure never undoes a filed issue: it appends `failed`, reason `project-unavailable`, with the URL, and the next run completes placement without creating anything.
- **Retries**: a transient `gh` error (network, 5xx, secondary rate limit) is retried once after 30 seconds; a second failure writes `failed` with `attempt` incremented and moves on. Authentication, permission, and primary rate-limit errors abort the run: remaining approved actions get `deferred`, reason `run-aborted`. After three `failed` attempts the row carries `reason: retry-exhausted` and the action is skipped until a human edits the artifact (bumping `revision`) or clears the flag in the receipt.
- **Preserving unfiled proposals** means: the action block is never rewritten, reordered, or removed; its `status` is `proposed`, `approved`, or `deferred`; the receipt has a row with a reason, or no row; the ledger has no line; and the next run picks it up once the reported cause is fixed. The artifact is the source of truth for what was proposed, the receipt for what happened.

## Capability preflight

Runs after validation and before any mutation, using the capability spec's tiers (fast path on a matching validation fingerprint; Tier 3 volatile rechecks otherwise, including its "Findings filing" and "Decomposition handoff" rows). All checks finish before the first side effect; a required failure stops with the spec's stop-with-guidance block and `outcome: setup-required` or `preflight-failed`.

| Check | Required | On failure |
| --- | --- | --- |
| `config.yaml` present with supported `version`, `findings.approval.mode`, `tracker.labels.bug` and `improvement` (and `needs_decision` when the provider is not `none`) | yes | stop, setup guidance |
| `gh auth status` on the receipt's host; login equals the validation receipt's; scopes include `repo` (plus `project` when placement is on); repository permission at least `triage` | yes | stop |
| Every label string the run will apply exists | yes | `findings.filing.create_missing_labels: false` (proposed key, default): stop naming the labels and `gh label create` commands; `true`: create them with a neutral description and record it in the run entry |
| `state/publications/` and the artifact directory writable; `gh issue list` probe succeeds | yes | stop; dedupe is never skipped |
| Project and status field resolve, when placement is on; milestone exists, when policy is `named` | optional | placement or milestone disabled for the run, downgrade reported |
| `ticket-decomposition` per ADR 0003 (provider declared; sibling discovery finds `to-tickets`; upstream tracker config names this repository; the three roles exist and shared strings agree) | optional | complex actions `deferred`, `decomposition-unavailable` |

## Producer handoff contract

1. The producer finishes its own job: the artifact is written under `findings.artifact_dir` and validated with the same rules; its own result is complete before publication starts and cannot be changed by it.
2. It locates `publish-findings` with the capability spec's sibling-skill discovery order (runtime listing; `<repo>/.agents/skills/<name>/SKILL.md` and `<repo>/.claude/skills/<name>/SKILL.md`; the runtime's global skills directory; a Claude Code plugin), reading only the frontmatter.
3. If found and the runtime supports sequential interactive skill invocation, it invokes the publisher with one argument, the artifact path, plus `--unattended` when the producer itself runs unattended, and waits for the result line.
4. The producer's report ends with a separate line: `publication: not-attempted (not-installed)` with the install command; `publication: not-attempted (invocation-unavailable)` with the command for the user; `publication: succeeded (<receipt path>; <n> filed, <n> handed-off, <n> duplicate, <n> deferred)`; `publication: succeeded-with-failures (...)` when any row is `failed`; or `publication: failed (<outcome>: <reason>)` with outcome `invalid-artifact`, `setup-required`, `preflight-failed`, or `aborted`. Deferrals alone never make a run failed.
5. Standalone: `publish-findings <artifact path>` with any conforming artifact, including a human-written one; `producer.skill` is used only for the artifact-ID check.

Recommendation for #11 and #12: without the publisher, produce the artifact, print the install command, and stop; do not embed a second filing path. The capability spec's remark that producers "file directly when the publisher is absent" should be read as the reason `findings.approval.mode` is required for them, not as a mandate to duplicate dedupe and receipt logic (open item 1).

## Configuration keys consumed

```yaml
version: 1
repository: { visibility: private }        # discovered; drives evidence embedding
tracker:
  project: { owner: acme, number: 7 }      # optional; enables placement
  fields: { status: Status, theme: Theme }
  statuses: { todo: Todo }
  labels:
    bug: bug                               # required
    improvement: enhancement               # required
    finding: finding                       # optional marker
    ready_for_agent: ready-for-agent
    needs_decision: needs-decision         # required when ticket_decomposition.provider is not none
    ready_for_human: ready-for-human
  milestones: { policy: none, active: "" }
  dependencies: { transport: auto, sub_issues: native }   # umbrella sub-issues only
evidence:
  ui: { embed: auto }
findings:
  artifact_dir: docs/findings
  approval:
    mode: interactive                      # REQUIRED: interactive | pre-approved-severities | never
    auto_file_severities: [critical, high]
    auto_file_kinds: [simple]              # PROPOSED for #8; default [simple]
  dedupe:
    scope: [receipts, open_issues, closed_issues]
    closed_window_days: 90
  severity_scale: [critical, high, medium, low]   # PROPOSED for #8; default shown
  filing:                                  # PROPOSED for #8; all optional
    create_missing_labels: false
    add_to_project: true                   # effective only with tracker.project
    umbrella: off                          # off | on
capabilities:
  ticket_decomposition: { provider: none, user_story_traceability: auto }
  interactive_skill_invocation: auto
```

Additions for #8 to absorb: `findings.approval.auto_file_kinds`, `findings.severity_scale`, `findings.filing.{create_missing_labels, add_to_project, umbrella}`. All optional with the defaults shown; none is mutation-affecting in the spec's sense, so none is asked during setup.

## Capabilities

| Capability | Required | Fallback |
| --- | --- | --- |
| GitHub issue read, search, create via `gh` (`repo` scope, permission at least `triage`) | yes | stop |
| Repository file writes (receipt, artifact, ledger) | yes | stop |
| GitHub Projects write | optional | placement skipped, downgrade reported |
| `ticket-decomposition` (ADR 0003) | optional | complex actions deferred |
| Sequential interactive skill invocation | optional | producers print the command; the overlay is printed as text |
| Live human in session | optional | interactive mode defers everything; the other modes run unattended |

Runtime notes: none. Both runtimes run `gh`; the only divergence, how a producer invokes a sibling skill, is covered by `capabilities.runtime_notes` in the capability spec.

## Rough `SKILL.md` outline

```yaml
---
name: publish-findings
description: File approved Proposed Actions from a conforming findings artifact as GitHub issues, deduplicating against receipts and existing issues, handing complex actions to a configured ticket-decomposition provider, and writing durable publication receipts.
license: MIT
compatibility: GitHub repository with Issues; gh CLI authenticated with repo scope; .agent-skills/config.yaml with findings.approval.mode. Optional: GitHub Projects; ticket-decomposition provider (upstream to-tickets, installed separately).
metadata:
  author: Christian Dowell
  version: "0.1.0"
  provenance: original
---
```

1. **When to use** — after `dogfood` or `research-project`, or standalone with any conforming artifact; never to discover or edit findings.
2. **Inputs** — one artifact path; `--unattended`; `.agent-skills/config.yaml`.
3. **Validate** — apply `references/findings-artifact.md`; any error stops with no side effects.
4. **Configure and preflight** — fast path or Tier 1/3 per `references/setup.md`; run the preflight table; stop before any mutation on a required failure.
5. **Load the receipt** — read or initialize it; check `revision` against `last_seen`; append the run entry.
6. **Deduplicate** — receipts, markers, fuzzy candidates per `references/dedupe-rules.md`.
7. **Approve** — apply the mode per `references/approval-policy.md`; bind to `(artifact_id, revision, action_hash)`; defer everything in unattended interactive runs.
8. **File simple actions** — one issue each per `references/issue-template.md`; row after each issue; then placement.
9. **Hand off complex actions** — parent issue with `needs_decision`, `handed-off`; overlay from `references/disposition-policy.md` in a live session; `deferred` when unavailable.
10. **Update artifact and ledger** — statuses, `## Publication` line, ledger lines.
11. **Report** — result line with outcome, counts, receipt path, paths to commit, downgrades; never commit.
12. **Safety rules** — no side effect before validation and preflight pass; never reclassify kind; never apply `ready_for_agent` to a non-agent action; never rewrite a proposal; never re-file an action with a terminal row.

`references/`: `findings-artifact.md` (schema, example, errors), `publication-receipt.md` (schema, reasons, append-only rules), `dedupe-rules.md` (normalization, fingerprint, markers, searches, windows), `approval-policy.md` (modes, binding, unattended behavior), `issue-template.md` (issue and parent bodies, evidence rule, marker placement), `disposition-policy.md` (generated from ADR 0003's canonical source, including the overlay; drift-tested), `protocol-contracts.md` (generated: produces and accepts `findings` 1, `publication` 1, `config` 1), `setup.md` (generated). No `NOTICE.md`; nothing upstream is copied. `to-tickets` appears in the README `## External skills` table as optional (ADR 0004).

## Acceptance scenarios

1. **Given** an artifact missing `## Publication`, **when** the publisher runs, **then** it prints `E-SECTION-MISSING Publication`, writes no row, creates no issue, and returns `invalid-artifact`.
2. **Given** an action body edited while `revision` still equals the receipt's `last_seen` with a different digest, **when** it runs, **then** it stops with `E-REVISION-STALE` and nothing is mutated.
3. **Given** an action approved at revision 2 with a `failed` row, **when** its title is edited and `revision` becomes 3, **then** the stored approval is ignored and interactive mode asks again.
4. **Given** issue #288 closed as completed 30 days ago carrying `agent-skills:fingerprint sha256:0b77e2a91c5d4f10`, **when** an action with that fingerprint is processed under a 90-day window, **then** the row is `duplicate` with #288 and `state: closed`, and no issue is created; under `closed_window_days: 14` it is not found and proceeds to approval.
5. **Given** three approved simple actions where the second `gh issue create` fails twice with a 502, **when** the run continues, **then** the receipt holds `filed`, `failed attempt: 1`, `filed`; the artifact statuses match; a re-run files only the second.
6. **Given** a run that created issue #301 and crashed before writing the row, **when** re-run, **then** the action-marker search finds #301, the row is `filed` with `recovered: true`, and no second issue exists.
7. **Given** `ticket_decomposition.provider: none`, **when** a complex action is approved, **then** the row is `deferred` with `decomposition-unavailable`, the action block is unchanged, and no single issue is created.
8. **Given** `provider: to-tickets` with a passing preflight in a live session, **when** a complex action is approved, **then** one parent issue labelled `needs-decision` and not `ready-for-agent` exists, the row is `handed-off` with its URL, and the overlay plus `/to-tickets #<parent>` are printed.
9. **Given** `mode: interactive` and `--unattended`, **when** the publisher runs, **then** every non-duplicate action is `deferred` with `approval-required` and no issue is created.
10. **Given** `pre-approved-severities` with `auto_file_severities: [critical, high]` and `auto_file_kinds: [simple]`, **when** an unattended run sees a `high` simple, a `medium` simple, and a `high` complex action, **then** only the first is filed; the others are `deferred` with `approval-required`.
11. **Given** `mode: never`, **when** the publisher runs, **then** every action is `deferred` with `approval-mode-never` and the run entry is recorded.
12. **Given** the `finding` label absent and `create_missing_labels: false`, **when** preflight runs, **then** it stops before any issue is created, naming the label and the `gh label create` command.
13. **Given** `dogfood` finishing with the publisher not installed, **when** it reports, **then** its own outcome is unchanged and the last line is `publication: not-attempted (not-installed)` with the install command; with the publisher installed but the artifact failing validation, the last line is `publication: failed (invalid-artifact: ...)` and the dogfood result is still successful.
14. **Given** a conforming human-written artifact, **when** a user runs `publish-findings docs/findings/manual-20260902-billing.md`, **then** it is processed exactly like a producer artifact.
15. **Given** a fuzzy title candidate in an unattended pre-approved run, **when** processed, **then** the row is `deferred` with `possible-duplicate` and the candidate URLs, and an interactive re-run presents them.

## Risks

- Receipts land only when the producer's docs PR merges; two machines can file the same finding in that window. The open-issue marker layer closes most of it; the residual is a duplicate a human closes.
- Evidence links resolve only after the docs PR merges, so an issue opened first has dead links briefly. Mitigation: `W-EVIDENCE-UNCOMMITTED` and the "paths to commit" line.
- GitHub search indexes with delay and is rate-limited, so a just-created issue may be invisible to the marker search; receipts cover same-machine runs and the action marker covers the crash case.
- The 80 percent fuzzy threshold will miss paraphrases and flag near-misses; it is advisory by design, and #15 tunes it on pilot data.
- `revision` discipline depends on producers and humans; `E-REVISION-STALE` catches silent edits only after the fact.
- Every `human-decision` simple action becomes a `needs_decision` gate; producers that turn each question into an action will flood the human-gate frontier. `question` findings should mostly stay actionless notes.

## Open items

1. **#11, #12 decide** whether producers file directly when the publisher is absent. Recommendation: no. If they do, they must write this receipt format, and #8 should reword its rationale sentence.
2. **Maintainer ratifies**: no umbrella issue by default; closed exact matches are duplicates with an interactive regression override; complex actions never auto-approved by default; the default severity scale; the 80 percent fuzzy threshold; the three-attempt retry limit.
3. **#8 absorbs** the proposed keys and confirms `findings.severity_scale` may be overridden per artifact as specified here.
4. **#10** documents that a `needs_decision` issue bearing an `agent-skills:action` marker is a decomposition gate when its source action was `complex` and a plain decision gate otherwise; the parent body states which.
5. Whether `research-project` needs a second axis (impact or confidence); if so it is an extension field and never enters `auto_file_severities`.
6. A validator script under `scripts/` so producers and #15 run the schema checks deterministically; deferred to implementation.

## Consequences

- **#11 (`dogfood`)** and **#12 (`research-project`)**: produce `findings` v1 exactly as above (IDs, fingerprints, dense numbering, `revision: 1`, empty `## Publication`); set `kind` and `disposition` per action; validate before handoff; implement the discovery, invocation, and the separate `publication:` line; commit the artifact and receipt in their docs PR; keep `note` and `wontfix` findings actionless. #12 reserves an append-only `## Publications` heading in its ledger.
- **#10 (human-gate)**: consumes parent proposal issues labelled `needs_decision` with the action marker; emits the overlay and verifies native edges; may read `state/publications/` to link back to the artifact.
- **#9 (work-execution)**: unchanged; filed `ready_for_agent` issues enter the frontier like any other.
- **#8 (capability configuration)**: absorbs the proposed keys; the `findings` and `publication` majors are already fingerprint inputs.
- **#15 (validation and pilot)**: tests all fifteen scenarios, the append-only receipt invariant under crash injection, marker search across open and closed issues, drift of the generated references against their canonical sources, and allowlists this document's `example.com` and `acme/widgets` placeholders in the hygiene scan.

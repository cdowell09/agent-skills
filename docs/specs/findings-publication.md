# Findings publication contract

Date: 2026-09-02
Question: How should a repository-agnostic findings-publication skill accept any conforming structured Markdown findings artifact, deduplicate its proposed actions, apply repository-configured approval policy, file simple actions as GitHub issues, hand complex actions to a configured ticket-decomposition capability when available, and write durable publication receipts?
Resolves: [#18 Specify generic findings publication](https://github.com/cdowell09/agent-skills/issues/18)
Inputs: [ADR 0001](../adr/0001-npx-skills-distribution-contract.md), [ADR 0002](../adr/0002-skill-boundaries-and-shared-contracts.md), [ADR 0003](../adr/0003-ticket-decomposition-provider.md), [ADR 0004](../adr/0004-licensing-attribution-and-contribution.md), [Capability configuration spec](capability-configuration.md), [source skill inventory](../research/2026-07-12-source-skill-inventory.md), [`to-issues` overlap audit](../research/2026-07-12-to-issues-overlap.md)

Skill names used here (`pick-up-work`, `work-the-board`, `pick-up-human-task`, `work-the-human-board`, `dogfood`, `research-project`, `publish-findings`) are working names; a later naming pass may rename them.

## Decision

- `publish-findings` is the single filing path for every findings producer. Its input is one **Findings Artifact** (contract `findings` v1): a Markdown file with YAML frontmatter, a stable `artifact_id`, finding IDs `F-<short>-<n>`, action IDs `A-<short>-<n>`, an integer `revision` that must change whenever any Proposed Action's content changes, and a per-finding dedupe fingerprint the publisher can recompute.
- Its output is one append-only **publication receipt** (contract `publication` v1) at `.agent-skills/state/publications/<artifact_id>.yaml`, one row per attempt per action, with outcomes `filed | handed-off | duplicate | rejected | deferred | failed`. Rows are written after each GitHub mutation and before the next, so a crash never loses a filed issue and a re-run only touches actions without a terminal row.
- A non-conforming artifact stops before any side effect. Deduplication runs three layers: receipts, exact HTML markers in open and closed issue bodies, then fuzzy title search that never decides on its own. Approval is bound to `(artifact_id, revision, action_hash)` and is void when any of the three changes. Approval mode is `findings.approval.mode` exactly as the capability spec defines it; there is no catalog default and `never` refuses filing explicitly.
- Simple actions become one issue each with role-mapped labels and exactly one work-disposition label. Complex actions follow ADR 0003: one parent proposal issue labelled `needs_decision`, outcome `handed-off`; with no provider, outcome `deferred`, reason `decomposition-unavailable`, and the proposal stays in the artifact untouched. No umbrella issue per artifact by default.
- Producers hand off automatically through sibling-skill discovery, pass the artifact path, keep their own result, and report publication on a separate line. The publisher accepts any conforming artifact standalone.

## Purpose and boundary

`publish-findings` turns approved Proposed Actions into GitHub issues and durable receipts. It does not discover findings, judge severity, rewrite proposals, decompose complex work, commit, or open pull requests. It reads the artifact and the repository's capability configuration, mutates GitHub issues (and optionally Project items and milestones), and writes three things: the receipt, the artifact's `## Publication` section and action `status` fields, and, when the frontmatter names one, a line in the producer's ledger.

### What is preserved, removed, configured, or delegated

The source is the filing overlay duplicated in the `dogfood` and `pipeline-research` source skills (`references/filing.md` in each) plus the `to-issues` handoff.

| Source behavior | Disposition |
| --- | --- |
| Deduplicate against open and closed issues before filing; ask approval; file selected findings as issues with labels | **Preserved** as the generic pipeline, with exact-marker matching added and the approval binding made explicit |
| Report/brief is durable and issues point at it | **Preserved**: the artifact is committed under `findings.artifact_dir`; every issue links to it |
| Improvements delegate to `to-issues`; bugs file directly | **Preserved as `kind: simple \| complex`** chosen by the producer; complex goes to the `ticket-decomposition` capability (ADR 0003), never to a catalog decomposer |
| Fixed labels, milestone, Project, Theme field, repository coordinates | **Configured**: `tracker.labels.*` roles, `tracker.milestones`, `tracker.project`, `tracker.fields.theme` |
| Severity scale and filing thresholds fixed in prose | **Configured**: `findings.severity_scale`, `findings.approval.*` |
| Auto-merge of the report PR; committing the report | **Removed**: the publisher never commits; producers own their docs PR (#11, #12) |
| Two copies of the filing rules | **Removed**: one skill; producers carry no filing logic |
| Overloaded `ready-for-human` for decisions and execution | **Delegated** to ADR 0003's disposition policy, generated into `references/disposition-policy.md` |

## Findings Artifact contract (`findings` v1)

A UTF-8 Markdown file with YAML frontmatter, stored under `findings.artifact_dir` (default `docs/findings`) as `<artifact_id>.md`. Producers create it; humans may edit it; the publisher edits only the `status` field of actions and the `## Publication` section.

### Frontmatter

| Key | Required | Meaning |
| --- | --- | --- |
| `artifact_version` | yes | Integer contract major; `1` |
| `artifact_id` | yes | `<producer>-<YYYYMMDD>-<slug>`: producer skill name, creation date (UTC), kebab-case slug of at most 32 characters chosen by the producer (journey id, research area id, or topic). Assigned once at creation and never changed. On collision with an existing artifact or receipt the producer appends `-2`, `-3`, and so on |
| `artifact_short_id` | yes | First 8 hex characters of `sha256(artifact_id)`, recorded so finding and action IDs are greppable without recomputation. The publisher recomputes and verifies |
| `producer` | yes | `{ skill: <name>, version: <metadata.version> }` |
| `created_at` | yes | RFC 3339 UTC |
| `subject` | yes | Stable identifier of what was examined, reused verbatim on later runs against the same target: `journey:<id>` or `url:<origin>` for dogfooding, `area:<id>` for research. Free text is allowed but must be stable |
| `revision` | yes | Integer starting at 1; incremented on any edit to a Proposed Action's `title`, `body`, `kind`, `disposition`, `labels`, or `findings`. Edits to `status`, to `## Findings` prose, or to `## Publication` do not bump it |
| `ledger` | no | Repository-relative path of a producer ledger the publisher appends to |
| `severity_scale` | no | Vocabulary override for this artifact; defaults to `findings.severity_scale` |

### Body

Exactly these H2 sections, in this order, all present even when empty:

1. `## Summary` — free Markdown.
2. `## Findings` — zero or more `### F-<artifact_short_id>-<n>: <title>` blocks, `n` starting at 1 and dense. Each block begins with a field list, then free prose:
   - `- severity:` one value from the effective severity scale (default `critical | high | medium | low`)
   - `- disposition:` `bug | improvement | question | note | wontfix`
   - `- fingerprint:` `sha256:<first 16 hex>` (derivation below)
   - `- evidence:` one or more sub-items, each a repository-relative path (screenshot, console or network capture, brief) or an absolute `https://` URL (a source). `note` findings may use `none`
   - `- subject:` optional per-finding override of the frontmatter subject
   - other `- key: value` lines are producer extensions and are ignored
3. `## Proposed Actions` — zero or more `### A-<artifact_short_id>-<n>: <title>` blocks, `n` dense from 1. Field list, blank line, then the body as free Markdown up to the next heading:
   - `- findings:` one or more finding IDs from this artifact
   - `- kind:` `simple | complex`. Simple means one issue an implementer can act on as written; complex means the proposal needs slicing into several tickets. The producer decides; the publisher never reclassifies
   - `- disposition:` `agent-doable | human-decision | human-execution` (ADR 0003 vocabulary), required. Complex actions may only be `human-decision`
   - `- labels:` zero or more **label roles** from `tracker.labels` (for example `bug`, `improvement`, `finding`). The publisher adds the disposition role itself; listing `ready_for_agent`, `needs_decision`, or `ready_for_human` here is an error
   - `- status:` `proposed | approved | rejected | filed | handed-off | deferred | duplicate`. Producers write `proposed`; the publisher writes the rest
4. `## Publication` — append-only lines written by the publisher; producers create it empty.

### Fingerprint and hashes

- **Finding fingerprint** = `sha256(normalize(title) + "\n" + subject + "\n" + disposition)`, first 16 hex, where `normalize` lowercases, strips a leading `bug:`/`fix:`/`feat:`-style prefix and any `#<n>` references, removes punctuation, and collapses whitespace. It is stable across artifacts, so the same defect found on two different days matches.
- **Action hash** = `sha256` over the canonical form (LF line endings, trailing whitespace stripped, fields in the order `title, kind, disposition, labels, findings, body`). It is not stored in the artifact; the publisher computes it and stores it in the receipt.
- **Actions digest** = `sha256` over the sorted list of action hashes; stored in the receipt to detect a content change without a `revision` bump.

### Example artifact

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

The greeting renders as "Hi, !" when `display_name` is null. The console shows no error; the template interpolates an empty string.

### F-3f9a1c2e-2: Sign-up form accepts an email with trailing whitespace and then rejects it on submit
- severity: medium
- disposition: bug
- fingerprint: sha256:0b77e2a91c5d4f10
- evidence:
  - docs/evidence/dogfood-20260902-onboarding/signup-trailing-space.png
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

Trim on blur and before submit; keep server-side validation unchanged.

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

Validation is the first step and has no side effects. An artifact is **conforming** when every check below passes; warnings never block.

| Code | Condition |
| --- | --- |
| `E-FRONTMATTER` | Frontmatter missing or not valid YAML |
| `E-VERSION-UNSUPPORTED` | `artifact_version` absent, not an integer, or a major this skill does not accept |
| `E-FIELD-MISSING <key>` / `E-FIELD-TYPE <key>` | Required frontmatter key absent or of the wrong type |
| `E-ARTIFACT-ID` | `artifact_id` does not match `<producer>-<YYYYMMDD>-<slug>` or `producer.skill` disagrees with its first segment |
| `E-SHORT-ID` | `artifact_short_id` is not the first 8 hex of `sha256(artifact_id)` |
| `E-SECTION-MISSING <name>` / `E-SECTION-ORDER` | One of the four H2 sections is absent, duplicated, or out of order |
| `E-ID-MALFORMED <id>` / `E-ID-GAP` / `E-ID-DUPLICATE <id>` | A finding or action heading does not match its pattern, numbering is not dense from 1, or an ID repeats |
| `E-FINDING-FIELD <id> <field>` | Required finding field missing |
| `E-VOCAB <id> <field> <value>` | Value outside the severity scale, disposition, kind, work-disposition, or status vocabularies |
| `E-FINGERPRINT <id>` | Stored fingerprint differs from the recomputed one |
| `E-ACTION-FINDING <id>` | An action references a finding ID not in this artifact |
| `E-ACTION-KIND <id>` | A complex action with a disposition other than `human-decision` |
| `E-LABEL-ROLE <id> <role>` | A label role not present in `tracker.labels`, or a disposition role listed under `labels` |
| `E-STATUS-OWNER <id>` | Status is a publisher-owned value but no receipt row justifies it (a hand-edited `filed`) |
| `E-REVISION-STALE` | Actions digest differs from the receipt's last-seen digest while `revision` is unchanged |
| `E-REVISION-REGRESSED` | `revision` is lower than the receipt's last-seen revision |
| `W-UNKNOWN-FIELD` | Extension field ignored |
| `W-EVIDENCE-UNCOMMITTED <path>` | A repository-relative evidence path is not in `git ls-files` |
| `W-NO-ACTIONS` | No Proposed Actions; publication ends with an empty run row |

On any `E-*` the publisher stops, prints one block listing every error with the artifact path, states that nothing was mutated, and returns result `outcome: invalid-artifact` to its caller. It does not write a receipt row, does not edit the artifact, and does not touch GitHub. Producers must run the same validation before handing off, so a producer-side failure is a producer bug rather than a publisher outcome.

## Deduplication

Runs per action after validation, before approval, in this order; the first exact hit decides.

1. **Receipts.** Every file under `state/publications/` is scanned for rows whose `findings` share any ID with the action, or whose `fingerprints` share any value, and whose effective outcome is `filed` or `handed-off`. Hit: outcome `duplicate`, `matched: { source: receipt, issue_url }`.
2. **Exact markers in issues.** Every issue the publisher creates carries these HTML comments at the end of its body:

   ```html
   <!-- agent-skills:finding F-3f9a1c2e-1 -->
   <!-- agent-skills:fingerprint sha256:9f2c7b1e44a0d3c8 -->
   <!-- agent-skills:action dogfood-20260902-onboarding rev=2 A-3f9a1c2e-1 sha256:… -->
   ```

   The publisher searches `gh issue list --state all --search '"agent-skills:fingerprint sha256:9f2c7b1e44a0d3c8" in:body'` and the same for each finding ID, restricted by `findings.dedupe.scope` (`open_issues`, `closed_issues`) and, for closed issues, to those closed within `closed_window_days`. Hit: outcome `duplicate` with the URL and the issue state; a closed hit records `closed_reason`. An open hit on the *action* marker with this artifact's `artifact_id` is not a duplicate but a **recovered filing** (a previous run crashed after creating the issue): the row is written as `filed` with `recovered: true`.
3. **Fuzzy title search.** `gh issue list --state all --search '<normalized title words> in:title'`, limited to the same scope and window, keeping candidates whose normalized title shares at least 80 percent of tokens. This is a signal only: in interactive mode the candidates are shown and the user marks the action `duplicate` of one URL or `proceed`; in unattended or pre-approved runs a fuzzy hit yields `deferred`, reason `possible-duplicate`, with the candidate URLs recorded so the next interactive run can decide.

A closed exact match is a duplicate by default because a fixed defect that reappears is a regression that deserves a fresh finding, not a re-filed proposal. In interactive mode the user may override with **file as regression**; the new issue then carries `Regression of <url>` in its body and the receipt row records `regression_of`. Unattended runs never override.

## Approval policy

`findings.approval.mode` is required and has no catalog default (capability spec). The publisher stops with setup guidance when it is absent.

| Mode | Behavior |
| --- | --- |
| `interactive` | Each non-duplicate action is presented (title, kind, disposition, labels, findings, fuzzy candidates, evidence) and the user chooses `approve`, `reject`, `defer`, or `duplicate of <url>`. Editing happens outside the publisher; an edit bumps `revision` and the run is restarted. In an unattended run every action is `deferred`, reason `approval-required`, and the result says so |
| `pre-approved-severities` | An action is auto-approved when every finding it covers has a severity in `auto_file_severities`, its kind is in `auto_file_kinds` (proposed key, default `[simple]`), and no fuzzy candidate exists. Everything else is `deferred`, reason `approval-required`, in unattended runs, or falls through to interactive handling in a live session |
| `never` | Filing is refused: every action is `deferred`, reason `approval-mode-never`; the receipt still records the run so the deferral is visible |

Approval is bound to the tuple `(artifact_id, revision, action_hash)` and recorded in the receipt row as `approval: { mode, by, at, revision, action_hash }`. On a later run the binding is reused only when all three still match; otherwise the action returns to `proposed` and, in interactive mode, is asked again. `by` is the `gh` login for interactive approvals and `policy:pre-approved-severities` for automatic ones. Approval never survives an artifact edit because the human approved specific words, not a slot.

## Filing

Preconditions per run: validation passed, preflight passed, and at least one action is approved. The publisher processes actions in ID order and writes a receipt row after each GitHub mutation.

### Simple actions

One issue per action via `gh issue create`:

- **Title**: the action title.
- **Body**: the action body; a `## Findings` section quoting each finding's title, severity, and prose; an `## Evidence` list; a `## Source` line linking the artifact at `https://github.com/<owner>/<repo>/blob/<default_branch>/<artifact path>`; the marker comments. Evidence follows `evidence.ui.embed`: `auto` embeds images inline when `repository.visibility` is `public` and links only when private; `inline` and `link-only` force either. Links are default-branch blob URLs of the repository-relative path, which resolve once the producer's docs PR merges; uncommitted paths are warned about.
- **Labels**: the roles listed on the action, plus exactly one work-disposition role (`agent-doable` → `ready_for_agent`, `human-decision` → `needs_decision`, `human-execution` → `ready_for_human`), plus `finding` when configured and not already listed. Roles map to strings through `tracker.labels`.
- **Milestone**: applied when `tracker.milestones.policy` is `named`; none otherwise.
- **Project**: when `tracker.project` is configured and `findings.filing.add_to_project` (proposed key, default `true` when a Project is configured) holds, the issue is added with status `todo` and, when `tracker.fields.theme` is configured and the action carries `- theme:` as an extension field, that single-select value.

Outcome `filed` with the issue URL.

### Complex actions

Per ADR 0003, after the `ticket-decomposition` preflight:

- Provider `to-tickets` or `manual`: create (or on retry reuse, found through the action marker) one **parent proposal issue** carrying the action title and body, the finding IDs, evidence links, acceptance intent, any `## User stories` section from the body, labelled with `needs_decision` and any non-disposition roles from the action, never with `ready_for_agent`. Outcome `handed-off` with the parent URL. In a live session with provider `to-tickets`, the publisher then prints the overlay from `references/disposition-policy.md` and the exact command `/to-tickets #<parent>` for the user; it never invokes the provider, never waits for it, and leaves native-edge verification to `pick-up-human-task` (#10).
- Provider `none`, or preflight failed: outcome `deferred`, reason `decomposition-unavailable` (ADR 0003's "ticket-decomposition unavailable"), the install command in the message, the action left as-is. Never filed as a single issue, never reclassified.

### Umbrella issue

No umbrella issue per artifact by default. The committed artifact is the durable parent, every issue links to it, and the receipt lists every issue; an umbrella would add an issue that no board skill consumes and that someone must close. `findings.filing.umbrella: off | on` (proposed key, default `off`) enables one for repositories that want a tracker-side rollup; when on, it is created first, labelled with `finding` only, each filed issue becomes its native sub-issue through `tracker.dependencies` transport order, and it is recorded in the receipt as `umbrella_url`.

## Publication receipt (`publication` v1)

File: `.agent-skills/state/publications/<artifact_id>.yaml`, committed. Append-only: `runs` and `rows` only grow; the effective state of an action is its newest row. The publisher writes the header on first use, appends a run entry when a run starts, and appends a row immediately after each mutation or decision.

```yaml
publication_version: 1
artifact_id: dogfood-20260902-onboarding
artifact_path: docs/findings/dogfood-20260902-onboarding.md
producer: { skill: dogfood, version: "0.1.0" }
last_seen: { revision: 2, actions_digest: "sha256:5c0e…9a12" }
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
    action_hash: "sha256:e4b1…77c0"
    revision: 2
    outcome: filed
    issue_url: https://github.com/acme/widgets/issues/301
    at: 2026-09-02T15:10:31Z
    run: claude-code@buildbox/20260902T151002Z-7be31d40
    approval: { mode: interactive, by: octocat, at: 2026-09-02T15:10:20Z, revision: 2, action_hash: "sha256:e4b1…77c0" }
    labels: [bug, finding, ready-for-agent]
  - action: A-3f9a1c2e-2
    findings: [F-3f9a1c2e-2]
    fingerprints: [sha256:0b77e2a91c5d4f10]
    action_hash: "sha256:19aa…03fe"
    revision: 2
    outcome: duplicate
    matched: { source: issue-marker, issue_url: https://github.com/acme/widgets/issues/288, state: closed, closed_reason: completed }
    at: 2026-09-02T15:10:35Z
    run: claude-code@buildbox/20260902T151002Z-7be31d40
  - action: A-3f9a1c2e-3
    findings: [F-3f9a1c2e-3]
    fingerprints: [sha256:6d1a9c3e2b80f4a7]
    action_hash: "sha256:b02d…4e61"
    revision: 2
    outcome: deferred
    reason: decomposition-unavailable
    detail: "capabilities.ticket_decomposition.provider is none"
    at: 2026-09-02T15:11:02Z
    run: claude-code@buildbox/20260902T151002Z-7be31d40
    approval: { mode: interactive, by: octocat, at: 2026-09-02T15:10:58Z, revision: 2, action_hash: "sha256:b02d…4e61" }
    attempt: 1
```

Row fields: `action`, `findings`, `fingerprints`, `action_hash`, `revision`, `outcome`, `at`, `run` always; `issue_url` for `filed` and `handed-off`; `matched` for `duplicate`; `reason` plus optional `detail` for `rejected`, `deferred`, and `failed`; `approval` whenever an approval was given; `attempt` for `failed` and retryable `deferred`; `recovered`, `regression_of`, `labels`, `project_item`, `milestone` when applicable. The run owner id uses the capability spec's claim-owner format.

Reason vocabulary: `approval-required`, `approval-mode-never`, `possible-duplicate`, `decomposition-unavailable`, `label-missing`, `project-unavailable`, `gh-error`, `rate-limited`, `retry-exhausted`, `run-aborted`, `user-rejected`.

**Artifact updates.** After each row the publisher sets that action's `status` to the outcome (`filed`, `handed-off`, `duplicate`, `deferred`, `rejected`; `failed` leaves `approved`) and, at run end, appends one line to `## Publication`:

```markdown
- 2026-09-02T15:11:40Z run claude-code@buildbox/20260902T151002Z-7be31d40 revision 2: 1 filed, 1 duplicate, 1 deferred -> .agent-skills/state/publications/dogfood-20260902-onboarding.yaml
```

**Ledger updates.** When `ledger` is set, the publisher appends one line per terminal row (`filed`, `handed-off`, `duplicate`, `rejected`) under a `## Publications` heading, creating the heading at the end of the file if absent: `- 2026-09-02 dogfood-20260902-onboarding A-3f9a1c2e-1 filed https://github.com/acme/widgets/issues/301`. It never edits other ledger content; #12 owns the rest of the ledger format and must keep this heading append-only.

The publisher never commits. Its result names the three paths to commit; producers include them in their docs PR, and a standalone user commits them by hand.

## Idempotency and recovery

- **Re-run on the same artifact and revision** validates, loads the receipt, and skips every action whose newest row is `filed`, `handed-off`, `duplicate`, or `rejected`. Actions with `deferred` (reason other than `approval-mode-never`) or `failed` rows, or no row, are processed again; a stored approval binding is reused when it still matches.
- **Write order per action**: dedupe → decide approval → create the issue (or parent) → append the receipt row → set Project and milestone → update the artifact status. A crash after issue creation and before the row is caught by the action-marker search on the next run (`recovered: true`). A crash after the row and before Project placement is repaired on the next run by re-reading the issue and completing placement without creating anything. Project and milestone failures never undo a filed issue; they append `failed` with `reason: project-unavailable` and the issue URL so the next run only completes placement.
- **Retry limits**: a transient `gh` error (network, 5xx, secondary rate limit) is retried once within the run after a 30-second wait. A second failure writes `failed` with `attempt` incremented and moves to the next action. Authentication, permission, and primary rate-limit errors abort the run: the remaining approved actions receive `deferred`, reason `run-aborted`, and the result reports the cause. After three `failed` attempts an action is marked `reason: retry-exhausted` and skipped until a human resets it by editing the artifact (which bumps `revision`) or removing the flag from the receipt.
- **Preserving unfiled proposals means**: the action block in `## Proposed Actions` is never rewritten, reordered, or removed by the publisher; its `status` is `proposed`, `approved`, or `deferred`; the receipt carries a row with a reason or no row; the ledger has no line for it; and the next run picks it up with no human action beyond fixing the reported cause. The artifact remains the source of truth for what was proposed, the receipt for what happened.

## Capability preflight

Runs after validation and before any mutation, following the capability spec's tiers (fast path when the validation receipt fingerprint matches; Tier 3 volatile rechecks otherwise). All checks complete before the first side effect; any required failure stops with the spec's stop-with-guidance block and result `outcome: setup-required` or `outcome: preflight-failed`.

| Check | Required | On failure |
| --- | --- | --- |
| `config.yaml` present, `version` supported, `findings.approval.mode` set, `tracker.labels.bug` and `improvement` mapped | yes | stop, setup guidance |
| `gh auth status` for the receipt's host; login equals the validation receipt's login; scopes include `repo` (and `project` when a Project is used) | yes | stop |
| Repository permission at least `triage` (labels) | yes | stop |
| Every label string the run will apply exists | yes | `findings.filing.create_missing_labels: false` (proposed key, default `false`): stop naming the labels and the `gh label create` commands; `true`: create them with a neutral description and record the creation in the run entry |
| `state/publications/` and the artifact directory writable | yes | stop |
| Dedupe sources reachable (`gh issue list` probe) | yes | stop; dedupe is never skipped |
| Project exists and the status field resolves, when placement is on | optional | placement disabled for the run, downgrade reported |
| Milestone exists, when policy is `named` | optional | milestone omitted, downgrade reported |
| `ticket-decomposition` per ADR 0003 (provider declared; sibling discovery finds `to-tickets`; upstream tracker config names this repository; the three roles exist and shared strings agree) | optional | complex actions `deferred`, reason `decomposition-unavailable` |
| Sequential interactive skill invocation (for the live overlay hand-back) | optional | overlay printed as text for the user to paste |

## Producer handoff contract

1. The producer finishes its own job first: the artifact is written under `findings.artifact_dir`, validated with the same rules, and the producer's own result is complete. Publication cannot change that result.
2. The producer locates `publish-findings` with the capability spec's sibling-skill discovery order (runtime listing; `<repo>/.agents/skills/<name>/SKILL.md` and `<repo>/.claude/skills/<name>/SKILL.md`; the runtime's global skills directory; a Claude Code plugin). It reads nothing but the frontmatter.
3. If found and the runtime supports sequential interactive skill invocation, the producer invokes it with exactly one argument, the artifact path, plus `--unattended` when the producer itself runs unattended. It waits for the publisher's result line.
4. The producer's report ends with a separate line, in this vocabulary:
   - `publication: not-attempted (not-installed)` with the install command;
   - `publication: not-attempted (invocation-unavailable)` with the command for the user to run;
   - `publication: succeeded (<receipt path>; <n> filed, <n> handed-off, <n> duplicate, <n> deferred)`;
   - `publication: failed (<outcome>: <reason>)` where outcome is `invalid-artifact`, `setup-required`, `preflight-failed`, or `aborted`.
   `deferred` rows do not make a run `failed`; a run with any `failed` row reports `succeeded-with-failures` and the count.
5. Standalone invocation is `publish-findings <artifact path>`; the artifact may come from any producer, including a human-written one, as long as it conforms. The publisher never asks who produced it beyond `producer.skill` for the artifact-ID check.

Recommendation for #11 and #12: when the publisher is not installed, produce the artifact and stop; do not embed a second filing path. The capability spec currently says producers "file directly when the publisher is absent"; that sentence should be read as the reason `findings.approval.mode` is required for producers, not as a mandate to duplicate dedupe and receipt logic (open item 1).

## Configuration keys consumed

```yaml
version: 1
repository:
  visibility: private              # discovered; drives evidence embedding
tracker:
  project: { owner: acme, number: 7 }    # optional; enables placement
  fields: { status: Status, theme: Theme }
  statuses: { todo: Todo }
  labels:
    bug: bug                       # required
    improvement: enhancement       # required
    finding: finding               # optional marker
    ready_for_agent: ready-for-agent
    needs_decision: needs-decision # required when ticket_decomposition.provider is not none
    ready_for_human: ready-for-human
  milestones: { policy: none, active: "" }
  dependencies: { transport: auto, sub_issues: native }   # umbrella sub-issues only
evidence:
  ui: { embed: auto }
findings:
  artifact_dir: docs/findings
  approval:
    mode: interactive              # REQUIRED: interactive | pre-approved-severities | never
    auto_file_severities: [critical, high]
    auto_file_kinds: [simple]      # PROPOSED for #8: default [simple]; complex never auto-approved unless listed
  dedupe:
    scope: [receipts, open_issues, closed_issues]
    closed_window_days: 90
  severity_scale: [critical, high, medium, low]   # PROPOSED for #8: default shown
  filing:                          # PROPOSED for #8, all optional
    create_missing_labels: false
    add_to_project: true           # effective only with tracker.project
    umbrella: off                  # off | on
capabilities:
  ticket_decomposition: { provider: none, user_story_traceability: auto }
  interactive_skill_invocation: auto
```

Additions flagged for #8 to absorb: `findings.approval.auto_file_kinds`, `findings.severity_scale`, `findings.filing.{create_missing_labels, add_to_project, umbrella}`. All are optional with the defaults shown; none is mutation-affecting in the spec's sense, so none is asked during setup.

## Capabilities

| Capability | Required | Fallback |
| --- | --- | --- |
| GitHub issue read, search, and create via `gh` (`repo` scope, permission at least `triage`) | yes | none; stop |
| Repository file writes (receipt, artifact, ledger) | yes | none; stop |
| GitHub Projects write | optional | placement skipped, downgrade reported |
| `ticket-decomposition` (ADR 0003) | optional | complex actions deferred |
| Sequential interactive skill invocation | optional | producers print the command; the publisher prints the overlay as text |
| Interactive human in session | optional | interactive mode defers everything; pre-approved and never modes run unattended |

Runtime notes: none are genuinely different. Both runtimes run `gh`; the only divergence is how a producer invokes a sibling skill, which the capability spec's `capabilities.runtime_notes` already covers.

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

Headings and one line per step:

1. **When to use** — after `dogfood` or `research-project`, or standalone with any conforming artifact path; never to discover or edit findings.
2. **Inputs** — one artifact path; `--unattended` flag; `.agent-skills/config.yaml`.
3. **Validate the artifact** — apply `references/findings-artifact.md`; any error stops with the error block and no side effects.
4. **Load configuration and preflight** — fast path or Tier 1/3 per `references/setup.md`; run the preflight table; stop before any mutation on a required failure; record optional downgrades.
5. **Load the receipt** — read or initialize `state/publications/<artifact_id>.yaml`; check `revision` against `last_seen`; append the run entry.
6. **Deduplicate** — per action: receipts, exact markers, fuzzy candidates, per `references/dedupe-rules.md`; write `duplicate` rows.
7. **Approve** — apply `findings.approval.mode` per `references/approval-policy.md`; bind approvals to `(artifact_id, revision, action_hash)`; defer everything in unattended interactive runs.
8. **File simple actions** — one issue each with `references/issue-template.md`, role-mapped labels, markers, evidence rule; row after each issue; then Project and milestone.
9. **Hand off complex actions** — parent proposal issue with `needs_decision`, outcome `handed-off`; print the overlay from `references/disposition-policy.md` in a live session; `deferred` when unavailable.
10. **Update the artifact and ledger** — set action statuses, append the `## Publication` line and ledger lines.
11. **Report** — print the result line (outcome, counts, receipt path, paths to commit, downgrades); never commit.
12. **Safety rules** — no side effect before validation and preflight pass; never reclassify kind; never apply `ready_for_agent` to a non-agent action; never rewrite a proposal; never re-file an action with a terminal row.

`references/`:

- `findings-artifact.md` — the `findings` v1 schema, example, error vocabulary.
- `publication-receipt.md` — the `publication` v1 schema, row fields, reason vocabulary, append-only rules.
- `dedupe-rules.md` — normalization, fingerprint derivation, marker format, search commands, closed-window and fuzzy rules.
- `approval-policy.md` — modes, binding tuple, unattended behavior, override rules.
- `issue-template.md` — issue and parent-issue body layout, evidence embedding rule, marker placement.
- `disposition-policy.md` — generated from ADR 0003's canonical source, including the overlay; drift-tested.
- `protocol-contracts.md` — generated; majors produced and accepted (`findings` 1, `publication` 1, `config` 1).
- `setup.md` — generated setup and tiered-validation procedure from the capability spec.

No `NOTICE.md`; the skill copies no upstream text. `to-tickets` appears in the README `## External skills` table (ADR 0004) as optional.

## Acceptance scenarios

1. **Given** an artifact missing `## Publication`, **when** `publish-findings` runs, **then** it prints `E-SECTION-MISSING Publication`, writes no receipt row, creates no issue, and returns `invalid-artifact`.
2. **Given** an artifact whose action body changed but `revision` still reads 2 while the receipt's `last_seen.revision` is 2 with a different digest, **when** it runs, **then** it stops with `E-REVISION-STALE` and nothing is mutated.
3. **Given** action `A-…-1` approved at revision 2 with a `failed` row, **when** the title is edited and `revision` becomes 3, **then** the stored approval is ignored, the action is `proposed` again, and interactive mode asks anew.
4. **Given** issue #288 closed as completed 30 days ago with marker `agent-skills:fingerprint sha256:0b77e2a91c5d4f10`, **when** an action covering a finding with that fingerprint is processed under a 90-day window, **then** the row is `duplicate` with #288, `state: closed`, and no issue is created.
5. **Given** the same closed match under `closed_window_days: 14`, **when** processed, **then** it is not found and the action proceeds to approval.
6. **Given** three approved simple actions where the second `gh issue create` fails twice with a 502, **when** the run continues, **then** the receipt holds `filed` for the first, `failed attempt: 1` for the second, `filed` for the third, the artifact statuses match, and a re-run files only the second.
7. **Given** a run that created issue #301 and crashed before writing the row, **when** re-run, **then** the action-marker search finds #301 and the row is `filed` with `recovered: true`; no second issue exists.
8. **Given** `ticket_decomposition.provider: none`, **when** a complex action is approved, **then** the row is `deferred` with `decomposition-unavailable`, the action block is unchanged, and no single issue is created for it.
9. **Given** `provider: to-tickets` and a passing preflight, **when** a complex action is approved in a live session, **then** one parent issue labelled `needs-decision` and not `ready-for-agent` is created, the row is `handed-off` with its URL, and the overlay plus `/to-tickets #<parent>` are printed.
10. **Given** `approval.mode: interactive` and `--unattended`, **when** the publisher runs, **then** every non-duplicate action is `deferred` with `approval-required`, no issue is created, and the result says the run needs a live session.
11. **Given** `pre-approved-severities` with `auto_file_severities: [critical, high]` and `auto_file_kinds: [simple]`, **when** an unattended run sees a `high` simple action, a `medium` simple action, and a `high` complex action, **then** only the first is filed; the other two are `deferred` with `approval-required`.
12. **Given** `approval.mode: never`, **when** the publisher runs, **then** every action is `deferred` with `approval-mode-never` and the run entry is recorded.
13. **Given** the `finding` label absent and `create_missing_labels: false`, **when** preflight runs, **then** it stops before any issue is created, naming the label and the `gh label create` command.
14. **Given** `dogfood` finishing with `publish-findings` not installed, **when** it reports, **then** its own outcome is unchanged and the last line is `publication: not-attempted (not-installed)` with the install command.
15. **Given** `dogfood` finishing with the publisher installed but the artifact failing validation, **when** it reports, **then** its own result is still successful and the last line is `publication: failed (invalid-artifact: …)`.
16. **Given** a human-written artifact that conforms, **when** a user runs `publish-findings docs/findings/manual-20260902-billing.md`, **then** it is processed exactly like a producer artifact.
17. **Given** an action with a fuzzy title candidate in an unattended pre-approved run, **when** processed, **then** it is `deferred` with `possible-duplicate` and the candidate URLs, and an interactive re-run presents them.

## Risks

- Receipts are committed and land only when the producer's docs PR merges; two machines can file the same finding in that window. Layer 2 (open-issue markers) closes most of it; the residual is a duplicate a human closes.
- Evidence links are default-branch blob URLs that resolve only after the docs PR merges; an issue opened first shows dead links briefly. Mitigation: `W-EVIDENCE-UNCOMMITTED` and the result's "paths to commit" line.
- Fuzzy matching at 80 percent token overlap will both miss paraphrases and flag near-misses; it is deliberately advisory. #15 tunes the threshold on pilot data.
- `gh issue list --search` uses GitHub search, which indexes with delay and has a rate limit; a fresh issue may not be found seconds later. Layer 1 (receipts) covers same-machine runs; the action-marker recovery covers the crash case.
- `revision` discipline depends on producers and humans bumping it; `E-REVISION-STALE` catches silent edits but only after the fact.
- A `human-decision` simple action files as a `needs_decision` gate, which the human-gate pair will surface; producers that label every question this way can flood the gate frontier. Producers should reserve actions for things worth deciding and leave `note` findings actionless.

## Open items

1. **#11, #12 decide** whether producers file directly when the publisher is absent. Recommendation: no; produce the artifact and print the install command. If they do file, they must write this receipt format. #8 should reword its rationale sentence accordingly.
2. **Maintainer ratifies**: no umbrella issue by default; closed exact matches count as duplicates by default with an interactive regression override; complex actions never auto-approved by default; the default severity scale `critical | high | medium | low`; the 80 percent fuzzy threshold; three-attempt retry limit.
3. **#8 absorbs** the flagged keys and states whether `findings.severity_scale` is per-repository only or may also be overridden per artifact as specified here.
4. **#10** must document that a `needs_decision` issue created by the publisher (marker `agent-skills:action`) is a decomposition gate when its source action was `complex`, and a plain decision gate otherwise; the parent-issue body says which.
5. Whether `research-project` findings need a second severity-like axis (impact or confidence); if so it lives in an extension field and does not enter `auto_file_severities`.
6. A small validator script under `scripts/` would let producers and #15 run the schema checks deterministically; deferred to implementation.

## Consequences

- **#11 (`dogfood`)** and **#12 (`research-project`)**: produce `findings` v1 exactly as above (IDs, fingerprints, dense numbering, `revision: 1`, empty `## Publication`); set `kind` and `disposition` per action; validate before handoff; implement the handoff and the separate `publication:` result line; own committing the artifact and receipt in their docs PR; keep `note` and `wontfix` findings actionless. #12 reserves an append-only `## Publications` heading in its ledger.
- **#10 (human-gate)**: consumes parent proposal issues labelled `needs_decision` with the action marker; runs the overlay and native-edge verification; may read `state/publications/` to link back to the artifact.
- **#9 (work-execution)**: unchanged; filed `ready_for_agent` issues enter the frontier like any other.
- **#8 (capability configuration)**: absorbs the proposed keys and adds the `findings` and `publication` majors to the fingerprint inputs (already listed).
- **#15 (validation and pilot)**: tests all seventeen scenarios, the append-only receipt invariant under crash injection, marker search against open and closed issues, drift between `references/disposition-policy.md`, `protocol-contracts.md`, `setup.md` and their canonical sources, and the hygiene scan over this skill's examples (`example.com`, `acme/widgets` placeholders must be allowlisted).

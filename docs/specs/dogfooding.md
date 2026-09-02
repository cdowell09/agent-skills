# Product dogfooding contract

Date: 2026-09-02
Question: How should the source project's `dogfood` workflow become a repository-agnostic product-dogfooding skill with configurable target, personas, journeys, authentication boundaries, competitor comparison, and evidence; what conforming findings artifact does it produce; and how does it hand that artifact to the findings publisher while staying complete without it?
Resolves: [#11 Specify generic dogfooding](https://github.com/cdowell09/agent-skills/issues/11)
Inputs: [ADR 0001](../adr/0001-npx-skills-distribution-contract.md), [ADR 0002](../adr/0002-skill-boundaries-and-shared-contracts.md), [ADR 0003](../adr/0003-ticket-decomposition-provider.md), [ADR 0004](../adr/0004-licensing-attribution-and-contribution.md), [Capability configuration spec](capability-configuration.md), [Findings publication spec](findings-publication.md), [source skill inventory](../research/2026-07-12-source-skill-inventory.md), [source provenance audit](../research/2026-07-12-source-provenance.md)

Skill names used here (`pick-up-work`, `work-the-board`, `pick-up-human-task`, `work-the-human-board`, `dogfood`, `research-project`, `publish-findings`) are working names; a later naming pass may rename them.

## Decision

- `dogfood` walks a deployed product through configured journeys in several behavioral stances, as a real user would, and writes one **Findings Artifact** (`findings` v1, exactly as the publication spec defines it) per run. It observes and never fixes: its only mutations are evidence files under `evidence.ui.dir`, the artifact under `findings.artifact_dir`, a run receipt, and, under the default persistence policy, one commit on a dedicated branch. Filing is delegated entirely to `publish-findings`; there is no second filing path.
- Everything product-specific is configuration: target and readiness, environment policy, authentication mode, journey map, personas and stances, competitors, evidence rules, severity vocabulary, budget. The skill ships the method and a fresh, product-neutral stance library.
- The deployed revision is proven when `dogfood.target.revision_source` allows it and otherwise recorded as unknown with an explicit assumption; the artifact never presents an inferred commit as fact.
- Handoff follows the publisher's producer contract: sibling discovery, one-argument invocation, a separate `publication:` line. A dogfood run is successful when its artifact is written and valid, regardless of what publication does.
- Persistence default is `commit-on-branch` (a branch, a commit, a push, no pull request, never a merge); `leave-uncommitted` is the opt-out. Maintainer ratifies.

## Purpose and boundary

`dogfood` answers "what does a real user hit on the currently deployed product?" It produces evidence and proposals; it does not diagnose code, open fixes, or touch the tracker.

| | `dogfood` |
| --- | --- |
| Owns | Readiness check, revision proof or disclosure, journey prioritization, stance walks, evidence capture, journey-first narrative, finding and action drafting, producer-side dedupe marking, artifact validation, persistence, handoff, receipt |
| Never does | Modify product code or data outside the data policy; enter or print credentials; file issues, open pull requests against the default branch, or merge anything; run the publisher's approval or dedupe logic; continue past its budget |
| Mutations, exhaustively | Files under `evidence.ui.dir` and `state/local/`; `<findings.artifact_dir>/<artifact_id>.md`; `state/local/runs/<owner-id>.json`; under `commit-on-branch`, one commit on a branch named by `findings.persistence.branch_pattern`, pushed; and whatever `publish-findings` does under its own contract |
| Completes when | The artifact exists, validates against `references/findings-artifact.md`, and the receipt is emitted |

**Removed: report publication through a docs PR with auto-merge.** The source skill committed its report, opened a pull request, and merged it under an admin convention: the hidden side-effect chain the inventory warns about. Persistence is now configuration:

```yaml
findings:
  artifact_dir: docs/findings
  persistence:                        # PROPOSED for #8
    commit: commit-on-branch          # commit-on-branch | leave-uncommitted
    branch_pattern: "findings/{artifact_id}"
    push: true                        # effective only with commit-on-branch
```

`commit-on-branch` creates `findings/<artifact_id>` from the default branch tip in a temporary git worktree (the user's checkout stays untouched; `git worktree` is part of the required git capability), commits the artifact, its evidence, and, after publication, the publication receipt, pushes when `push` is true, prints the `gh pr create` command, and stops. It never opens or merges the pull request. `leave-uncommitted` writes the same files into the working tree and lists them in the result.

Default `commit-on-branch`, because every downstream contract assumes a committed artifact: publication receipts are committed so other machines can deduplicate, issue evidence links are default-branch blob URLs, and this skill's own dedupe reads prior artifacts from the tree. A commit on a fresh branch is the least surprising durable mutation available: reversible with one `git branch -D`, invisible to the default branch, reviewable as an ordinary pull request. `leave-uncommitted` serves repositories with their own documentation process and dry runs. Interactive runs confirm at stop point S3; unattended runs apply the policy.

## Preserve, remove, configure, delegate

| Preserve (the method) | Remove (source identity and side effects) | Configure | Delegate |
| --- | --- | --- | --- |
| Deployment readiness check before touching the product; prioritize recently changed flows; walk several behavioral stances; journey-first narration; screenshot plus console plus network evidence per step of interest; deduplicate before proposing; approval-gated filing | Product and health URLs; `main` as a proxy for the deployed revision; fixed repository; real-user authentication as the only mode; the fixed core journey; React/monorepo route heuristics for change mapping; the competitor roster and its descriptions; report path, docs PR, and admin auto-merge; Project, Theme, and milestone metadata; label routing; private-repository screenshot rules as defaults; the fixed stance catalog text; the duplicated filing overlay; web-oriented agent routing | `dogfood.target` (URL, readiness, revision source, environment, production allow flag); `dogfood.auth`; `dogfood.journeys`; `dogfood.personas` and `dogfood.stances`; `dogfood.competitors`; `dogfood.recent_changes`; `dogfood.budget`; `dogfood.data_policy`; `evidence.ui.*`; `findings.severity_scale`; `findings.artifact_dir` and `findings.persistence` | Filing, dedupe authority, approval, receipts: `publish-findings` (#18). Complex-action slicing: the `ticket-decomposition` capability through the publisher (ADR 0003). Browser mechanics: the browser-control capability and `references/runtime-notes.md`. Setup and validation: the generated `references/setup.md` |

The stance library in `references/stances.md` is written fresh for this catalog from the requirement "several distinct behavioral stances that surface different classes of friction"; it does not reproduce the source project's stance text, per ADR 0004's rules for rewritten material.

## Target, environment, and readiness

```yaml
dogfood:
  target:
    url: https://staging.example.com
    environment: staging                       # staging | production   (PROPOSED for #8)
    allow_production: false                    # required true when environment is production (PROPOSED)
    readiness:
      url: https://staging.example.com/health  # either url + expect_status ...
      expect_status: 200
      command: ""                              # ... or a command exiting 0 (PROPOSED alternative)
    revision_source: none                      # none | header | endpoint (capability spec)
    revision_header: X-App-Revision            # with header (PROPOSED)
    revision_url: https://staging.example.com/version   # with endpoint (PROPOSED)
    revision_json_path: "$.sha"                # with endpoint; default: whole body (PROPOSED)
```

- **Readiness** is the capability spec's Tier 3 "Dogfood session start" recheck: the URL returns `expect_status`, or the command exits 0, within 30 seconds, retried once. Failure ends the run with `target-unhealthy` before any browser session opens; no artifact is written.
- **Revision.** With `header` or `endpoint`, the skill reads the value, resolves it to a commit on the default branch (`git cat-file -e`), and records `revision: proven <sha>`. With `none`, or when the value does not resolve, it records `revision: unknown; assumed <default_branch>@<sha> at <time> (unverified)` from the fetched default-branch tip. The line appears verbatim in the artifact summary and the receipt, turning the inventory's "inferred, not proven" risk into a disclosed assumption; recent-change mapping uses whichever sha was recorded.
- **Environment policy.** `staging` is the default. `production` is refused unless `allow_production: true`, and then adds a data-safety rule configuration cannot weaken: no destructive actions (delete, cancel, archive, bulk edit), no payments or checkout completion past the last review screen, no messages to third parties, and state-changing steps only under `data_policy: scratch-account` with a configured test account. `data_policy: observe-only` (the capability spec's default) forbids submitting any state-changing form on any environment; journeys tagged `mutating` are skipped under it, disclosed.

## Authentication boundary

The skill never sees, types, stores, or prints a raw credential. `dogfood.auth.mode` (PROPOSED for #8; supersedes `browser.auth.boundary` for this skill, and #8 should consider the same values for `pick-up-work`'s UI-evidence step):

| Mode | What the browser capability must provide | Fallback when it cannot |
| --- | --- | --- |
| `none` | Nothing beyond navigation | n/a; journeys tagged `auth` are skipped and disclosed |
| `attach-to-user-session` | Attachment to a browser profile in which a human is already signed in (the optional user-session-attach capability) | Interactive: offer `manual-login-handoff`; unattended: unauthenticated journeys only, `auth` journeys skipped, downgrade recorded |
| `provided-test-account` | A fill step performed by the browser tool or a script that reads the credential from the reference in `dogfood.auth.account_ref` (an environment variable name or a runtime secret name); the value never enters the conversation, artifact, evidence, or receipt, and no screenshot is taken while a credential field is focused | Unauthenticated journeys only, disclosed. Never falls back to asking the human to paste a secret into the session |
| `manual-login-handoff` | A live human: the skill opens the sign-in route, pauses at stop point S2, and resumes when the human says the session is ready | Unattended: unauthenticated journeys only, disclosed |

`browser.auth.bypass: header:<name>` from the capability spec is treated as `provided-test-account` whose reference supplies the header value. In every mode the skill signs nothing out and clears nothing; session state belongs to the human or the runtime.

## Journeys, stances, and prioritization

**Journey map** (config-owned; the capability spec's `{id, routes, changed_paths}` extended additively):

```yaml
dogfood:
  journeys:
    - id: checkout
      name: Buy one item as a signed-in customer
      entry: /products
      steps:                                   # PROPOSED: free-text steps a user would take
        - open a product and add it to the cart
        - open the cart and apply a coupon
        - proceed to the review screen; stop before payment
      expected: The review screen shows the discounted total and the shipping estimate
      tags: [auth, mutating, core]             # auth | mutating | mobile | core | free text
      routes: ["/products", "/cart", "/checkout"]
      changed_paths: ["apps/web/src/checkout/**", "packages/pricing/**"]
  personas:
    - { id: new-shopper, stance: first-time-user }
    - { id: regular, stance: returning-power-user }
  stances: []                                  # PROPOSED: custom [{ id, summary, notices, ignores }]
  recent_changes:                              # PROPOSED
    since: last-artifact                       # last-artifact | window
    lookback: 14d                              # cap, or the window when since is window
  budget:                                      # PROPOSED
    max_journeys: 4
    max_stances_per_journey: 2
    max_minutes: 60
```

**Stance library.** `references/stances.md` ships five product-neutral stances, each defined by what the user wants, notices first, ignores, and does when blocked: `first-time-user` (no product vocabulary, reads everything, gives up at the first ambiguity), `returning-power-user` (knows the goal, uses keyboard and shortcuts, resents extra clicks and re-entered data), `impatient-mobile-user` (narrow viewport, one thumb, poor connection, abandons slow or blocked steps), `accessibility-focused-user` (keyboard-only; focus order, labels, contrast, announcements), and `skeptical-evaluator` (measures the product against its own marketing copy and competitors; looks for pricing and trust signals). Personas bind a stance to a name and notes; repositories add stances under `dogfood.stances`. A stance the runtime cannot fully serve (no viewport control for `impatient-mobile-user`) is walked with the closest approximation and the gap recorded in the journey table.

**Recent-change prioritization.** With `since: last-artifact`, the window starts at the revision recorded in the newest `dogfood-*` artifact under `findings.artifact_dir` (proven or assumed), capped by `lookback`; with `window`, or no prior artifact, it is `lookback`. `git log --name-only <since>..<default_branch>` yields changed paths matched against each journey's `changed_paths` globs; journeys with hits are ordered by hit count, then the `core` tag, then config order; the rest follow in config order. No hits, or no `changed_paths` configured, means all journeys in config order, disclosed. `--journey <id>` (repeatable) overrides.

**Budget.** At most `max_journeys` journeys, each in at most `max_stances_per_journey` stances (configured personas in order); no new walk starts after `max_minutes`, and walks in progress finish. Defaults 4, 2, and 60 fit one sitting and a report reviewable in fifteen minutes. Exceeding the budget yields `partial` with the unwalked journeys listed; still a successful run.

**Walk protocol.** Per journey and stance: navigate to `entry`, follow `steps` as that stance would, judge against `expected`, capture evidence at each step of interest, stop before any step the data policy forbids and record the boundary. Narrate the walk as the stance experienced it; findings are extracted afterwards, so one defect seen from two stances becomes one finding with two observations.

## Competitor comparison

Optional. `dogfood.competitors[]` is `{ name, url, compare: [<what to compare>], journey: <id> }` (the capability spec's `{name, url}` plus two additive keys). An empty list means no competitor material anywhere in the artifact. When present, after the journey walks and within the same budget, the skill visits each public URL as an anonymous visitor and records observations against `compare`. Third-party rules: public pages only; no account creation, sign-in, or form submission; nothing beyond what a visitor sees; screenshots stay in `state/local/` and are never committed; the artifact cites the public URL as evidence. Comparison text lives in a `### Competitor comparison` block inside `## Summary`; competitor observations are never findings or actions on their own but may support an `improvement` finding about the configured product.

## Evidence

- **Capture rules.** At each step of interest (entry, every state change, every deviation from `expected`, every error): one screenshot; console errors and warnings since the previous capture; failed or slow network requests (status ≥ 400, or over 3 seconds) with method, URL, status, and timing, query strings and bodies stripped; step timing. Screenshots that would show a credential field or third-party personal data are skipped and noted.
- **Two tiers.** Walk captures go to `state/local/evidence/<artifact_id>/<journey>/<stance>/<NN>-<step-slug>.<ext>` (gitignored). When a finding is drafted, its supporting captures are copied to `<evidence.ui.dir>/<artifact_id>/<finding-id>/<kind>-<NN>.<ext>`, `kind` in `screenshot | console | network | timing`, and those paths are the finding's `evidence` list. Only finding-linked evidence is committed; the walk tier is kept for `evidence.ui.retention_days` (PROPOSED for #8, default 14) and pruned at the next run's start.
- **Embedding.** `evidence.ui.embed` from the capability spec: `auto` inlines screenshots in the artifact when discovered visibility is `public` and lists paths only when `private`; `inline` and `link-only` force either. The publisher applies the same rule to issue bodies. Competitor screenshots never leave the walk tier.
- **Artifact versus disk.** The artifact carries paths, one-line captions, and the console or network lines that matter; full logs and HAR-style captures stay as files. Nothing under `state/local/` is referenced from the artifact.

## The artifact

`dogfood` produces `findings` v1 exactly as the publication spec defines it: four H2 sections in order, dense IDs, recomputable fingerprints, `revision: 1`, empty `## Publication`. The dogfood profile adds only what the contract allows:

- One artifact per run. `artifact_id` slug is the journey id when one journey was walked, otherwise `recent-changes` or the `--topic` the user supplied. Frontmatter `subject` is `url:<target origin>`; every finding carries `- subject: journey:<id>` so its fingerprint is stable across runs and independent of which journeys shared the artifact. The fingerprint is computed over the finding's effective subject (open item 2 asks #18 to confirm this reading).
- `## Summary` is the journey-first narrative: a run header, one `### Journey: <id>` block per journey walked, an optional `### Competitor comparison`, and a `### Dedupe` note. H3 headings inside `## Summary` are free Markdown and do not disturb the H2 structure.
- Finding extension fields, ignored by the publisher: `journey`, `stance`, `step`, `route`, `likely_duplicate_of`.
- `note` and `wontfix` findings never get actions. `question` findings get an action only when a decision is worth a gate.
- Severity comes from `findings.severity_scale` (default `critical | high | medium | low`); `references/severity-rubric.md` defines the default levels in user-impact terms (blocked from the goal; goal reached with data loss or a workaround; friction; polish). A custom scale must supply `dogfood.severity_rubric: { <level>: <one line> }` (PROPOSED) or validation stops.

Example (hashes are real: `sha256("dogfood-20260902-recent-changes")` starts `6989422f`; fingerprints follow the publication spec's derivation over the effective subject):

````markdown
---
artifact_version: 1
artifact_id: dogfood-20260902-recent-changes
artifact_short_id: 6989422f
producer: { skill: dogfood, version: "0.1.0" }
created_at: 2026-09-02T16:40:07Z
subject: url:https://staging.example.com
revision: 1
---

## Summary

Target https://staging.example.com (staging). Revision: unknown; assumed main@4c1d9e2 at 2026-09-02T16:02Z (unverified).
Auth: attach-to-user-session (test account "qa-shopper"). Data policy: scratch-account. Budget: 2 of 4 journeys, 2 stances each, 38 of 60 minutes.
Recent changes since dogfood-20260819-checkout (assumed main@a71f03b): 23 files under apps/web/src/checkout/** and packages/pricing/** mapped to `checkout`; `search` walked as the next core journey.

| Journey | Stance | Result | Findings |
| --- | --- | --- | --- |
| checkout | first-time-user | reached review screen; total wrong after coupon removal | F-6989422f-1, F-6989422f-3 |
| checkout | returning-power-user | reached review screen; same stale total | F-6989422f-1 |
| search | accessibility-focused-user | goal reached by keyboard; no visible focus on results | F-6989422f-2 |
| search | impatient-mobile-user (viewport approximated at 390px) | goal reached | none |

### Journey: checkout

As a first-time user I found the product page quickly and added one item. Applying the coupon worked, but removing it left the summary showing the discounted total until I reloaded (F-6989422f-1). While the coupon validated there was no feedback for about four seconds, and I clicked the button again (F-6989422f-3). I stopped at the review screen as instructed; the payment step was not attempted. The confirmation page could not be checked because the journey stops before it, but an analytics request failed on the review screen (F-6989422f-4).

### Journey: search

Keyboard-only: the search field is reachable and labelled, results announce their count, but tabbing through result cards shows no visible focus ring, so I lost my place twice (F-6989422f-2).

### Dedupe

F-6989422f-3 matches a finding in dogfood-20260819-checkout whose action was filed and is still open; marked as a likely duplicate and re-proposed for the publisher to decide.

## Findings

### F-6989422f-1: Checkout summary shows a stale total after a coupon is removed
- severity: high
- disposition: bug
- fingerprint: sha256:29a74386009f1d7a
- evidence:
  - docs/evidence/dogfood-20260902-recent-changes/F-6989422f-1/screenshot-01.png
  - docs/evidence/dogfood-20260902-recent-changes/F-6989422f-1/screenshot-02.png
  - docs/evidence/dogfood-20260902-recent-changes/F-6989422f-1/network-01.txt
- subject: journey:checkout
- journey: checkout
- stance: first-time-user, returning-power-user
- step: remove coupon on the cart page
- route: /cart

After "Remove" the coupon row disappears but the order total keeps the discounted value; `DELETE /api/cart/coupon` returns 204 and no cart refetch follows. A reload shows the correct total. Reproduced in both stances.

### F-6989422f-2: Search results page has no visible focus indicator on result cards
- severity: medium
- disposition: bug
- fingerprint: sha256:9b09bf5982806cd6
- evidence:
  - docs/evidence/dogfood-20260902-recent-changes/F-6989422f-2/screenshot-01.png
- subject: journey:search
- journey: search
- stance: accessibility-focused-user
- route: /search

Tab moves focus through cards (the URL bar shows the link target) but nothing visible changes; the global `outline: none` rule on `.card a` is the visible cause in the computed styles.

### F-6989422f-3: Coupon field gives no feedback while the code is validating
- severity: low
- disposition: improvement
- fingerprint: sha256:c631e7d346e3a256
- evidence:
  - docs/evidence/dogfood-20260902-recent-changes/F-6989422f-3/timing-01.txt
- subject: journey:checkout
- journey: checkout
- stance: first-time-user
- likely_duplicate_of: dogfood-20260819-checkout/F-b2e07a19-2 (https://github.com/acme/widgets/issues/244, open)

Validation took 3.9 s with no spinner or disabled state; a second click sent a second request.

### F-6989422f-4: Order confirmation page triggers a failed analytics request
- severity: low
- disposition: note
- fingerprint: sha256:a95416968e0024de
- evidence:
  - docs/evidence/dogfood-20260902-recent-changes/F-6989422f-4/network-01.txt
- subject: journey:checkout
- journey: checkout
- route: /checkout/review

`POST https://analytics.example.com/collect` fails with 403 on the review screen; no user-visible effect. Recorded for the owning team; no action proposed.

## Proposed Actions

### A-6989422f-1: Refetch the cart after a coupon is removed
- findings: [F-6989422f-1]
- kind: simple
- disposition: agent-doable
- labels: [bug, finding]
- status: proposed

Invalidate the cart query on coupon removal (mirror the add path). Add a test asserting the total returns to the undiscounted value. Evidence shows the 204 followed by no refetch.

### A-6989422f-2: Restore a visible focus style on search result cards
- findings: [F-6989422f-2]
- kind: simple
- disposition: agent-doable
- labels: [bug, finding]
- status: proposed

Replace the `outline: none` rule with a focus-visible style meeting contrast guidance; verify with keyboard navigation.

### A-6989422f-3: Show validating state on the coupon field
- findings: [F-6989422f-3]
- kind: simple
- disposition: agent-doable
- labels: [improvement, finding]
- status: proposed

Likely duplicate of https://github.com/acme/widgets/issues/244 (still open); re-proposed so the publisher can confirm and link. Disable the button and show a spinner while the coupon request is in flight.

## Publication
````

## Deduplication before Proposed Actions

Read-only, producer-side, and advisory; the publisher's three-layer dedupe remains the authority (ADR 0002).

1. Compute each finding's fingerprint.
2. Scan every `<findings.artifact_dir>/*.md` with `artifact_version: 1` for findings with the same fingerprint, and every `state/publications/*.yaml` for rows whose `fingerprints` contain it, taking the newest row per action.
3. Mark the finding `- likely_duplicate_of: <artifact_id>/<finding-id> (<issue url>, <state>)` when a match exists. The finding stays in `## Findings` because a re-observation is itself information.
4. Draft the action anyway unless the matched row is `filed` or `handed-off` **and** the issue is open, in which case still draft it but open the body with the likely-duplicate line, as in `A-6989422f-3`; a closed match gets the body line "possibly a regression of <url>". The publisher will return `duplicate`, offer the regression override, or file.
5. Record the matches in the receipt (`dedupe.likely_duplicates`) and in the `### Dedupe` note.

Nothing is written to GitHub during this step; `gh` is not needed.

## Handoff, result, and receipt

**Order at the end of a run:** write the artifact → validate with the publisher's rules (`references/findings-artifact.md`) → emit the dogfood result → hand off → persist → report. The outcome is fixed before handoff, so publication cannot change it. Persistence follows publication so one commit carries the artifact, evidence, the publication receipt, and the `## Publication` line together; if publication is skipped or fails, the commit carries what exists.

**Handoff** is the publication spec's producer contract, verbatim: locate `publish-findings` by the capability spec's sibling-discovery order, reading only its frontmatter; if found and the runtime supports sequential interactive skill invocation, invoke `publish-findings <artifact path>` (plus `--unattended` when this run is unattended) and wait for its result line; end the report with exactly one `publication:` line from the contract's vocabulary (`not-attempted (not-installed)` with the install command, `not-attempted (invocation-unavailable)` with the command for the user, `succeeded (...)`, `succeeded-with-failures (...)`, `failed (<outcome>: <reason>)`). Without the publisher the skill produces the artifact, prints the install command, and stops; it never files directly (publication spec open item 1, decided as recommended; #8 rewords its rationale sentence).

**Receipt `receipt.dogfood` v1.** Emitted like the other families: the last line of output as `AGENT_SKILLS_RECEIPT ` plus one-line JSON, and `state/local/runs/<owner-id>.json`. Required: `receipt`, `family`, `skill`, `owner`, `outcome`, `artifact`, `publication`, `protocols`; everything else additive.

```json
{"receipt": 1, "family": "dogfood", "skill": "dogfood", "skill_version": "0.1.0",
 "owner": "claude-code@buildbox/20260902T160207Z-9c4e11ab",
 "outcome": "partial",
 "artifact": {"id": "dogfood-20260902-recent-changes", "path": "docs/findings/dogfood-20260902-recent-changes.md", "revision": 1, "findings": 4, "actions": 3, "valid": true},
 "target": {"url": "https://staging.example.com", "environment": "staging", "revision": "unknown; assumed main@4c1d9e2 (unverified)", "readiness": "pass"},
 "auth": {"mode": "attach-to-user-session", "effective": "attach-to-user-session"},
 "journeys": [{"id": "checkout", "stances": ["first-time-user", "returning-power-user"], "status": "walked", "findings": ["F-6989422f-1", "F-6989422f-3", "F-6989422f-4"]},
              {"id": "search", "stances": ["accessibility-focused-user", "impatient-mobile-user"], "status": "walked", "findings": ["F-6989422f-2"]},
              {"id": "account-settings", "status": "skipped", "reason": "budget"}],
 "budget": {"max_journeys": 4, "max_stances_per_journey": 2, "max_minutes": 60, "used_minutes": 38},
 "evidence": {"dir": "docs/evidence/dogfood-20260902-recent-changes", "files": 6, "embed": "link-only"},
 "dedupe": {"likely_duplicates": [{"finding": "F-6989422f-3", "matched": "dogfood-20260819-checkout/F-b2e07a19-2", "issue_url": "https://github.com/acme/widgets/issues/244", "state": "open"}]},
 "persistence": {"policy": "commit-on-branch", "branch": "findings/dogfood-20260902-recent-changes", "commit": "e9d3a7c", "pushed": true},
 "publication": {"status": "succeeded", "receipt": ".agent-skills/state/publications/dogfood-20260902-recent-changes.yaml", "filed": 2, "handed_off": 0, "duplicate": 1, "deferred": 0, "failed": 0},
 "downgrades": ["viewport control unavailable; impatient-mobile-user approximated with a 390px window"],
 "reason": null, "started_at": "2026-09-02T16:02:07Z", "finished_at": "2026-09-02T16:52:30Z",
 "protocols": {"config": 1, "findings": 1, "receipt.dogfood": 1}}
```

| Outcome | Meaning | Artifact | Successful |
| --- | --- | --- | --- |
| `completed` | Every planned journey and stance walked; artifact valid (zero findings allowed) | written | yes |
| `partial` | Artifact valid but at least one planned walk was skipped (budget, auth fallback, data policy, target flake); the summary lists which | written | yes |
| `target-unhealthy` | Readiness failed | none | no |
| `setup-required` | Missing or invalid configuration; stop-with-guidance block | none | no |
| `preflight-failed` | Required capability missing, production without `allow_production`, custom scale without rubric | none | no |
| `aborted` | Human stop or error mid-session; findings captured so far are written if at least one walk completed | maybe | no |

`publication.status` mirrors the `publication:` line (`not-attempted`, `succeeded`, `succeeded-with-failures`, `failed`) and never alters `outcome`. Consumers (today only humans and #15's tests) ignore unknown fields and stop on a higher major.

## Capabilities

| Capability | Required | Minimum operations | Fallback |
| --- | --- | --- | --- |
| Browser control | yes | navigate, click, type, screenshot, read console, read network (failed requests with status and timing) | none; stop at preflight with `preflight-failed` |
| Viewport or device emulation | optional | set viewport size | mobile stances approximated with a narrow window, downgrade recorded |
| User-session attach | optional | attach to a signed-in profile | see the authentication table |
| Sequential interactive skill invocation | optional | invoke `publish-findings` and wait | print the command; `publication: not-attempted (invocation-unavailable)` |
| Web fetch or a second browser tab | optional | read public competitor pages | competitor section omitted, downgrade recorded |
| git | yes | `log`, `worktree`, `commit`, `push` | none; stop |
| `gh` | optional | print the PR command; nothing else | not needed by the core skill; the publisher requires it |
| `publish-findings` sibling | optional | producer handoff contract | artifact only, install command printed |
| `ticket-decomposition` | optional, via the publisher | ADR 0003 | complex actions deferred by the publisher |
| Live human in session | optional | confirmations, manual login | unattended policy applies; `manual-login-handoff` unavailable |

Runtime notes, kept in `references/runtime-notes.md` and limited to genuine differences: Claude Code commonly provides browser control through an installed browser MCP server or an extension that attaches to the user's running profile, which is what makes `attach-to-user-session` available there; Codex commonly provides a launched automation browser, so attach is usually unavailable and `provided-test-account` or `manual-login-handoff` is the practical mode. No runtime tool is named as a requirement; `capabilities.runtime_notes` carries the repository's actual provider.

## Configuration keys consumed

Keys marked PROPOSED are additions for #8 to absorb; all others are the capability spec's. Only `dogfood.target.url`, `dogfood.target.environment` (when `production`, with `allow_production`), and `findings.approval.mode` are mutation-affecting in the spec's sense and asked during setup; the rest are optional with the defaults shown.

```yaml
version: 1
repository: { visibility: private }            # discovered; drives evidence embedding
dogfood:
  target:
    url: https://staging.example.com           # required
    environment: staging                       # PROPOSED: staging | production; default staging
    allow_production: false                    # PROPOSED: must be true for production
    readiness: { url: https://staging.example.com/health, expect_status: 200, command: "" }   # command PROPOSED
    revision_source: none                      # none | header | endpoint
    revision_header: ""                        # PROPOSED
    revision_url: ""                           # PROPOSED
    revision_json_path: ""                     # PROPOSED
  auth:                                        # PROPOSED (supersedes browser.auth.boundary for this skill)
    mode: none                                 # none | attach-to-user-session | provided-test-account | manual-login-handoff
    account_ref: ""                            # env var or runtime secret name; never a value
  journeys: []                                 # at least one; { id, name, entry, steps, expected, tags, routes, changed_paths }
  personas: []                                 # at least one; { id, stance, notes }
  stances: []                                  # PROPOSED: custom stances { id, summary, notices, ignores }
  competitors: []                              # { name, url, compare, journey }; compare/journey PROPOSED
  recent_changes: { since: last-artifact, lookback: 14d }        # PROPOSED
  budget: { max_journeys: 4, max_stances_per_journey: 2, max_minutes: 60 }   # PROPOSED
  data_policy: observe-only                    # observe-only | scratch-account
  severity_rubric: {}                          # PROPOSED; required when findings.severity_scale is customized
browser:
  mode: attach                                 # attach | launch | none
evidence:
  ui:
    dir: docs/evidence                         # required
    embed: auto                                # auto | inline | link-only
    retention_days: 14                         # PROPOSED; walk-tier evidence under state/local/
findings:
  artifact_dir: docs/findings
  approval: { mode: interactive }              # REQUIRED; consumed by the publisher, validated here
  severity_scale: [critical, high, medium, low]   # proposed by #18
  persistence:                                 # PROPOSED
    commit: commit-on-branch                   # commit-on-branch | leave-uncommitted
    branch_pattern: "findings/{artifact_id}"
    push: true
capabilities:
  browser: auto
  interactive_skill_invocation: auto
  runtime_notes: { claude_code: "", codex: "" }
```

Additions for #8, summarized: `dogfood.target.{environment, allow_production, readiness.command, revision_header, revision_url, revision_json_path}`, `dogfood.auth.{mode, account_ref}`, `dogfood.journeys[].{name, entry, steps, expected, tags}`, `dogfood.personas[].notes`, `dogfood.stances`, `dogfood.competitors[].{compare, journey}`, `dogfood.recent_changes`, `dogfood.budget`, `dogfood.severity_rubric`, `evidence.ui.retention_days`, `findings.persistence`. The Tier 3 "Dogfood session start" row gains the `allow_production` and `readiness.command` checks. `browser.auth.boundary` is not consumed by this skill.

## Rough `SKILL.md` outline

```yaml
---
name: dogfood
description: Walk a deployed product through configured user journeys in several behavioral stances, capture screenshot, console, and network evidence, and write a conforming findings artifact with proposed actions; hands off to publish-findings when installed and never modifies the product.
license: MIT
compatibility: Browser control (navigate, click, type, screenshot, console, network); git; .agent-skills/config.yaml with dogfood.target and findings.approval.mode. Optional: user-session attach, viewport emulation, web fetch, publish-findings (this catalog), ticket-decomposition provider via the publisher.
metadata:
  author: Christian Dowell
  version: "0.1.0"
  provenance: original
  contracts: "config=1 findings=1 receipt.dogfood=1"
---
```

1. **Purpose and boundary** — observe as a user; never fix; the exhaustive mutation list.
2. **Arguments** — `--journey <id>` (repeatable), `--stance <id>`, `--topic <slug>`, `--unattended`, `--dry-run`, `--no-publish`, `--no-commit`.
3. **Preflight** — config and validation tiers per `references/setup.md`; capability detection; environment policy; budget; print the plan (target, revision line, auth mode, ordered journeys and stances, competitors, persistence). **S1: stop and confirm in interactive runs; unattended proceeds only when nothing needs a human.**
4. **Readiness and revision** — Tier 3 check; prove or disclose the revision; `target-unhealthy` on failure.
5. **Authenticate** — apply `dogfood.auth.mode` per `references/auth-modes.md`. **S2: `manual-login-handoff` pauses here.**
6. **Prioritize** — recent-change mapping per `references/journey-map.md`; fallback to all journeys.
7. **Walk** — per journey and stance per `references/stances.md`; capture per `references/evidence-rules.md`; stop before any step the data policy forbids; respect the budget.
8. **Compare competitors** — when configured; public pages only.
9. **Draft findings and actions** — one finding per defect across stances; severity per `references/severity-rubric.md`; producer-side dedupe marking; `kind` and `disposition` per action; `note` and `wontfix` actionless.
10. **Write and validate the artifact** — `references/artifact-template.md` and `references/findings-artifact.md`; a validation failure is a producer bug: fix and re-validate, never hand off an invalid artifact.
11. **Emit the receipt** — `references/receipt-dogfood.md`; outcome fixed here.
12. **Hand off** — `references/handoff.md`: discover, invoke, wait, one `publication:` line.
13. **Persist** — `findings.persistence`; **S3: confirm the branch commit and push in interactive runs.**
14. **Report** — plan versus walked, findings by severity, artifact path, branch and PR command, downgrades, the `publication:` line last.
15. **Safety rules** — never enter or print credentials; never submit state-changing forms under `observe-only`; never perform destructive or payment actions on production; never file issues or open or merge pull requests; never exceed the budget; never mutate anything before S1 passes.

`references/`: `stances.md` (fresh stance library), `journey-map.md` (journey schema, tags, recent-change mapping), `auth-modes.md` (the four modes and fallbacks), `evidence-rules.md` (capture, tiers, naming, embedding, retention, third-party rules), `severity-rubric.md` (default scale in user-impact terms), `artifact-template.md` (dogfood profile of the artifact with the summary layout), `receipt-dogfood.md` (`receipt.dogfood` v1), `handoff.md` (producer handoff and the `publication:` vocabulary), `findings-artifact.md` (generated from #18's canonical source; drift-tested), `protocol-contracts.md` (generated: produces `findings` 1, `receipt.dogfood` 1; accepts `config` 1), `setup.md` (generated from the capability spec), `runtime-notes.md` (browser adapter differences only). No `NOTICE.md`; nothing upstream is copied.

## Acceptance scenarios

1. **Given** `readiness.url` returning 503, **when** `dogfood` runs, **then** no browser session opens, no file is written, and the receipt is `target-unhealthy` with the status observed.
2. **Given** `environment: production` and `allow_production` absent, **when** it runs, **then** it stops at preflight with `preflight-failed`, names the key, and states that nothing was touched.
3. **Given** `auth.mode: attach-to-user-session` and no attach capability in an unattended run, **when** it runs, **then** `auth` journeys are skipped with reason `auth-unavailable`, the others are walked unauthenticated, the summary and receipt disclose it, and the outcome is `partial`.
4. **Given** `dogfood.journeys` empty, **when** it runs, **then** it stops with the setup block naming `dogfood.journeys` before any browser session.
5. **Given** a prior artifact recording `main@a71f03b` and commits since then touching only `packages/pricing/**` mapped to `checkout`, **when** it runs with `max_journeys: 1`, **then** `checkout` is the journey walked and the summary states the mapping.
6. **Given** no `changed_paths` on any journey, **when** it runs, **then** all journeys are walked in config order within the budget and the fallback is disclosed.
7. **Given** `competitors: []`, **when** it runs, **then** the artifact contains no competitor heading and no third-party URL.
8. **Given** `repository.visibility: private` and `embed: auto`, **when** a finding is written, **then** evidence appears as paths only, no image is inlined, and the paths exist under `evidence.ui.dir/<artifact_id>/<finding-id>/`.
9. **Given** `publish-findings` not discoverable, **when** the run finishes, **then** the artifact is committed on `findings/<artifact_id>`, the receipt outcome is `completed` or `partial`, and the last line is `publication: not-attempted (not-installed)` with the install command.
10. **Given** the publisher installed and returning `preflight-failed` (a missing label), **when** the run finishes, **then** the dogfood outcome is unchanged, the artifact and evidence are committed, and the last line is `publication: failed (preflight-failed: ...)`.
11. **Given** a finding whose fingerprint matches a `filed` row with an open issue, **when** the artifact is drafted, **then** the finding carries `likely_duplicate_of` with the URL, its action body opens with the likely-duplicate line, and the publisher (not dogfood) records `duplicate`.
12. **Given** a completed run, **when** `publish-findings` validates the artifact, **then** no `E-*` fires; in particular `E-SHORT-ID`, `E-FINGERPRINT`, `E-SECTION-ORDER`, and `E-ID-GAP` pass, and every extension field yields only `W-UNKNOWN-FIELD`.
13. **Given** `data_policy: observe-only` and a journey tagged `mutating`, **when** prioritization selects it, **then** it is skipped with reason `data-policy`, and no state-changing form is submitted anywhere in the run.
14. **Given** `revision_source: none`, **when** the artifact is written, **then** its summary's first lines contain `Revision: unknown; assumed <branch>@<sha> ... (unverified)` and the receipt's `target.revision` matches.
15. **Given** `max_minutes: 60` reached after the third journey's first stance, **when** the walk in progress ends, **then** no new walk starts, the summary lists the unwalked journeys and stances, and the outcome is `partial`.
16. **Given** `findings.persistence.commit: leave-uncommitted`, **when** the run finishes, **then** no branch or commit exists, and the report lists the artifact, evidence, and receipt paths to commit.

## Risks

- **Real user data.** An attached session is the human's own account. Mitigations: `observe-only` default, `mutating` tags, the production rule set, no screenshots of credential fields or third-party personal data. Residual: a journey author who omits `mutating`; #16's pilot reviews journey maps for it.
- **Destructive actions.** Configuration cannot lift the production prohibitions; a step that would violate them is refused and recorded. Residual: ambiguous UI (a "Remove" that deletes more than the coupon); walks stop at the review screen for anything financial.
- **Cost and time.** Bounded by `dogfood.budget` (4 journeys, 2 stances, 60 minutes); browser sessions close on abort; retention prunes the walk tier.
- **Credential leakage through tooling.** `provided-test-account` depends on the browser tool honouring the by-reference rule; `references/auth-modes.md` requires a fill step whose output is not echoed, and #15 adds a transcript scan for the reference's value.
- **Assumed revision** can still be wrong when the deployment lags the default branch; teams that care configure `revision_source`.
- **Competitor material** stays in the walk tier by construction; a human editing the artifact could still add it. Documented in `references/evidence-rules.md`.
- **Runtime browser variance** may make the same journey yield different evidence on Codex and Claude Code; the summary records runtime and provider so a reviewer can tell.

## Open items

1. **Maintainer ratifies:** `commit-on-branch` with `push: true` as the default persistence; the budget defaults; `dogfood.auth.mode` superseding `browser.auth.boundary`; the `receipt.dogfood` family name (see item 3).
2. **#18 confirms** that the fingerprint uses a finding's effective subject (per-finding `subject` when present), which this spec relies on for stable cross-run identity inside multi-journey artifacts.
3. **#12 alignment:** `research-project` will define a producer receipt of the same envelope; if the two are near-identical, #8 may register one `receipt.producer` family instead of `receipt.dogfood` and `receipt.research`. Until then each family stands alone.
4. **#8 absorbs** the PROPOSED keys above and decides whether `dogfood.stances` custom entries may override the shipped stance ids or only add to them (recommendation: add only).
5. Whether `question` findings should ever produce `human-decision` actions from dogfooding, given the publisher's warning about flooding the human-gate frontier. Recommendation: only with `--ask-gates`; default off.
6. A scripted validator (publication spec open item 6) would let step 10 run deterministically; until it exists the skill applies the rules from `references/findings-artifact.md` by hand.

## Consequences

- **#12 (`research-project`):** keep the producer structure parallel: one artifact per run, per-finding `subject` overrides, producer-side dedupe marking, the same end-of-run order (artifact → validate → receipt → hand off → persist → report), the same `publication:` line, and a receipt with the same envelope and success semantics. It shares `findings.persistence` and may reuse `references/handoff.md` as generated text.
- **#15 (validation and pilot strategy):** tests the sixteen scenarios, artifact validation against the publisher's rules, fingerprint derivation over the effective subject, the credential-reference transcript scan, evidence-tier separation (nothing under `state/local/` referenced from a committed artifact), drift of the generated references, and the hygiene allowlist for `example.com`, `acme/widgets`, and `analytics.example.com` used here.
- **#16 (pilot consumer and acceptance scenarios):** the pilot needs a deployed staging target with a health endpoint, two or more journeys with `changed_paths`, one `auth` journey with a test account reachable by reference, and `revision_source: endpoint` so both the proven and assumed revision paths are exercised; it reviews journey maps for missing `mutating` tags.
- **#17 (release sequencing and readiness gates):** `dogfood` ships with or after `publish-findings`, never before; the README presents `dogfood` as complete without the publisher, gives the paired install command, documents the persistence default and opt-out, and names browser providers only through `capabilities.runtime_notes`.
- **#8 (capability configuration):** absorbs the PROPOSED keys, extends the Tier 3 dogfood row, and rewords "file directly when the publisher is absent" as #18 recommends.
- **#18 (findings publication):** unchanged; consumed exactly, with one clarification requested in open item 2.

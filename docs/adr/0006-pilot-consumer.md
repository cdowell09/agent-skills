# ADR 0006: Pilot consumer and acceptance scenarios

Date: 2026-09-02
Status: Proposed (maintainer ratifies the repository choice)
Resolves: [#16 Choose the pilot consumer and acceptance scenarios](https://github.com/cdowell09/agent-skills/issues/16)
Inputs: [ADR 0001](0001-npx-skills-distribution-contract.md), [ADR 0002](0002-skill-boundaries-and-shared-contracts.md), [ADR 0003](0003-ticket-decomposition-provider.md), [ADR 0004](0004-licensing-attribution-and-contribution.md), [Capability configuration](../specs/capability-configuration.md), [Work execution](../specs/work-execution.md), [Human gate](../specs/human-gate.md), [Findings publication](../specs/findings-publication.md), [Dogfooding](../specs/dogfooding.md), [Project research](../specs/project-research.md), [`npx skills` compatibility](../research/2026-07-12-npx-skills-compatibility.md), a survey of the maintainer's public repositories (2026-09-02)

Question: Which adjacent software repository should be the first consumer, which catalog skills should it exercise, and which concrete successful workflows count as pilot acceptance, without that repository becoming a hidden hard-coded dependency?

Skill names used here (`pick-up-work`, `work-the-board`, `pick-up-human-task`, `work-the-human-board`, `dogfood`, `research-project`, `publish-findings`) are working names; a later naming pass may rename them. ADR 0005 (validation and pilot strategy, #15) owns the automated checks and the synthetic fixture; this ADR owns the real-project pilot only.

## Decision

1. **Two pilots, one primary.** [`cdowell09/math-bowl`](https://github.com/cdowell09/math-bowl) (React, TypeScript, Vite, Vitest, Node 24; public Vercel deployment with no sign-in) is the primary pilot and runs every wave. [`cdowell09/herdr-pr-board`](https://github.com/cdowell09/herdr-pr-board) (Go 1.24 terminal plugin; gofmt, vet, race, Python and shell checks; CI required on `main`) is the secondary pilot for the shared contracts, work-execution, and publication waves, one real human gate, and research under Codex with subagents unavailable. It has no web surface, which is the point: it proves browser control and UI evidence are optional and that configuration is not shaped around a JavaScript web application.
2. **Dogfooding is piloted on the primary only**; the secondary waives that wave ("no deployed web target"). Authenticated journeys are waived on both, because neither repository has a sign-in (open item 4).
3. **The primary's empty backlog is seeded through the catalog's own decomposition path**: one parent plan issue, `/to-tickets` under the ADR 0003 overlay, eight native sub-issues with blocked-by edges, two `needs-decision` gates, one `ready-for-human` item, plus one hand-made legacy `ready-for-human` issue for the migration scenario. Seeding is the first measurement of ADR 0003's overlay-accuracy switching metric.
4. **Acceptance is a per-wave checklist** mapped to spec scenario numbers with pass conditions, runtime assignment, and evidence rules. A wave is pilot-accepted only when its mandatory rows pass on both pilots, or the second pilot is waived for that wave with a written reason.
5. **Anti-coupling is enforced**: no pilot identifier in `skills/**` or `README.md`; pilot configuration lives only in each pilot's `.agent-skills/config.yaml`; ADR 0004's hygiene scan denylists both pilot names and the deployment host, with `docs/pilots/` as the single allowlisted place for evidence links; catalog tests never target a pilot; every pilot finding closes as a generic catalog fix or a pilot configuration change.

I agree with the survey's expected outcome. Verifying the public pages changed details, not the choice: math-bowl's CI is `npm test` and `npm run build` only, its labels are GitHub's defaults, it has no glossary, ADR convention, or version endpoint, and its one server function calls a paid third-party model. Each is handled by seeding or a waiver; none justifies a candidate without tests, CI, or a deployment.

## Context

The specs' "consequences for #16" bullets together require a repository with Issues, a Project with Status and Priority fields, native dependency edges, the disposition labels, tests and CI that can fail, PR flow, both runtimes (one without subagents), a UI surface, a legacy `ready-for-human` issue, a dependency graph with a leverage tie, a deployed target with readiness and revision signals and mapped journeys, a research areas file and seeded ledger, web egress, and public visibility. `CONTEXT.md` excludes the source project as a consumer; the first release targets software repositories on GitHub only.

## Selection criteria and scoring

Scores: 2 meets, 1 partly or fixable inside the pilot, 0 fails. Only public repositories could be inspected; `book`, `milos-monthly-cookies`, `joule24`, and `cv-christian-dowell` were not, and the source project and its companion are excluded by the map.

| Criterion | math-bowl | herdr-pr-board | chess | blog |
| --- | --- | --- | --- | --- |
| C1 Real software repository the maintainer owns | 2 | 2 | 2 | 0 (content site) |
| C2 Public visibility | 2 | 2 (MIT) | 2 | 2 |
| C3 Issues and tracker conventions | 1 (0 open; default labels) | 2 (`ready-for-agent`, `in-progress` in use; 1 open issue, 1 open PR) | 1 | 1 |
| C4 Project with Status and Priority fields creatable | 2 | 2 | 2 | 2 |
| C5 Tests and CI checks that can fail | 2 (Vitest, build) | 2 (gofmt, `git diff --check`, `go test`, Python tests, race, vet, `bash -n`, multi-platform build) | 0 | 0 |
| C6 PR flow and branch protection | 1 (PR CI; protection not visible publicly) | 2 (closed #15 "require CI before changes reach main") | 1 | 1 |
| C7 Deployed web target without credentials | 2 (Vercel; daily smoke workflow; no version endpoint) | 0 (terminal UI) | 2 | 1 (not a product) |
| C8 Docs conventions for the human-gate worker | 1 (`CLAUDE.md`, `AGENTS.md`, `DESIGN.md`, `PRODUCT.md`, `docs/plans/`; no ADRs or glossary) | 1 (`AGENTS.md` completion gates, `CONTRIBUTING.md`, `SECURITY.md`, `docs/releasing.md`, `docs/troubleshooting.md`; no ADRs) | 1 | 1 |
| C9 Both runtimes usable | 2 | 1 (`AGENTS.md` only; `CLAUDE.md` import needed) | 2 | 1 |
| C10 UI surface for evidence | 2 | 0 (deliberate) | 2 | 0 |
| C11 Stack diversity against the primary | – | 2 (Go, no Node) | 0 | 1 |
| Total | 17 | 16 | 13 | 8 |

`chess` fails C5: without tests or CI the work-execution wave cannot exercise failing checks, retries, or verification commands, and it duplicates the primary's stack. `blog` is not a software product. The two winners are complementary: the primary covers everything web-shaped; the secondary covers strict CI, no browser, non-Node, and brings real tracker history.

## Waves, skills, and pilot assignment

| Wave | Skills | math-bowl | herdr-pr-board |
| --- | --- | --- | --- |
| 0 Contracts, installer, setup, seeding | every skill's generated `setup.md` and `protocol-contracts.md`; installer commands | both runtimes; seeding | both runtimes; labels `needs-decision`, `needs-info`, `wontfix`, `ready-for-human`, `finding` added; real backlog |
| 1 Work execution | `pick-up-work`, `work-the-board` | Claude Code primary (subagents, PR monitoring, browser); one wave on Codex | Codex primary (sequential fallback, coordinator-owned monitoring); one wave on Claude Code |
| 2 Human gate | `pick-up-human-task`, `work-the-human-board` | full sitting on seeded gates, both runtimes | worker only, on one real decision-shaped issue, Codex; coordinator waived (a one-gate frontier is not a sitting) |
| 3 Ticket decomposition (ADR 0003) | overlay and `to-tickets` via the human-gate worker and the publisher | seeding (wave 0) and one split (wave 2) | waived (nothing to decompose); `provider: none` and `manual` paths run here |
| 4 Findings publication | `publish-findings` | producer handoff from `dogfood` and `research-project` | standalone human-written artifact; `provider: none` deferral |
| 5a Dogfooding | `dogfood` | Claude Code with browser attach; Codex with a launched browser if available | waived: no deployed web target |
| 5b Research | `research-project` | Claude Code, isolated subagents | Codex, `capabilities.subagents: unavailable` |

### Seeding the primary backlog

1. **Pilot setup PR** in the pilot repository: `.agent-skills/config.yaml` with every required key (verification `npm test` and `npm run build`; `pr.state: ready`; `milestones.policy: none`; `approval.mode: interactive`; readiness at the deployed root; `revision_source: none` until S1 lands; `human.domain_docs.adr_dir: docs/adr`); `state/.gitignore`; an empty `docs/adr/README.md` so the human-gate worker finds an ADR convention; `docs/research/areas.yaml` with three areas (generators, tutor API, results and timing) and a ledger seeded inside and outside 180 days; the seven labels; a user-owned Project with Status and Priority fields.
2. **Provider install**: `npx skills add mattpocock/skills --skill to-tickets --skill setup-matt-pocock-skills --agent codex --agent claude-code --yes`, then `/setup-matt-pocock-skills` for GitHub with label strings equal to the config.
3. **Parent plan issue** "Session history and deployed-revision visibility", with a `## User stories` section so traceability is exercised, then `/to-tickets #<parent>` with the ADR 0003 overlay pasted verbatim from the canonical `disposition-policy.md` source. Expected slices:

   | Slice | Disposition | Blocked by | Role in the pilot |
   | --- | --- | --- | --- |
   | S1 Expose the deployed commit at `/api/version` | agent-doable | – | unblocked; no UI (evidence `none`); enables `revision_source: endpoint` |
   | S2 Persist finished practice sessions locally | agent-doable | – | unblocked; with S1, a wave of two |
   | D1 Decide history-view scope (per grade or per child profile) | human-decision | – | gate blocking S3 |
   | D2 Decide retention and clearing policy | human-decision | – | gate blocking S4, S5 |
   | S3 History view on the results screen | agent-doable | S2, D1 | UI surface; work after a decision |
   | S4 Clear-history control with confirmation | agent-doable | S3, D2 | second-level dependency |
   | S5 Document storage and retention in `PRODUCT.md` | agent-doable | D2 | docs-only; makes D2 (transitive 2, direct 2) outrank D1 (transitive 2, direct 1): the tie-break case |
   | S6 Show the deployed revision in the footer | agent-doable | S1 | merge boundary behind S1's PR |
   | H1 Make the `validate` check required on `main` | human-execution | – | `ready-for-human`; never selected; satisfies C6 |

4. **Legacy issue** L1 "Decide whether `/api/tutor` needs rate limiting", hand-labelled `ready-for-human` before `needs-decision` is configured, so the migration scenario has something to classify.
5. **Record** the quiz's Disposition lines, mislabels, and the post-publication edge check (sub-issues and blocked-by on every slice, repairs) in `docs/pilots/math-bowl/0-contracts.md`.

Post-seed: two agent-eligible issues, three gates (after migration), one human-execution item, four blocked slices: enough for every wave 1 and 2 scenario without contrivance.

## Acceptance scenarios

Notation: MB = math-bowl, HB = herdr-pr-board, CC = Claude Code, CX = Codex, both = each runtime once; cap, we, hg, pub, df, rp = scenario numbers in the capability-configuration, work-execution, human-gate, publication, dogfooding, and research specs. Rows are mandatory unless listed under "Optional". Evidence is a link into the pilot repository or a receipt excerpt in `docs/pilots/` with opaque IDs, hostnames, and logins removed. "Pass when" is the whole condition; anything less fails.

### Wave 0: installer lifecycle (ADR 0001 release verification)

| ID | Scenario (exercises) | Where | Pass when | Evidence |
| --- | --- | --- | --- | --- |
| I1 | `--list` discovery (ADR 0001) | MB, HB | every skill listed with its description; no root `SKILL.md` | output |
| I2 | Exact selection `--skill <name> --agent codex --agent claude-code --yes` | MB, HB, both | only that skill under `.agents/skills/` with symlinks from `.claude/skills/`; one `skills-lock.json` entry | tree, lock diff |
| I3 | `--copy` mode | MB (CC), HB (CX) | independent copies; skill runs identically | tree, receipt |
| I4 | Update from the default branch after a published change | MB, HB | `npx skills update` picks it up; lock hash changes | lock diff |
| I5 | Remove, including the project-lock caveat (compatibility research) | MB, HB | agent directories cleaned; stale lock entry recorded as fact; if confirmed, upstream issue filed and cleanup documented | lock file, issue link |
| I6 | Coordinator without its worker (we 4, hg 12) | MB (CC), HB (CX) | both coordinators stop before any claim, printing the paired install command; after pairing, the `metadata.contracts` check passes | stop message, receipt |

Optional: `--global` install shadowed by a project install (MB, CC).

### Wave 0: setup and shared contracts

| ID | Scenario (exercises) | Where | Pass when | Evidence |
| --- | --- | --- | --- | --- |
| C1 | No config, interactive `pick-up-work` (cap 1) | MB, HB, both | guided setup writes `config.yaml`, `state/.gitignore`, the validation receipt; no GitHub or tree mutation before the receipt | commit, receipt `checks` |
| C2 | No config, unattended `work-the-board` (cap 2, we 1) | MB (CC), HB (CX) | setup block, nothing claimed, `setup-required` | output |
| C3 | Comment-only config edit (cap 3) | MB | fast path; zero `gh` calls in the read-only phase | output |
| C4 | Status option renamed to a nonexistent value (cap 4) | MB | Tier 1 re-runs, stop names `tracker.statuses.in_progress` | stop block |
| C5 | `version: 2` config (cap 13) | MB | stop names the mismatch and the update | stop block |
| C6 | Label agreement with upstream `triage-labels.md` (cap Tier 3; ADR 0003) | MB | check passes; after one string is changed it fails naming the file | receipt, stop block |

Optional: `gh` login differs from the receipt (cap 5; HB, logins redacted).

### Wave 1: work execution

| ID | Scenario (exercises) | Where | Pass when | Evidence |
| --- | --- | --- | --- | --- |
| E1 | Wave of two parallel workers (we 8; claim protocol; `receipt.work`) | MB (CC), HB (CC); one wave each on CX | preview prints filters and effective cap; claims `/w1`, `/w2`; two open PRs each closing its issue; coordinator receipt lists both children | preview, claim comments, PRs, receipt |
| E2 | Claim collision (we 3, cap 6) | MB | a hand-posted foreign claim yields `skipped-claimed`, nothing posted, next item dispatched | comment, receipt |
| E3 | Failing check, `fix-once`, then exhaustion (we 6) | HB (gofmt failure), MB (failing Vitest case) | first fix run `checks-fixed`; after `max_retries`, `retries-exhausted`, comments on PR and issue, released `failed`, `In Review`, no re-dispatch | workflow runs, comments, receipt |
| E4 | Merge boundary (we 13) | MB (S6 behind S1) | S6 reported as merge-boundary while S1's PR is open, never dispatched; dispatched after merge | previews before and after |
| E5 | PR merged between runs (we 5) | MB, HB | released `merged`, issue closed, `Done`, worktree removed | receipt, timeline |
| E6 | UI evidence present and absent (we 7) | MB (CC, browser, S3); HB (no browser) | MB PR inlines a screenshot (public repository, `embed: auto`); HB PR and receipt carry `evidence: none` with the reason | PR bodies |
| E7 | Unattended thin spec (we 10) | HB (an underspecified issue) | one questions comment, `needs-info`, released, no branch | comment, receipt |
| E8 | Stranded claims after a killed coordinator (we 12) | MB | next run recovers child claims via the local ledger, listed under `recoveries` | receipt |
| E9 | Sequential fallback, coordinator-owned monitoring (we 16) | HB (CX) | workers one at a time; downgrade and monitoring owner `coordinator` with reason in the receipt | receipt |
| E10 | Filter echo with `tracker.milestones` absent (we 9) | MB, HB | preview shows `milestone: none` as a declared default | preview |

Optional: closed unmerged PR (we 15) and foreign commit on a conflicted branch (we 17), both on HB.

### Wave 2: human gate

| ID | Scenario (exercises) | Where | Pass when | Evidence |
| --- | --- | --- | --- | --- |
| G1 | Legacy label migration (hg 1) | MB (L1) | setup offers `needs-decision`, walks L1, relabels on confirmation; H1 never selected before or after | transcript, label history |
| G2 | Two gates in one sitting including one split (hg 4, hg 8 with `cap: 2`) | MB, CC | D1 `cleared-for-agent` (S3 gains `ready-for-agent`); D2 split via overlay and `/to-tickets`, native edges verified and re-pointed; coordinator stops at the cap and prints the handoff naming the unblocked issues | decision comments, sub-issues, receipts, report |
| G3 | Leverage order and tie-break (hg 7) | MB | order D2, D1, L1; a second run reproduces it | two previews |
| G4 | Closed gate releases the agent frontier (hg 11) | MB | after `state_reason: decided`, the blocked issues appear in the next `work-the-board` preview | previews |
| G5 | Human-driven invocation (hg 13) | HB, CX, the real gate | with `interactive_skill_invocation: unavailable` the coordinator claims, prints the worker command, waits for the run receipt, reconciles it | command, receipt file |
| G6 | Split with the provider absent (hg 5) | HB (`provider: none`, a synthetic gate the maintainer opens) | worker creates native sub-issues with one disposition label each, or `manual` records slices and returns `deferred` | sub-issues, receipt |
| G7 | No status mirror by default (hg 14) | MB | every gate stays `Todo` | Project history |

Optional: two sessions racing (hg 2) and skip-set persistence across interruption (hg 6), both on MB.

### Wave 3: ticket decomposition (rows filled from waves 0, 2, 4)

| ID | Scenario (exercises) | Where | Pass when | Evidence |
| --- | --- | --- | --- | --- |
| D1 | Overlay accuracy on seeding and on the G2 split (ADR 0003 switching metric) | MB, both (seeding on one runtime, split on the other) | at most one in ten slices mislabelled or missing a Disposition line per runtime; otherwise the fallback extension is triggered | quiz transcripts, labels |
| D2 | Native edge verification and repair (ADR 0003) | MB | sub-issue and blocked-by sets match the approved breakdown; repairs in the receipt | `gh issue view --json`, receipt |
| D3 | Codex honours `allow_implicit_invocation: false` (ADR 0003 open item 4) | MB, CX | the worker prints the `/to-tickets` command rather than invoking it; the contrary is recorded for #10's runtime note | transcript |

### Wave 4: findings publication

| ID | Scenario (exercises) | Where | Pass when | Evidence |
| --- | --- | --- | --- | --- |
| P1 | Producer handoff files one simple action, defers one complex, then hands off (pub 7, 8) | MB, CC, the F1 artifact | with `provider: none`: `filed` with markers and `deferred` `decomposition-unavailable`, action untouched; after `provider: to-tickets`: `handed-off`, a `needs-decision` parent, overlay printed | issue links, receipt, `## Publication` lines |
| P2 | Dedupe on re-run (pub 4, 6, idempotency) | MB | second run files nothing; a new artifact repeating one fingerprint returns `duplicate` with the URL; a run interrupted after `gh issue create` recovers via the action marker | receipt rows |
| P3 | Standalone human-written artifact (pub 14) | HB, CX | validates and files like a producer artifact | artifact, issues |
| P4 | Unattended interactive mode (pub 9) | HB | every action `deferred` `approval-required`, no issue | receipt |
| P5 | Missing label stops before mutation (pub 12) | HB (before `finding` exists) | stop names the label and the `gh label create` command | stop block |

Optional: invalid artifact (pub 1) on HB.

### Wave 5a: dogfooding (primary only)

| ID | Scenario (exercises) | Where | Pass when | Evidence |
| --- | --- | --- | --- | --- |
| F1 | Full run: two journeys (grade selection and practice; results and tutor), two stances each (df 12; public-repository inline embedding) | MB, CC | no `E-*`; finding-linked evidence under `docs/evidence/<artifact_id>/`; walk tier stays under `state/local/`; commit on `findings/<artifact_id>` pushed, no PR | branch, artifact, receipt |
| F2 | Readiness failure (df 1) | MB | `readiness.url` at a nonexistent route: no browser session, `target-unhealthy` | receipt |
| F3 | Revision assumed, then proven (df 14) | MB | before S1: `Revision: unknown; assumed ...`; after S1 with `revision_source: endpoint`: `proven <sha>` | two summaries |
| F4 | Recent-change mapping (df 5) | MB | after S3 merges, the results journey is walked first and the summary says why | summary |
| F5 | Publisher not installed (df 9) | MB | artifact committed; last line `publication: not-attempted (not-installed)` with the install command | output |
| F6 | Data policy on the tutor journey (df 13) | MB | journey tagged `mutating` (paid third-party API); skipped under `observe-only` with reason `data-policy`; walked within budget under `scratch-account` | journey map, receipt |
| F7 | Journey-map review (df risks) | MB | the maintainer records that every state-changing or cost-incurring journey is tagged `mutating` | wave file note |

### Wave 5b: research

| ID | Scenario (exercises) | Where | Pass when | Evidence |
| --- | --- | --- | --- | --- |
| R1 | One area researched, ledger updated (rp 2, 13) | MB (CC), HB (CX) | never-researched area selected with the printed reason; artifact validates; one `## Sources` line per source; docs PR opened, not merged | PRs, ledger diff, receipt |
| R2 | Avoid-list inside and outside expiry (rp 3, 4) | MB | in-window source dropped `on-avoid-list`; out-of-window source re-scored with a new line | ledger diff, receipt |
| R3 | Subagents unavailable (rp 5, 6) | HB, CX | `branches.mode` `sequential` or `single-lens`, disclosed in receipt and summary | receipt |
| R4 | Publisher present and absent (rp 10, 11) | MB present, HB absent | research outcome unchanged; `publication:` line matches | outputs |
| R5 | Example artifact URLs re-verified (rp consequences) | MB | every URL in the spec's example fetches and supports its claim, or the fixture is amended | fetch log |

## Anti-coupling rules

1. **No pilot identifiers in the shipped surface.** `math-bowl`, `herdr-pr-board`, `Herdr`, the deployment host, and the seed issue titles never appear in `skills/**`, `README.md`, or generated references. ADR 0004's hygiene denylist (kept outside the public tree) gains these terms for those paths.
2. **One allowlisted place for evidence.** `docs/pilots/**` may name the two pilot repositories and the deployment host and nothing else the scan forbids; opaque IDs, logins, hostnames, tokens, and personal paths stay denied there too. #15 implements it as a path-scoped exception.
3. **Pilot configuration lives only in the pilot.** Each pilot commits its `.agent-skills/config.yaml`; the catalog links to it and never copies it. Catalog examples keep `acme/widgets` and `example.com`.
4. **Catalog tests never target a pilot.** #15's fixtures are a synthetic repository with recorded `gh` responses; no workflow here holds a pilot token or coordinate. A pilot run is readiness evidence, not a test.
5. **Two dispositions per finding.** A *catalog bug* is fixed generically with a linked issue and a synthetic-fixture test; a *pilot configuration change* is a commit in the pilot. A finding fixable only by special-casing the pilot is a design bug and returns to the owning spec.
6. **Second-consumer rule.** No wave is released until it has run on both pilots or the second is waived for that wave in the wave file with a reason. This ADR pre-approves the herdr-pr-board waivers for waves 3 and 5a and the wave 2 coordinator waiver. A waiver is a sentence, not a checkbox.
7. **No defaults inferred from the pilot.** A value both pilots share (`main`, `npm test`, `Todo`) enters the catalog only as an example or discovered value, never as a mutation-affecting default; the capability spec's no-default rule is what makes this checkable.

## Pilot logistics

- **Where.** `docs/pilots/<repo>/<wave>.md` in this catalog, one file per wave per pilot (`0-contracts.md` through `5b-research.md`; a waived wave is a file holding the waiver). Each records date, catalog commit, skill versions, the upstream `to-tickets` revision, runtime and browser provider, the rows with status and links, findings with disposition, and the verdict. Receipt excerpts drop `resolved.*`, `github.*`, and hostnames; no private data is copied.
- **Who.** The maintainer runs every interactive row; unattended rows may run in a remote agent session on the maintainer's account, recorded as such. Racing scenarios use a second terminal session, not a second person.
- **Order.** Waves in numeric order; within a wave the primary first (most scenarios, so catalog bugs surface early), then the secondary. Wave 3 has no run of its own. Wave 5b may precede 5a if the browser provider is not ready.
- **Pilot acceptance for #17.** A wave is pilot-accepted when every mandatory row passed on both pilots (or the second is waived), every "both" row ran on both runtimes, every finding is closed under rule 5 or deferred to a linked issue with the maintainer's sign-off, the wave file is merged, and the hygiene scan passes with the pilot denylist active. #17 treats this as one of two gates per wave; ADR 0005's automated checks are the other, and neither substitutes for the other.

## Alternatives considered

- **The source project.** Rejected: `CONTEXT.md` says it will not consume the extracted skills; it is private, so evidence cannot be public; and it is the one repository whose conventions the skills came from, so it cannot detect overfitting.
- **A synthetic fixture only.** Rejected as the pilot, kept as #15's test bed: no real CI latency, deployment, review, flaky checks, or human with something at stake, which is exactly what the installer caveat, merge boundaries, and the overlay under a real quiz need.
- **A purpose-built demo repository.** Rejected: a fixture with a deployment and no real backlog, and a temptation to shape the demo to the skills.
- **`chess` as primary.** Rejected on C5 and C11. **`blog` as secondary.** Rejected: content, not software; nothing to verify.
- **A single pilot.** Rejected: one JavaScript web application cannot show that browser control is optional or that `work.verification` is stack-neutral; the second pilot is the cheapest available proof.
- **An uninspected private repository.** Not selectable on public evidence (open item 1).

## Consequences

- **#15 (ADR 0005):** implements the `docs/pilots/**` allowlist and pilot denylist; keeps every automated test on the synthetic fixture; consumes the wave files as readiness input; tunes the fuzzy-title threshold and the accelerator decision on pilot data; takes I5 upstream if confirmed.
- **#17 (release sequencing):** adopts pilot acceptance as one of two gates per wave; ships each wave only after its mandatory rows pass (E1–E10, G1–G7, P1–P5, F1–F7 on the primary with the waiver recorded, R1–R5); states in release notes that dogfooding was piloted on one public, unauthenticated web application.
- **Specs #9, #10, #18, #11, #12:** their consequences for #16 are met by the assignments above, with two explicit gaps: no authenticated journey and no `revision_source: header` path (open item 4).
- **ADR 0003:** D1 is the first measurement of the composition-versus-extension criterion; D3 answers its open item 4.
- **ADR 0004:** denylist and allowlist change as in the anti-coupling rules; pilot licenses are irrelevant because nothing is copied.
- **Capability configuration (#8):** the seeding assumes `human.domain_docs.adr_dir` and `research.areas_file`, both already proposed; nothing new.

## Open items

1. **Maintainer ratifies** the two pilots, the pre-approved waivers, and the seed plan (slices may change; the counts and shapes must not). The private repositories `book`, `milos-monthly-cookies`, and `joule24` were not inspected; one with a sign-in and a health endpoint could later close item 4 without changing this ADR's structure.
2. **Branch protection on math-bowl** is not visible publicly; H1 adds it. Until then wave 1 runs on plain PR flow and the wave file says so.
3. **Deployment reachability** could not be verified from this session (its egress proxy blocks the Vercel host); the maintainer confirms the no-sign-in claim in wave 0 before committing `dogfood.target.url`. The daily smoke workflow suggests it holds.
4. **Unpiloted paths:** `dogfood.auth.mode` values other than `none`, and `revision_source: header`. They ship as specified but untested on a real target; #17 says so.
5. **The tutor journey costs money** and sends prompt text to a third party; it is tagged `mutating` and budgeted. The maintainer decides whether wave 5a runs it under `scratch-account` at all.
6. **The secondary's real gate.** The one open issue on herdr-pr-board was opened by another account; the maintainer confirms it may serve as the wave 2 gate or opens a decision-shaped issue of their own.
7. **Small pilot changes:** herdr-pr-board needs a `CLAUDE.md` importing `AGENTS.md` for Claude Code; math-bowl has no license file (irrelevant to the catalog, but worth adding before public evidence links point at it). Both are pilot configuration changes recorded in wave 0.

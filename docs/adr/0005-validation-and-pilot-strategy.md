# ADR 0005: Validation and pilot strategy

Date: 2026-09-02
Status: Proposed (maintainer ratifies)
Resolves: [#15 Define the validation and pilot strategy](https://github.com/cdowell09/agent-skills/issues/15)
Inputs: [ADR 0001](0001-npx-skills-distribution-contract.md), [ADR 0002](0002-skill-boundaries-and-shared-contracts.md), [ADR 0003](0003-ticket-decomposition-provider.md), [ADR 0004](0004-licensing-attribution-and-contribution.md), [ADR 0006](0006-pilot-consumer.md), [Capability configuration](../specs/capability-configuration.md), [Work execution](../specs/work-execution.md), [Human gate](../specs/human-gate.md), [Findings publication](../specs/findings-publication.md), [Dogfooding](../specs/dogfooding.md), [Project research](../specs/project-research.md), [`npx skills` compatibility research](../research/2026-07-12-npx-skills-compatibility.md)

Question: What automated checks, static release-hygiene scans, installer tests, runtime compatibility tests, behavioral scenarios, and real-project pilot evidence must demonstrate that each release wave is repository-agnostic and safe to publish?

Skill names used here (`pick-up-work`, `work-the-board`, `pick-up-human-task`, `work-the-human-board`, `dogfood`, `research-project`, `publish-findings`) are working names; a later naming pass may rename them. ADR 0006 owns the pilot repositories and their acceptance checklist; this document owns everything automated and the rules pilot evidence must follow.

## Context

A published skill is prose executed by an agent runtime plus a few scripts. Most of its behavior cannot be unit-tested, but the specs made the *contracts* precise: a config schema with a fingerprint, a claim comment, four receipt families under one envelope, a findings artifact with recomputable fingerprints, an append-only publication receipt, two frontier algorithms written as `gh` recipes, and a `metadata.contracts` convention. Those can be tested against real GitHub without an agent. What needs an agent (implementing an issue, walking a browser, grilling a human) is a small, nameable set that the pilot covers.

Verified from this session on 2026-09-02: `skills-ref` 0.1.5 on npm exposes `skills-ref validate <skill_path>`, the command the Agent Skills specification recommends; the `skills` CLI is at 1.5.23 and requires Node 22.20 or newer (the compatibility research reviewed 1.5.16, which required Node 18). The repository holds documents only. `docs.github.com` and `cli.github.com` were unreachable, so the minimum `gh` release with native dependency flags remains unverified.

The six specs define 95 Given/When/Then scenarios and name, in their "#15" consequences, the cross-cutting tests this document must own (drift, malformed receipts, crash injection, marker search, leverage order, rotation determinism, the shingle diff, the placeholder allowlist). ADR 0004 adds the hygiene scan, provenance consistency, notices drift, and the release gate; ADR 0003 the decomposition checks and the overlay metric; ADR 0006 a path-scoped `docs/pilots/**` allowlist, pilot names in the denylist, and the installer-caveat upstream report.

## Decision

- **Seven-layer pyramid, L0–L7.** L0–L3 are hermetic and run on every pull request; L4 runs the real installer against a synthetic consumer; L5 runs the contracts' `gh` recipes against a long-lived fixture repository; L6 is a maintainer-run runtime smoke matrix whose results are committed; L7 is pilot evidence in ADR 0006's format. Nothing below L6 needs an agent runtime or an API key.
- **Existing tools only:** `skills-ref validate` for specification conformance, the `skills` CLI itself for the lifecycle, `markdownlint-cli2`, Node's built-in `node:test` runner with no framework, and two catalog scripts (`scripts/generate-contracts.mjs`, `scripts/hygiene-scan.mjs`). Node is justified because the installer already requires Node 22, so every consumer and CI job has it; `node:test` avoids a framework the catalog would otherwise ship nothing else in.
- **Canonical protocol sources live in `contracts/`** at the repository root, are generated into each participating skill by one generator, and a byte-for-byte drift test blocks hand edits. The findings validator is one generated Node script, the reference implementation of the fingerprint rule the capability spec pins.
- **L5 proves the contracts, not the prose.** The harness executes the exact `gh` commands quoted in the contract files; a recipe-extraction test fails if it uses a command the contracts do not contain. Agent judgment is covered by L6 and L7 only.
- **Repository-agnostic proof rule:** every wave is green on the synthetic fixture *and* pilot-accepted (ADR 0006) on a pilot whose stack differs from the source project's. Pilot deviations become catalog tests at the lowest reproducing layer before the fix merges.
- **CI never touches a pilot.** The only mutating secret is a token for a fixture repository in a dedicated throwaway GitHub organization, and the harness refuses to mutate any repository outside it.

## 1. Validation pyramid

| Layer | Checks | Tooling | Runs | Blocks |
| --- | --- | --- | --- | --- |
| L0 Static | spec frontmatter; catalog frontmatter rules; reference and link integrity; Markdown lint | `skills-ref validate`, `tests/static/`, `markdownlint-cli2` | optional pre-commit; every PR; release | merge, release |
| L1 Hygiene | ADR 0004's seven classes plus `to-issues` and pilot terms; provenance consistency; notices drift; shingle diff | `scripts/hygiene-scan.mjs`, `tests/hygiene/` | pre-commit (no denylist); every PR (denylist when available); release (denylist required) | merge; release refuses without a class 1 run |
| L2 Drift | generated copies equal generator output; majors agree everywhere; companion pairs equal | `scripts/generate-contracts.mjs --check` | every PR; release | merge, release |
| L3 Contract | config, receipts, claim, artifact, publication receipt, ledger, frontier goldens, frontmatter compatibility | `node:test` over `tests/fixtures/` (recorded `gh` responses) | every PR; release | merge, release |
| L4 Installer | ADR 0001's lifecycle list on a synthetic consumer | the `skills` CLI, pinned | local source on every PR; GitHub source on `main`, nightly, tags | merge (local); release (GitHub) |
| L5 Behavioral | automatable scenarios against the live fixture | `node:test` plus `gh`, serial | `main`, nightly, `run-fixture` label, tags | release; a red nightly blocks the next release |
| L6 Runtime | Codex × Claude Code × skill × capability cells | `tests/smoke/`, maintainer-run, results committed | before each wave's release PR | release, via result freshness |
| L7 Pilot | ADR 0006's mandatory rows | `docs/pilots/<repo>/<wave>.md` | per wave | release, via pilot acceptance |

**L0.** `skills-ref validate skills/<name>` covers the specification (kebab-case `name` equal to the directory, at most 64 characters; `description` at most 1,024). `tests/static/` adds the catalog's rules: `license: MIT`; non-empty `compatibility`; `metadata.author`, semver `metadata.version`, `metadata.provenance ∈ {original, derivative}`; `metadata.third-party-notices: NOTICE.md` present exactly when provenance is `derivative` and the file exists; `metadata.contracts` in `name=major` form on every skill that embeds a generated protocol, equal to `contracts/manifest.yaml`; a coordinator's `compatibility` names its worker; no root `SKILL.md`; every file under `references/`, `scripts/`, `assets/` is referenced from `SKILL.md` and every reference resolves; no path escapes the skill directory or names a sibling. CI checks internal links only; an external link check runs nightly without blocking, because cited documentation hosts may be unreachable from a runner. `markdownlint-cli2` uses a small committed config.

**L1.** `scripts/hygiene-scan.mjs` implements ADR 0004's table as regular-expression classes. Scope: all classes plus `to-issues` and the ADR 0006 pilot terms over `skills/**`, `README.md`, and generated references; classes 2–7 over all of `docs/**`; class 1 over `docs/**` except `docs/research/**` (ADR 0004) and `CONTEXT.md`, which names the source project in its vocabulary (an exemption ADR 0004 did not list; maintainer ratifies; #17 may reword the entry so it can be dropped). `docs/pilots/**` is a path-scoped exception for the two pilot coordinates and the deployment host only (ADR 0006 rule 2). Any `.env*` file fails.

- *Denylist.* Class 1 patterns come from the file named by `AGENT_SKILLS_DENYLIST_FILE` (one case-insensitive regular expression per line); CI writes the `HYGIENE_DENYLIST` secret to a temporary file. Fewer than three entries fails as "denylist not loaded"; the maintainer keeps the excluded skill's name and the pilot terms in it. Without the variable (fork PRs, fresh clones) classes 2–7 run and class 1 reports `skipped`; the release job refuses without a class 1 run. New terms are added out of tree whenever a pilot, review, or the shingle diff surfaces one; `docs/provenance.md` records the date of each change without the term.
- *Allowlist.* `tests/hygiene/allowlist.yaml`, committed, each entry with a reason: `example.com` and subdomains; coordinates under the owner `acme`; `<...>` placeholders; `cdowell09/agent-skills`; the upstream and documentation hosts the ADRs and specs cite; the specs' worked short ids and `sha256:` fingerprints for the hexadecimal class. Additions go through a pull request.
- *Fail behavior.* Any hit prints file, line, class, and the match with denylist terms masked, and exits non-zero. No warning level.
- *Provenance.* `THIRD_PARTY_NOTICES.md` must equal the generator's union of `skills/*/NOTICE.md`; every register row has a pinned commit and verbatim notice. The shingle diff compares `skills/**` against the source package at `AGENT_SKILLS_SOURCE_DIR` for eight or more consecutive shared words plus the excluded skill's name, and is skipped with a visible notice when that directory is absent (only the maintainer's machine has it); the release job requires a `docs/provenance.md` entry recording a run against the tagged commit.

**L2.** Canonical sources live in `contracts/` at the root: the installer copies only `skills/<name>/`, so nothing there is installed, and ADR 0004 already exempts generated protocol text from notices.

```text
contracts/
├── manifest.yaml            # contract -> major, source, consuming skills, target path
├── claim-protocol.md        # claim v1 (capability spec)
├── setup.md                 # setup flow and tiered validation (capability spec)
├── frontier-rules.md        # work-execution frontier v1
├── post-pr-lifecycle.md     # work-execution lifecycle rules
├── receipt-work.md          # receipt.work v1
├── leverage-rules.md        # human-gate frontier and leverage v1
├── receipt-human.md         # receipt.human v1
├── disposition-policy.md    # ADR 0003 policy and overlay
├── findings-artifact.md     # findings v1, codes, producer handoff, pinned fingerprint rule
├── handoff.md               # producer handoff text shared by dogfood and research-project
└── scripts/findings-validate.mjs   # validator and fingerprint; generated into the three findings skills
```

`scripts/generate-contracts.mjs` writes each source into every listed skill under `references/<file>` or `scripts/<file>` with a header `<!-- generated from contracts/<file> sha256:<12 hex>; do not edit -->` (a `//` comment for scripts) and writes each skill's `references/protocol-contracts.md` from the majors it produces and accepts. `--check` regenerates into a temporary directory and byte-compares; a difference, a missing copy, or a copy without a manifest entry fails. A second test asserts the major in each source's heading equals the manifest, every consumer's `metadata.contracts` lists exactly the majors it embeds, and both skills of a companion pair declare identical majors. *Version rules:* additive fields change the hash only and the generator rewrites copies; renaming, removing, or re-typing a field bumps the major in the manifest, the source heading, and every consumer's `metadata.contracts` and `metadata.version` in one pull request (L2 fails until all agree), and requires an L3 fixture for the previous major so "higher major stops with guidance" stays tested. `config.version` follows the same rule and is bumped only by the capability spec.

**L3.** Hermetic tests in `tests/contract/` over fixtures and recorded `gh` responses in `tests/fixtures/`, with parsers reimplemented from the contract text in `tests/lib/`; the findings validator is the generated script run as a child process.

- *Config.* The spec's minimal `pick-up-work` config validates; removing each required key, per skill and the required-keys matrix, names exactly that key in the stop block; the fingerprint is stable under comment and whitespace edits and changes for any value or contract-major change; `version: 2` stops; unknown keys warn; path-escape, duration, status-ambiguity, `browser.auth.boundary` alias, `allow_production`, and both-`research.areas`-forms rules. Fixtures follow the reconciled key registry.
- *Validation receipt.* Round-trip; fast path within `validation_max_age`; each invalidation condition; identity failures invalidate, transient ones do not; `state/.gitignore` written.
- *Claim.* Parse and format; owner-id grammar with coordinator suffixes; active-claim selection among several comments; expired, released, superseded, and release-marker forms; malformed input (no marker, no fence, unknown major or `state`) is held-by-unknown; idempotency by prefix; abandonment including the stranded fast path; unknown dispositions never stop a reader.
- *Receipts.* All four families under the common envelope (marker block and identical JSON run file): required fields, every outcome, `claim` values, owner prefix; unknown family, major, outcome, or owner stops; a missing block or field reconciles as `failed` with `no-receipt` or `malformed-receipt`; additive fields ignored. Frontmatter compatibility: equal, higher (stop with the paired update command), lower with rules (warn).
- *Findings validator.* The three worked artifacts (publisher, dogfood, research) pass with only `W-UNKNOWN-FIELD`; one fixture per `E-*` code fails with exactly that code; `artifact_short_id` and every fingerprint recompute under the rule the capability spec pins (effective subject; punctuation deleted without substitution); the first run confirms the specs' worked values, and a mismatch corrects the examples, never the rule. Action hash, actions digest, and both `E-REVISION-*` codes against a receipt fixture.
- *Publication receipt and ledger.* Crash injection after every write boundary: `runs` and `rows` only grow, the header survives, a re-run touches only actions without a terminal row; approval binding void on `revision` or `action_hash` change; every reason code; ledger sections append-only; avoid-list derivation with expiry; area and lens rotation identical from two copies of one ledger.
- *Frontier goldens.* Work-execution order per `work.priority.order` strategy and the filter echo; human-gate leverage total order on a graph with a cycle, a closed intermediate, and the spec's A/B/C tie; effective concurrency `min(max_parallel, capacity, --max)`.

**L4.** `tests/installer/` builds a synthetic consumer in a temporary directory: `git init`, a `Makefile` with `check:` and a `pyproject.toml`, deliberately not a Node project, so setup's command discovery meets a stack unlike the source project's. `HOME` is redirected so `--global` and the global lock are sandboxed. The `skills` CLI version is pinned in `package.json` and recorded in release notes. Source is the checkout path on pull requests and `cdowell09/agent-skills#<ref>` (the pushed branch on `main`, the tag on releases) otherwise. Assertions, in ADR 0001's order: `--list` shows exactly the published names; `--skill <name> --agent codex --agent claude-code --yes` creates `.agents/skills/<name>/` with full `references/` and `scripts/` trees and a `.claude/skills/<name>` symlink; `--copy` yields two independent directories; `--global` writes under the sandboxed global locations; update installs from the previous tag, runs `skills update`, and asserts the folder matches the default branch; remove deletes both agent locations and the canonical copy and records whether `skills-lock.json` keeps a stale entry (the test encodes the observed upstream behavior so a change in either direction surfaces; if confirmed, #15 files the upstream report ADR 0006 assigns it); companion selection installs `work-the-board` alone and checks the README's paired command and the coordinator's `compatibility` field (the runtime stop is L6). Bare `--yes` is never run.

**L5.** *Fixture:* a long-lived repository `agent-skills-fixture` in a dedicated throwaway GitHub organization (name: maintainer ratifies) with Issues, a Project carrying Status and Priority, the seven label roles, and a trivial Actions check that file content can fail. Long-lived beats throwaway: a Project and labels are slow to create per run, the token can be scoped to one repository, and the run history is evidence. Each run prefixes titles with `[run <id>]` and labels `fixture-run:<id>`; a sweeper closes leftovers older than one hour at job start and everything of its own run at job end (issues `not_planned`, PRs closed, branches deleted, items archived). A run stays under 500 REST calls; jobs are serialized by a concurrency group. *What runs:* the contract recipes for claim, release, supersede, and release-marker comments; frontier computation with the exact `gh issue list`, `gh project item-list`, and dependency probes in transport order; stale-mirror reset; stranded-claim recovery from a fabricated run ledger; PR reconciliation with PRs merged, closed, and conflicted, including a foreign-author commit; dedupe searches (receipts, exact markers in open and closed issues inside and outside the window, fuzzy titles); filing with role-mapped labels and exactly one disposition label; parent proposal issues; Project placement; action-marker recovery; native sub-issue and blocked-by writes and repair; label agreement with upstream's `triage-labels.md`; `gh auth status` against a receipt for another login. A `gh` shim on `PATH` forwards to the real binary except for scripted failures (two 502s, a missing dependency flag, a rate limit), making retry and transport-downgrade scenarios deterministic. The recipe-extraction test parses every fenced `gh` command in `contracts/*.md` and fails if the harness issues a command whose shape is absent. *Not here:* anything needing an agent's judgment or a browser.

**L6.** Matrix `{codex, claude-code}` × seven skills × `{all optional capabilities present; one optional capability unavailable, per relevant capability; one required capability absent}`. Absence is simulated with `capabilities.<name>: unavailable` in the consumer config, or by removing the tool from `PATH` for required ones. **Compatible** means: preflight passes, or stops with the setup block and zero mutations when a required capability is absent; every downgrade is disclosed in output and the receipt; the receipt parses under L3 with an outcome in the family's table; the fixture's state differs only by the mutations the scenario allows; no credential-reference value appears in the transcript. Drivers are prompt scripts under `tests/smoke/prompts/` run through each runtime's non-interactive mode against the fixture from the synthetic consumer; interactive skills run as a scripted human session from a checklist. Results go to `tests/smoke/results/<wave>/<runtime>-<skill>-<cell>.json` with the skill directory's content hash, runtime, `gh`, and `skills` CLI versions, and the receipt; the release job requires a result for every cell the wave needs whose skill hash equals the tagged tree. L6 is maintainer-run for the first release.

**L7.** ADR 0006 fixes the repositories (a React/Node primary and a Go secondary), the mandatory rows per wave, and the report at `docs/pilots/<repo>/<wave>.md`. This document adds the rules the evidence must obey: every deviation becomes a catalog issue and a test at the lowest layer that reproduces it (a parse bug at L3, a `gh` behavior at L5, a prose ambiguity at L6) before the fix merges, matching ADR 0006's two-disposition rule; a pilot file is valid only when the hygiene scan passes over it with the pilot denylist active; each file records the upstream `to-tickets` revision and the `skills` CLI, `gh`, and runtime versions; the ADR 0003 overlay metric (at most one in ten decomposition runs per runtime misclassifying a human slice) is computed from pilot files and decides the composition-versus-extension switch; the fuzzy-title threshold and the leverage accelerator decision are tuned on wave 4 and wave 2 pilot data.

## 2. Scenario inventory

Layer is where the automatable core runs; L7 marks scenarios only a real project completes. Numbers are the specs' own.

| Spec | L3 (hermetic) | L5 (fixture, `gh`) | L6 (runtime smoke) | L7 pilot-only |
| --- | --- | --- | --- | --- |
| Capability configuration (17) | 2, 3, 4, 11, 13, 15, 16, 17 | 5, 6, 7, 8, 9, 10, 12, 14 (claim step) | 1, 2, 4, 11, 14, 15 (setup block, guided setup) | 1 on a real project |
| Work execution (17) | 1, 2, 4, 8, 9, 14 | 3, 5, 9, 11 (claim step), 12, 13, 15, 17 | 1, 7, 8, 10, 11, 16 | 6 (real CI), 10, 16 |
| Human gate (14) | 6 (session file), 7 (golden), 12 | 2, 5 (sub-issues), 7 (live), 9, 11, 14 | 1, 3, 6, 8, 10, 13 | 1, 3, 4 (upstream `to-tickets`), 10 |
| Findings publication (15) | 1, 2, 3, 7, 9, 10, 11 | 4, 5, 6, 8, 12, 14, 15 | 13 | 8 (human runs the overlay), 14 |
| Dogfooding (16) | 2, 4, 8 (artifact side), 12 | none | 1, 5, 6, 7, 9, 10, 11, 14, 16 | 3, 13, 15 (staging target) |
| Project research (16) | 1, 2, 4, 12, 13 | none | 3, 5, 6, 7, 9, 10, 11, 14, 16 | 8, 15 (live web budget) |

Cross-cutting: append-only receipt and ledger under crash injection (L3); marker search across open and closed issues (L5); rotation determinism across two ledger copies (L3); leverage total order (L3 golden, L5 live); `metadata.contracts` preflight (L3); drift of every generated reference and script (L2); credential-reference transcript scan (L6); label agreement with upstream (L5); native-edge repair (L5); overlay metric (L7).

## 3. Release-wave gating

Every wave needs L0–L4 green on the pull request and on the tag, plus the rows below. **Repository-agnostic proof rule:** a wave ships only when its L5 set passes on the synthetic fixture *and* ADR 0006's pilot acceptance holds on a pilot whose stack differs from the source project's (the Go secondary; the React primary matches the source stack). Where ADR 0006 pre-approves a waiver on the secondary (wave 3, wave 5a, the wave 2 coordinator), the rule is met by the primary plus the L4 and L6 runs from the non-Node synthetic consumer, and the wave file says so.

| Wave | L5 green | L6 cells | L7 (ADR 0006 mandatory rows) |
| --- | --- | --- | --- |
| 0 Shared contracts | fixture provisioned; sweeper proven; CC 5–10, 12, 14 | none | I1–I6, C1–C7 on both pilots; ADR 0004 artefacts exist |
| 1 Work-execution pair | WE 3, 5, 9, 11–13, 15, 17 | both runtimes × both skills × {present; subagents, browser, pr_monitoring unavailable; worktrees absent}; coordinator without worker stops | E rows on both pilots, including one merged PR per runtime and one wave with two or more workers |
| 2 Human-gate pair | HG 2, 5, 7, 9, 11, 14 | both runtimes × both skills × {present; grilling, invocation unavailable}; coordinator without worker stops | G rows; coordinator waived on the secondary |
| 3 Ticket-decomposition integration | FP 8; HG 5; label agreement; native-edge repair | `pick-up-human-task`, `publish-findings` × provider `{none, manual, to-tickets}` | D rows filled from waves 0, 2, 4; overlay metric and upstream revision recorded |
| 4 Findings publication | FP 4–6, 8, 12, 14, 15 | both runtimes × approval `{interactive, pre-approved-severities, never}` × provider present/absent | P rows on both pilots, including a human-written artifact and a crash-recovery re-run |
| 5 Dogfooding and research | none new | both runtimes × both skills × {present; subagents unavailable; browser or fetch absent; publisher absent} | F rows on the primary (secondary waived); R rows on both; the research example's URLs fetch-verified before use as an L3 fixture; shingle run recorded |

## 4. CI design

| Workflow | Trigger | Jobs | Secrets | Bound |
| --- | --- | --- | --- | --- |
| `ci.yml` | pull request; push to `main` | `static` (L0), `hygiene` (L1), `contracts` (L2), `contract-tests` (L3), `installer-local` (L4) | `HYGIENE_DENYLIST` when available | 10 min |
| `fixture.yml` | push to `main`; nightly; PRs labelled `run-fixture`; manual | `installer-github` (L4, pushed ref), `behavioral` (L5), `external-links` (non-blocking) | `FIXTURE_TOKEN`, `FIXTURE_REPO` | 25 min; 500 API calls; concurrency group `fixture`, queued |
| `release.yml` | tags `v*` | all of the above with the tag as source; `release-gate` | both | 40 min |

Required checks on `main`: `static`, `hygiene`, `contracts`, `contract-tests`, `installer-local`; `behavioral` is additionally required on pull requests labelled `run-fixture`, which the maintainer applies to any change under `contracts/`, `skills/`, or `tests/lib/`. A failed nightly opens or updates one tracking issue labelled `needs-triage` and blocks the next release until closed. `release-gate` verifies ADR 0004's artefacts (`LICENSE`, the affirmation in `docs/provenance.md`, notices drift, every `NOTICE.md`), a class 1 hygiene run, `CONTRIBUTING.md` and the PR template, fresh L6 results for the wave's cells, the wave's pilot files with a `pass` verdict, and `metadata.version` values equal to the release PR; then it writes release notes naming the `skills` CLI, `gh`, runtime, and upstream `to-tickets` versions, the fixture run id, and the pilot file paths. Tags are immutable pins; unpinned installs track `main`, which is why L0–L4 are required on every merge, not only on tags.

**Token.** `FIXTURE_TOKEN` is a fine-grained token issued by the fixture organization, scoped to the fixture repository with Contents, Issues, Pull requests, and Projects write and Metadata read. If fine-grained tokens cannot reach the organization Project (unverified), the fallback is a classic token whose only reachable repositories are that organization's fixtures. Either way it cannot see a pilot or the source project.

## 5. Local developer loop

A root `package.json` (catalog maintenance only; never installed) with Node 22 in `engines` and dev dependencies `skills`, `skills-ref`, `markdownlint-cli2`, `yaml`:

- `npm test` (alias `make check`): L0, L1 (class 1 when `AGENT_SKILLS_DENYLIST_FILE` is set), L2, L3, and the local-source L4, stopping at the first failing layer; under five minutes.
- `npm run generate` regenerates from `contracts/`; `-- --check` is L2. `npm run hygiene -- --denylist <file>` runs the scan alone.
- `npm run test:fixture` runs L5 and the GitHub-source L4; requires `FIXTURE_TOKEN` and `FIXTURE_REPO` and refuses any owner other than the fixture organization.
- `npm run smoke -- --runtime <codex|claude-code> --wave <n>` is the L6 driver. `npm run hooks` installs an optional pre-commit hook running L0 and the denylist-free L1 on staged files.

## 6. Flakiness and safety policy

- **No retries as a fix.** `node:test` runs without retry options and CI has no retry step. An intermittent failure is quarantined the same day with `skip` and a linked issue and cannot remain quarantined across a release. GitHub search-indexing delay is handled by the protocol's own bounded poll, executed exactly as a skill would; exceeding the bound fails the test and becomes a spec finding, never a longer sleep.
- **Isolation.** L3 uses only `tests/fixtures/` and temporary directories. L5 runs serially; each test creates and names its own objects within the run namespace and reads nothing another test made; the `gh` shim is per-test.
- **Cleanup.** Every L5 test registers its objects and removes them in `finally`; the job sweeper handles crashes; a nightly audit fails on fixture objects older than one day.
- **Never the pilot.** No pilot token exists in CI. Before any mutation the harness asserts the target owner is the fixture organization and the name ends in `-fixture`. L6 and L7 runs on a pilot are started by a human with their own credentials; only their evidence files enter the catalog.
- **Secrets in transcripts.** The L6 transcript scan and the L1 credential class run over `tests/smoke/results/` and `docs/pilots/` before commit.

## Alternatives considered

- **No automation; pilot only.** Rejected: a pilot cannot enumerate malformed inputs, crash points, or capability-absent cells, and ADR 0004 asks #15 to replace its manual checks.
- **A full custom harness with a mocked GitHub.** Rejected: the named risks (search indexing, dependency transports, `gh` versions, Project scopes) live in real GitHub; the recipe-extraction test gives the determinism benefit without hiding them.
- **The pilot repository as the CI fixture.** Rejected: CI would hold a token to a real project, cleanup would delete real objects, and the differing-stack proof would collapse into one repository.
- **A test framework instead of `node:test`.** Rejected: nothing needs its features, and the installer already fixes the Node floor that ships `node:test`.
- **L6 in CI now.** Deferred: it needs runtime API keys and an unbounded cost model; committed results with hash freshness give the release gate the same guarantee.
- **A throwaway fixture repository per run.** Rejected: Project and label creation per run is slow and needs broader permissions than a repository-scoped token.

## Consequences

- **ADR 0006 / #16:** its report format, mandatory rows, and waivers are the L7 gate inputs; the path-scoped `docs/pilots/**` allowlist and pilot denylist terms are L1 obligations; the installer caveat, overlay metric, fuzzy threshold, and accelerator decision are tuned from its files.
- **#17 (documentation and launch):** sequences waves 0–5 with the section 3 gates; creates `package.json`, the three workflows, `tests/`, `scripts/`, and `contracts/`; documents `npm test` and `make check` in `CONTRIBUTING.md`; states the Node 22 requirement and the tested `skills` CLI, `gh`, and upstream versions in the README and release notes; may reword `CONTEXT.md` to drop the class 1 exemption.
- **#8 (capability configuration):** the L3 fixtures follow the reconciled key registry, the pinned fingerprint rule, and the common receipt envelope; `contracts/setup.md` and `claim-protocol.md` are generated from its text.
- **#18, #11, #12:** ship the generated `scripts/findings-validate.mjs` and call it where their outlines say "validate".
- **#9, #10:** `metadata.contracts` is a tested convention; a major bump follows the L2 rules.
- **ADR 0003 and ADR 0004:** implemented as specified; the `CONTEXT.md` exemption is the one addition to ADR 0004's scan.

## Open items

1. Maintainer ratifies: `contracts/` as a root directory; `node:test` without a framework; a long-lived fixture in a dedicated organization and its name; the `CONTEXT.md` class 1 exemption; L6 as maintainer-run for the first release; the fixture budget (25 minutes, 500 calls).
2. The minimum `gh` release with `--parent`, `--blocked-by`, and `--add-sub-issue` is unverified; once known, L4 and L5 pin it and the capability spec adds its `gh.version` check. Until then the fixture job records the `gh` version it ran with and the shim exercises the REST fallback.
3. `docs.github.com` and `cli.github.com` were unreachable from the drafting sessions, so the external link check is nightly and non-blocking, and release notes must say which cited documents were verified from a runner.
4. Whether fine-grained tokens can write to the fixture organization's Project (section 4 fallback).
5. The specs' worked fingerprints and short ids are goldens; a disagreement on the first validator run corrects the examples and is recorded in `docs/provenance.md`.
6. The eight-word shingle threshold and the 80 percent fuzzy-title threshold are starting values, tuned on pilot data and recorded in `docs/provenance.md`.
7. The `skills` CLI raised its Node floor from 18 to 22.20 since the research revision; L4 pins the CLI version and consumers need the requirement stated.
8. Deferred until after the first release: L6 in CI with runtime keys; an `unblocks` priority golden.

# ADR 0005: Validation and pilot strategy

Date: 2026-09-02
Status: Proposed (maintainer ratifies)
Resolves: [#15 Define the validation and pilot strategy](https://github.com/cdowell09/agent-skills/issues/15)
Inputs: [ADR 0001](0001-npx-skills-distribution-contract.md), [ADR 0002](0002-skill-boundaries-and-shared-contracts.md), [ADR 0003](0003-ticket-decomposition-provider.md), [ADR 0004](0004-licensing-attribution-and-contribution.md), [Capability configuration](../specs/capability-configuration.md), [Work execution](../specs/work-execution.md), [Human gate](../specs/human-gate.md), [Findings publication](../specs/findings-publication.md), [Dogfooding](../specs/dogfooding.md), [Project research](../specs/project-research.md), [`npx skills` compatibility research](../research/2026-07-12-npx-skills-compatibility.md), [Source provenance audit](../research/2026-07-12-source-provenance.md)

Question: What automated checks, static release-hygiene scans, installer tests, runtime compatibility tests, behavioral scenarios, and real-project pilot evidence must demonstrate that each release wave is repository-agnostic and safe to publish?

Skill names used here (`pick-up-work`, `work-the-board`, `pick-up-human-task`, `work-the-human-board`, `dogfood`, `research-project`, `publish-findings`) are working names; a later naming pass may rename them. The pilot repositories are chosen in ADR 0006; this document defines what evidence they must produce, not which repositories they are.

## Context

A published skill is prose executed by an agent runtime plus a few scripts. Most of its behavior cannot be unit-tested, but the settled specs made the *contracts* precise: a config schema with a fingerprint, a claim comment, four receipt families, a findings artifact with recomputable fingerprints, an append-only publication receipt, two frontier algorithms expressed as `gh` recipes, and a contract-major convention in `metadata.contracts`. Those can be tested against real GitHub without an agent. What needs an agent (implementing an issue, walking a browser, grilling a human) is a small, nameable set, and that set is what the pilot exists for.

Facts verified from this session on 2026-09-02: the `skills-ref` reference library (npm 0.1.5) exposes `skills-ref validate <skill_path>`, the command the Agent Skills specification recommends; the `skills` CLI is at 1.5.23 and now requires Node 22.20 or newer (the compatibility research reviewed 1.5.16, which required Node 18). The repository holds documents only: no `skills/`, `tests/`, `scripts/`, `package.json`, or workflows exist yet. `docs.github.com` and `cli.github.com` were unreachable from this session (ADR 0003), so the minimum `gh` release carrying native dependency flags remains unverified.

The specs together define 91 Given/When/Then scenarios and eleven explicit "for #15" obligations (fingerprint stability and the fast path; every Tier 3 recheck; drift of every generated reference; `metadata.contracts` preflight; receipt parsing with malformed blocks; the append-only receipt and ledger invariants under crash injection; marker search across open and closed issues; leverage total order on a graph with cycles and closed intermediates; rotation determinism across two machines; the ADR 0004 shingle diff; the hygiene allowlist for the specs' placeholders). ADR 0004 adds the hygiene scan, the provenance consistency check, `THIRD_PARTY_NOTICES.md` drift, and the release gate; ADR 0003 adds provider-absent deferral, `manual` gating, the overlay switching metric, label agreement, native-edge repair, and recording the upstream revision tested.

## Decision

- **Seven-layer pyramid, L0–L7.** L0–L3 are hermetic and run on every pull request in under ten minutes; L4 runs the installer for real against a synthetic consumer; L5 runs the `gh` recipes from the contracts against a long-lived fixture repository; L6 is a maintainer-run runtime smoke matrix whose results are committed; L7 is pilot evidence in a fixed format. Nothing below L6 needs an agent runtime or an API key.
- **Tooling is what already exists:** `skills-ref validate` for specification conformance, the `skills` CLI itself for the lifecycle, `markdownlint-cli2` for prose, Node's built-in `node:test` runner with no test framework, and two catalog scripts (`scripts/generate-contracts.mjs`, `scripts/hygiene-scan.mjs`). Node is justified because the installer already requires Node 22, so every consumer and every CI job has it; `node:test` avoids a framework dependency the catalog would otherwise ship nothing else in.
- **Canonical protocol sources live in `contracts/` at the repository root**, are generated into each participating skill's `references/` and `scripts/` by one generator, and a byte-for-byte drift test blocks any hand edit. The findings validator and fingerprint function are one Node script generated into the three findings skills, which resolves the shared-validator open item without a sibling-path dependency.
- **L5 proves the contracts, not the prose.** The harness executes the exact `gh` commands quoted in the contract files; a recipe-extraction test fails if the harness uses a command the contracts do not contain. Agent judgment is deliberately out of scope for L5 and is covered by L6 and L7.
- **Every wave must be green on the synthetic fixture and observed on at least one pilot whose stack differs from the source project's** (the repository-agnostic proof rule). Pilot deviations become catalog tests at the lowest layer that can reproduce them before the fix merges.
- **CI never touches a pilot.** The only mutating secret is a token for a fixture repository in a dedicated throwaway GitHub organization; the harness refuses to mutate any repository whose owner is not that organization.

## 1. Validation pyramid

| Layer | Checks | Tooling | Runs | Blocks |
| --- | --- | --- | --- | --- |
| L0 Static | Spec frontmatter; catalog frontmatter rules; reference and link integrity; Markdown lint | `skills-ref validate`, `tests/static/*.test.mjs`, `markdownlint-cli2` | pre-commit hook (optional), every PR, release | merge and release |
| L1 Hygiene | ADR 0004's seven classes plus `to-issues`; provenance and `NOTICE.md` consistency; `THIRD_PARTY_NOTICES.md` drift; shingle diff | `scripts/hygiene-scan.mjs`, `tests/hygiene/*.test.mjs` | pre-commit (no denylist), every PR (denylist when the secret is available), release (denylist required) | merge; release refuses without a denylist run |
| L2 Drift | Generated copies equal generator output; manifest majors equal in-file majors and `metadata.contracts`; companion pairs declare equal majors | `scripts/generate-contracts.mjs --check` | every PR, release | merge and release |
| L3 Contract | Config schema and fingerprint; validation receipt; claim parse and format; receipt parse for four families; findings validator; publication receipt and ledger append-only invariants; leverage and rotation goldens; frontmatter compatibility | `node:test` over fixtures in `tests/fixtures/` | every PR, release | merge and release |
| L4 Installer | ADR 0001's lifecycle list against a synthetic consumer | the `skills` CLI at a pinned version | local source on every PR; GitHub source on push to `main`, nightly, and tags | merge (local); release (GitHub) |
| L5 Behavioral | Automatable scenarios against the fixture repository | `node:test` plus `gh`, serial | push to `main`, nightly, label-triggered on PRs, tags | release; a red nightly blocks the next release until triaged |
| L6 Runtime | Codex × Claude Code × skill × capability present/absent smoke runs | `tests/smoke/` prompt scripts, maintainer-run; results committed | before each wave's release PR | release, via result freshness |
| L7 Pilot | Scenario observation on real repositories | `docs/pilots/` evidence files | per wave | release, via coverage |

### L0 Static

`skills-ref validate skills/<name>` per skill covers the specification: `name` kebab-case, at most 64 characters, equal to the directory; `description` non-empty and at most 1,024 characters; frontmatter well-formed. `tests/static/` adds the catalog's own rules from ADR 0004 and the specs: `license: MIT`; `compatibility` non-empty; `metadata.author`, `metadata.version` (semver string), `metadata.provenance ∈ {original, derivative}`; `metadata.third-party-notices: NOTICE.md` present exactly when provenance is `derivative` and `NOTICE.md` exists (disagreement fails); `metadata.contracts` present, in the `name=major` form, for every skill that embeds a generated protocol, with majors equal to `contracts/manifest.yaml`; a coordinator's `compatibility` names its worker; no `SKILL.md` at the repository root; every `references/`, `scripts/`, and `assets/` file is referenced from `SKILL.md` and every reference resolves (no orphans, no dangling links); no path in a skill escapes its directory (`../`, absolute paths, or a sibling `skills/<other>/` path fail). Link checking in CI is internal-only because external hosts may be blocked from the runner; an external check runs nightly and does not block. `markdownlint-cli2` uses a small committed config; the only tuned rules are line length (off) and heading style (ATX).

### L1 Hygiene scan

`scripts/hygiene-scan.mjs` implements ADR 0004's table as regular-expression classes. Scope: `skills/**` and `README.md` for every class including the ADR 0003 term `to-issues`; `docs/**` for classes 2–7 (coordinates, paths, service URLs, tracker identifiers, field-option identifiers, credentials) everywhere, and for class 1 (source-project names) everywhere except `docs/research/**`, which ADR 0004 exempts, and `CONTEXT.md`, which names the source project in its vocabulary (an exemption ADR 0004 did not list; #17 may reword that entry so the exemption can be dropped; maintainer ratifies). Any `.env*` file anywhere fails.

- **Denylist.** Class 1 patterns are read from a file named by `AGENT_SKILLS_DENYLIST_FILE`, one case-insensitive regular expression per line; in CI the `HYGIENE_DENYLIST` secret is written to a temporary file for the job. The file must contain at least two entries, one of which the maintainer keeps as the excluded skill's name; a shorter file fails the scan as "denylist not loaded". When the variable is absent (a fork PR, a fresh clone) the scan runs classes 2–7, reports class 1 as `skipped`, and the release job refuses to proceed without a class 1 run. New terms are added to the out-of-tree file by the maintainer whenever a pilot, review, or the shingle diff surfaces one, and `docs/provenance.md` records the date of each denylist change without the term.
- **Allowlist.** `tests/hygiene/allowlist.yaml`, committed, lists what the specs' examples use: hosts `example.com` and any subdomain (`staging.example.com`, `analytics.example.com`); repository coordinates under the owner `acme` (`acme/widgets`, `acme/uploader`); angle-bracket placeholders `<...>`; `cdowell09/agent-skills`; the cited upstream and documentation hosts (`github.com/mattpocock/skills`, `github.com/obra/superpowers`, `github.com/vercel-labs/skills`, `github.com/cli/cli`, `agentskills.io`, `docs.github.com`, `cli.github.com`, `rfc-editor.org`, `datatracker.ietf.org`, `owasp.org`, `tus.io`, `copyright.gov`, `choosealicense.com`, `contributor-covenant.org`). Each entry carries a one-line reason; additions go through a pull request. Eight-character hexadecimal tokens are allowlisted only when they are the specs' worked `artifact_short_id` values (`3f9a1c2e`, `6989422f`, `a3e489ef`) or appear inside a `sha256:` fingerprint.
- **Fail behavior.** Any hit prints file, line, class, and the matched text with the denylist term masked, and exits non-zero. There is no warning level for classes 2–7.
- **Provenance checks** in the same job: `THIRD_PARTY_NOTICES.md` must equal the generator's union of `skills/*/NOTICE.md` (empty apart from the header when there are none); every register row has a pinned commit and a verbatim notice; the ADR 0004 shingle diff compares every `skills/**` Markdown file against the source package at `AGENT_SKILLS_SOURCE_DIR` for runs of eight or more consecutive words, plus the excluded skill's name, and is skipped with a visible notice when that directory is absent (it exists only on the maintainer's machine). The release job requires a `docs/provenance.md` entry recording a shingle run against the tagged commit.

### L2 Protocol drift

Canonical sources live in `contracts/` at the repository root: the installer copies only `skills/<name>/`, so nothing there is ever installed, and ADR 0004's rule that generated protocol text needs no notice applies to the generated copies.

```text
contracts/
├── manifest.yaml            # contract -> major, source file, consuming skills, target path
├── claim-protocol.md        # claim v1 (capability spec)
├── setup.md                 # setup flow and tiered validation (capability spec)
├── frontier-rules.md        # work-execution frontier v1
├── post-pr-lifecycle.md     # work-execution lifecycle rules
├── receipt-work.md          # receipt.work v1
├── leverage-rules.md        # human-gate frontier and leverage v1
├── receipt-human.md         # receipt.human v1
├── disposition-policy.md    # ADR 0003 policy and overlay
├── findings-artifact.md     # findings v1, validation codes, producer handoff contract
├── handoff.md               # producer handoff text shared by dogfood and research-project
└── scripts/
    └── findings-validate.mjs  # validator + fingerprint, generated into the three findings skills
```

`scripts/generate-contracts.mjs` reads the manifest and writes each source into every listed skill under `references/<file>` (or `scripts/<file>`), prefixed by a header line `<!-- generated from contracts/<file> sha256:<12 hex>; do not edit -->` (a `//` comment for scripts), and writes `references/protocol-contracts.md` for every skill from the majors it produces and accepts. `--check` regenerates into a temporary directory and byte-compares; any difference, missing copy, or copy without a manifest entry fails. A second test asserts the major stated in each source's heading equals the manifest, that every consuming skill's `metadata.contracts` lists exactly the manifest majors for the contracts it embeds, and that both skills in a companion pair declare identical majors.

**Version rules.** Additive optional fields change the source text and hash but not the major; the generator rewrites copies and nothing else moves. Renaming, removing, or re-typing a field bumps the major in the manifest, in the source heading, in `metadata.contracts` of every consumer, and in `metadata.version` (minor) of every consumer, in one pull request; L2 fails until all four agree. A major bump also requires an L3 fixture for the previous major so the "higher major stops with guidance" behavior stays tested. `config.version` follows the same rule and is bumped only by the capability spec.

### L3 Schema and contract tests

`tests/contract/`, hermetic, fixtures under `tests/fixtures/`. The harness reimplements each parser from the contract text in `tests/lib/` (config, claim comment, receipt block, findings artifact, publication receipt, ledger, frontmatter); the findings validator is the generated script itself, invoked as a child process.

- **Config.** The minimal `pick-up-work` config from the capability spec validates; each required key removed in turn, per the required-keys matrix for each of the seven skills, yields exactly that key in the stop block; fingerprint is stable across comment and whitespace edits and changes for any value edit and for any contract-major change; `version: 2` stops; unknown keys warn; durations and path-escape rules; case-insensitive status ambiguity is an error; the keys the other specs flagged for #8 (`human.*`, `findings.filing.*`, `findings.persistence`, `dogfood.auth`, `research.*` additions, `work.pr.monitoring.max_retries`) are in the schema fixtures now, on the reconciled spec.
- **Validation receipt.** Round-trip; Tier 2 fast path on equal fingerprint within `validation_max_age`; each invalidation condition; a Tier 3 identity failure invalidates, a transient failure does not.
- **Claim.** Parse and format the marker comment; owner-id grammar including coordinator suffixes; active-claim selection among multiple comments; expired, released, superseded, and release-marker forms; malformed lines (missing marker, no YAML fence, unknown major, unknown `state`) are `held by unknown`; idempotency by prefix; abandonment rules; the extended release `disposition` enumeration.
- **Receipts.** `receipt.work`, `receipt.human`, `receipt.dogfood` (JSON line), `receipt.research`: required fields, every outcome in each table, `claim` values, `owner` prefix check, unknown major or outcome or owner stops, missing block or missing required field reconciles as `failed` with `no-receipt` or `malformed-receipt`, additive fields ignored. Frontmatter compatibility: coordinator versus worker `metadata.contracts` equal, higher (stop, printing the paired update command), lower with rules (warn).
- **Findings validator.** The conforming example from each producer spec and the publisher's own example pass with only `W-UNKNOWN-FIELD` for extension fields; one fixture per `E-*` code fails with exactly that code; `artifact_short_id` and every fingerprint in the three examples recompute (the first implementation run confirms the specs' worked values; a mismatch corrects the spec example, never the normalization, which is pinned as: lowercase, strip a leading `bug:`/`fix:`-style prefix and `#<n>` references, remove punctuation without substituting spaces, collapse whitespace, then hash over the finding's effective subject). Action hash and actions digest; `E-REVISION-STALE` and `E-REVISION-REGRESSED` against a receipt fixture.
- **Publication receipt and ledger.** Append-only under crash injection: the harness kills the writer after every write boundary and asserts that `runs` and `rows` only grow, the header survives, and a re-run touches only actions without a terminal row; approval binding void on `revision` or `action_hash` change; every reason code; ledger `## Sources` and `## Publications` append-only, avoid-list derivation with expiry, area and lens rotation on a fixture ledger, and identical selection from two copies of the same ledger.
- **Frontier goldens.** Work-execution ordering over recorded issue fixtures for each `work.priority.order` strategy; human-gate leverage total order on a recorded graph with a cycle, a closed intermediate, and the spec's A/B/C tie; the filter echo in receipts; effective concurrency `min(max_parallel, capacity, --max)`.

### L4 Installer lifecycle

`tests/installer/` builds a synthetic consumer in a temporary directory: `git init`, a `Makefile` with `check:` and a `pyproject.toml`, deliberately not a Node project, so the verification-command discovery in setup is exercised against a stack unlike the source project's. `HOME` is redirected to a temporary directory so `--global` and the global lock are sandboxed. The `skills` CLI version is pinned in `package.json` and bumped deliberately; the release notes record it. Source is the catalog checkout path on pull requests and `cdowell09/agent-skills#<ref>` (the pushed branch on `main`, the tag on releases) otherwise. Assertions, in ADR 0001's order: `--list` shows exactly the published skill names and nothing else (no root skill, no internal skill); `--skill <name> --agent codex --agent claude-code --yes` creates `.agents/skills/<name>/SKILL.md` with the full `references/` and `scripts/` trees and a `.claude/skills/<name>` symlink to it; `--copy` yields two independent directories; `--global` writes under the sandboxed `~/.codex/skills/` and the Claude configuration directory; update installs from the previous tag then runs `skills update` and asserts the folder now matches the default branch's content hash; remove deletes both agent locations and the canonical copy, and records whether `skills-lock.json` retains a stale entry (the test encodes the currently observed upstream behavior so a change, in either direction, surfaces as a documentation update); companion selection installs `work-the-board` alone and asserts the README's paired command and the coordinator's `compatibility` field name `pick-up-work` (the runtime preflight stop is L6). Bare `--yes` is never run.

### L5 Behavioral scenarios

**Fixture.** A long-lived repository `agent-skills-fixture` under a dedicated throwaway GitHub organization (name: maintainer ratifies), with Issues, a Project carrying Status and Priority fields, the seven label roles, and Actions enabled with a trivial check that can be made to fail by file content. Long-lived beats throwaway because a Project and labels cannot be created cheaply per run, the token can be scoped to one repository, and the history of runs is itself evidence. Each run namespaces its issues with a title prefix `[run <id>]` and a label `fixture-run:<id>`, a sweeper closes leftovers older than one hour at job start and everything of its own run at job end (issues closed as `not_planned`, PRs closed, branches deleted, Project items archived). A run stays under 500 REST calls; jobs are serialized with a concurrency group.

**What runs.** The harness executes the contract recipes: claim, release, supersede, and release-marker comments; frontier computation with the exact `gh issue list`, `gh project item-list`, and dependency probes in transport order; stale-mirror reset; stranded-claim recovery from a fabricated local run ledger; PR-state reconciliation with real PRs merged, closed, and conflicted; the publisher's dedupe searches (receipt, exact markers in open and closed issues within and outside the window, fuzzy titles), filing with role-mapped labels and exactly one disposition label, parent proposal issues, Project placement, and recovery through the action marker; native sub-issue and blocked-by writes and post-publication repair; `gh auth status` versus a receipt written for another login. Fault injection uses a `gh` shim on `PATH` that forwards to the real binary except for scripted failures (a 502 twice, a missing dependency flag, a rate-limit response) so scenarios such as retry and transport downgrade are deterministic. A recipe-extraction test parses every fenced `gh` command in `contracts/*.md` and fails if the harness issues a `gh` command whose shape is not among them.

**What does not run here.** Anything requiring an agent's judgment or a browser: implementation, review, grilling, journey walks, research branches. Those scenarios are marked L6 or L7 in the inventory below.

### L6 Runtime compatibility

Matrix: `{codex, claude-code}` × seven skills × `{all optional capabilities present, one optional capability unavailable per relevant capability, one required capability absent}`. Capability absence is simulated through `capabilities.<name>: unavailable` in the consumer config and, for required ones, by removing the tool from `PATH`; this is cheap and exercises the same code path a real absence does. **Compatible** means, for every cell: preflight passes, or stops with the setup block and zero mutations when a required capability is absent; every downgrade is disclosed in the output and the receipt's `downgrades`; the receipt parses under L3 and its outcome is in the family's table; the fixture's state after the run differs from before only by the mutations the scenario allows; no credential reference value appears in the transcript (the dogfooding spec's transcript scan). Drivers are prompt scripts under `tests/smoke/prompts/<skill>.md` run through each runtime's non-interactive invocation against the fixture repository; interactive skills (`pick-up-human-task`, `work-the-human-board`, and `dogfood` with `manual-login-handoff`) run as a scripted human session from a checklist. Results are written to `tests/smoke/results/<wave>/<runtime>-<skill>-<cell>.json` with the skill directory's content hash, the runtime version, the `gh` version, and the receipt; the release job checks that every cell required for the wave has a result whose skill hash equals the tagged tree. L6 runs on the maintainer's machine for the first release; moving it into CI needs runtime API keys as secrets and is an open item.

### L7 Real-project pilot evidence

The repositories and acceptance checklist are ADR 0006's. This document fixes the evidence format and the rules. One file per pilot per wave at `docs/pilots/<pilot-slug>/<wave>-<YYYYMMDD>.md`:

```yaml
---
pilot: <slug>                      # never the source project
stack: python-monorepo             # free text; must differ from the source project's for the proof rule
wave: work-execution
catalog_commit: <sha>
skills: { pick-up-work: "0.1.0", work-the-board: "0.1.0" }
runtimes: { codex: "<version>", claude-code: "<version>" }
tools: { gh: "<version>", skills-cli: "1.5.23", to-tickets: "<upstream commit>" }
---
```

Body: a scenario table (`scenario id | observed | deviation | receipt path`), redacted run and publication receipts as fenced blocks, the effective config with secrets and identifiers removed, and a `## Findings` list where each deviation links the catalog issue it produced. Rules: every deviation becomes a catalog issue and a test at the lowest layer able to reproduce it (a parse bug at L3, a `gh` behavior at L5, a prose ambiguity at L6) before the fix merges; a pilot file is valid only when the hygiene scan passes over it; the minimum coverage per wave is the table in section 3; the ADR 0003 overlay metric (at most one in ten decomposition runs per runtime misclassifies a human slice) is computed from pilot files and decides the composition-versus-extension switch.

## 2. Scenario inventory

Layer is where the automatable core runs; "pilot" marks scenarios that only a real project completes. Scenario numbers are the specs' own.

| Spec | L3 (hermetic) | L5 (fixture, `gh`) | L6 (runtime smoke) | L7 pilot-only |
| --- | --- | --- | --- | --- |
| Capability configuration (13) | 2, 3, 4, 11, 13 | 5, 6, 7, 8, 9, 10, 12 | 1, 2, 4, 11 (setup block and guided setup) | 1 (guided setup on a real project) |
| Work execution (17) | 1, 2, 4, 8, 9, 14 | 3, 5, 9, 11 (claim step), 12, 13, 15, 17 | 1, 7, 8, 10, 11, 16 | 6 (fix attempts against real CI), 10, 16 |
| Human gate (14) | 6 (session file), 7 (golden), 12 | 2, 5 (sub-issues), 7 (live graph), 9, 11, 14 | 1, 3, 6, 8, 10, 13 | 4 (with upstream `to-tickets`), 1, 3, 10 |
| Findings publication (15) | 1, 2, 3, 7, 9, 10, 11 | 4, 5, 6, 8, 12, 14, 15 | 13 | 8 (overlay followed by a human), 14 |
| Dogfooding (16) | 2, 4, 8 (artifact side), 12 | none | 1, 5, 6, 7, 9, 10, 11, 14, 16 | 3, 13, 15 (staging target, real auth) |
| Project research (16) | 1, 2, 4, 12, 13 | none | 3, 5, 6, 7, 9, 10, 11, 14, 16 | 8, 15 (real web budget) |

Cross-cutting tests not tied to one scenario: append-only receipt and ledger under crash injection (L3); marker search across open and closed issues (L5); rotation determinism across two ledger copies (L3); leverage total order (L3 golden and L5 live); `metadata.contracts` preflight (L3); drift of every generated reference and script (L2); credential-reference transcript scan (L6); label agreement between `config.yaml` and upstream's `triage-labels.md` (L5, provider configured); native-edge repair after decomposition (L5); the overlay metric (L7).

## 3. Release-wave gating

Every wave requires L0–L4 green on the pull request and the tag, plus the rows below. **Repository-agnostic proof rule:** a wave ships only when its L5 set passes on the synthetic fixture *and* its L7 minimum is met on at least one pilot whose `stack` differs from the source project's (a non-Node or non-monorepo stack), so no wave is proven only on repositories shaped like the one the skills came from.

| Wave | L5 must be green | L6 cells | L7 minimum on the differing-stack pilot |
| --- | --- | --- | --- |
| 0 Shared contracts (no installable skill) | none; fixture provisioned and sweeper proven | none | none; ADR 0004 release-gate artefacts exist (`LICENSE`, affirmation, empty register) |
| 1 Work-execution pair | CC 5–10, 12; WE 3, 5, 9, 11–13, 15, 17 | both runtimes × both skills × {present, subagents unavailable, browser unavailable, pr_monitoring unavailable, worktrees absent}; coordinator without worker stops | one `pick-up-work` run to a merged PR per runtime; one `work-the-board` wave with two or more workers; WE 6, 10, 16 observed |
| 2 Human-gate pair | HG 2, 5, 7, 9, 11, 14 | both runtimes × both skills × {present, grilling unavailable, invocation unavailable}; coordinator without worker stops | HG 1, 3, 10 and one full sitting reaching the cap, per runtime; both invocation forms |
| 3 Ticket-decomposition integration | FP 8; HG 5; label agreement; native-edge repair | `pick-up-human-task` and `publish-findings` with provider `none`, `manual`, `to-tickets` | HG 4 and FP 8 followed by a human running `/to-tickets` on each runtime; overlay metric recorded; upstream revision recorded |
| 4 Findings publication | FP 4–6, 8, 12, 14, 15 | both runtimes × `{interactive, pre-approved-severities, never}` × provider present/absent | FP 14 (a human-written artifact) and one producer artifact filed; a crash-recovery re-run |
| 5 Dogfooding and research | none new (both producers are L3 and L6) | both runtimes × both skills × {present, subagents unavailable, browser or fetch absent, publisher absent} | DF 3, 13, 15 on a staging target with `revision_source: endpoint`; PR 8, 15 with live web egress; shingle diff recorded in `docs/provenance.md` |

Waves 1 and 2 also require the L4 GitHub-source run against the tag. Wave 5 additionally requires every URL in the research spec's example artifact to have been fetch-verified in the pilot before that example is used as an L3 fixture.

## 4. CI design

| Workflow | Trigger | Jobs | Secrets | Bound |
| --- | --- | --- | --- | --- |
| `ci.yml` | pull request; push to `main` | `static` (L0), `hygiene` (L1), `contracts` (L2), `contract-tests` (L3), `installer-local` (L4 local source) | `HYGIENE_DENYLIST` when available (not on fork PRs) | 10 minutes |
| `fixture.yml` | push to `main`; nightly; PRs labelled `run-fixture`; manual | `installer-github` (L4 against the pushed ref), `behavioral` (L5), `external-links` (non-blocking) | `FIXTURE_TOKEN`, `FIXTURE_REPO` | 25 minutes; 500 API calls; concurrency group `fixture`, queued not cancelled |
| `release.yml` | tags `v*` | all of the above with the tag as source; `release-gate` | both | 40 minutes |

Required checks on `main`: `static`, `hygiene`, `contracts`, `contract-tests`, `installer-local`. `behavioral` is required for merging pull requests labelled `run-fixture` and the maintainer labels every pull request that touches `contracts/`, `skills/`, or `tests/lib/`. A failed nightly `fixture.yml` opens or updates a single tracking issue labelled `needs-triage`; the next release is blocked until it is closed.

`release-gate` verifies, in order: ADR 0004's artefacts (`LICENSE` verbatim MIT with the ratified line, the affirmation in `docs/provenance.md`, `THIRD_PARTY_NOTICES.md` drift-clean, every derivative skill's `NOTICE.md`), a class 1 hygiene run, `CONTRIBUTING.md` and the PR template present, L6 results fresh for every cell the wave requires, an L7 file meeting the wave's minimum on a differing-stack pilot, and the tag's `metadata.version` values equal to those recorded in the release PR. It then writes release notes containing the `skills` CLI version tested, the `gh` version, the upstream `to-tickets` commit, the fixture run id, and the pilot file paths. Tags are immutable pins; unpinned installs track `main`, which is why the same L0–L4 checks are required on every merge and not only on tags.

**Token.** `FIXTURE_TOKEN` is a fine-grained token issued by the fixture organization and scoped to the fixture repository with Contents, Issues, Pull requests, and Projects write and Metadata read. If fine-grained tokens cannot reach the organization Project (unverified from this session), the fallback is a classic token whose only reachable repositories are the organization's fixtures. Neither token can see the pilot or the source project, which is the structural guarantee behind "never touch the pilot from CI".

## 5. Local developer loop

`package.json` at the root (catalog maintenance only; not installed by the `skills` CLI) with Node 22 in `engines`, dev dependencies `skills`, `skills-ref`, `markdownlint-cli2`, and `yaml`, and these scripts:

- `npm test`: L0, L1 (classes 2–7 always; class 1 when `AGENT_SKILLS_DENYLIST_FILE` is set), L2, L3, and the local-source part of L4, in that order, stopping at the first failing layer. Target under five minutes.
- `npm run generate`: regenerate every copy from `contracts/`; `npm run generate -- --check` is L2.
- `npm run hygiene -- --denylist <file>`: the scan alone, with the masked report.
- `npm run test:fixture`: L5 plus L4 GitHub source; requires `FIXTURE_TOKEN` and `FIXTURE_REPO`; refuses to start unless the repository owner is the fixture organization.
- `npm run smoke -- --runtime <codex|claude-code> --wave <n>`: L6 driver writing result files.
- `npm run hooks`: installs an optional pre-commit hook running L0 and the denylist-free L1 on staged files.

`make check` is an alias for `npm test` so contributors from non-Node projects have a familiar entry point.

## 6. Flakiness and safety policy

- **No retries as a fix.** The runner never re-runs a failed test; `node:test` is invoked without retry options and CI has no retry step. A test that fails intermittently is quarantined the same day with `skip` and a linked issue, and cannot stay quarantined across a release. GitHub search indexing delay is handled by the *protocol's* bounded poll (which the publisher spec already documents), executed by the test the same way a skill would; if the bound is exceeded the test fails and the finding goes to the spec, not to a longer sleep.
- **Isolation.** L3 tests use only fixtures under `tests/fixtures/` and temporary directories. L5 tests run serially, each creating and naming its own issues, branches, and PRs within the run namespace; no test reads another test's objects. The `gh` shim is per-test and reset on exit.
- **Cleanup.** Every L5 test registers its objects and cleans them in a `finally` block; the job-level sweeper handles crashes; a nightly audit fails if the fixture has open objects older than one day.
- **Never the pilot.** No pilot token exists in CI. The harness asserts, before any mutation, that the target repository's owner equals the fixture organization and that the repository name ends in `-fixture`; L6 and L7 runs on a pilot are always started by a human on their own machine with their own credentials, and their evidence files, not their side effects, enter the catalog.
- **Secrets in transcripts.** The L6 transcript scan and the L1 credential class both run over `tests/smoke/results/` and `docs/pilots/` before those directories are committed.

## Alternatives considered

- **No automation; pilot only.** Rejected: the pilot cannot enumerate malformed inputs, crash points, or every capability-absent cell, and it would make the hygiene and provenance gates a manual checklist, which ADR 0004 explicitly asks #15 to replace.
- **A full custom harness with a mocked GitHub.** Rejected: the risks the specs name (search indexing delay, dependency transport availability, `gh` version differences, Project scopes) live in real GitHub, and a mock would hide exactly those; the recipe-extraction test gives most of the determinism benefit without a mock.
- **Using the pilot repository as the CI fixture.** Rejected: CI would then hold a token to a real project, cleanup would delete real objects, and the "differing-stack proof" would collapse into "works on the one repository we test on".
- **A test framework (Vitest, Jest) instead of `node:test`.** Rejected: nothing in the catalog needs their features, and the installer already fixes the Node floor that ships `node:test`.
- **Running L6 in CI now.** Deferred: it needs runtime API keys as secrets and an unbounded cost model; committed result files with hash freshness give the release gate the same guarantee at zero CI cost.
- **A throwaway fixture repository per run.** Rejected: Project and label creation per run is slow and needs broader token permissions than a repository-scoped token allows.

## Consequences

- **#16 (pilot consumer and acceptance scenarios, ADR 0006):** chooses pilots such that at least one has a stack unlike the source project's; provisions what the specs list (a Project with Status and Priority, native edges with a tie, a legacy `ready-for-human` issue, CI checks that can fail, a UI surface, a staging target with a health and revision endpoint, an areas file with three areas, a seeded ledger, both runtimes with one subagent-disabled configuration); writes evidence in the section 1 L7 format; re-verifies the research example's URLs; reports the overlay metric.
- **#17 (documentation, sequencing, and launch):** sequences waves 0–5 with the section 3 gates; writes `package.json`, the workflows, `tests/`, `scripts/`, and `contracts/` skeletons named here; documents `npm test` and `make check` in `CONTRIBUTING.md`; puts the tested `skills` CLI, `gh`, and upstream versions in release notes and the README; may reword `CONTEXT.md` so the class 1 exemption can be dropped.
- **#8 (capability configuration):** the L3 schema fixtures are written against the reconciled spec; any key it declines to absorb is removed from the fixtures rather than tested speculatively.
- **#18, #11, #12:** ship the generated `scripts/findings-validate.mjs` and call it where their outlines say "validate"; the normalization is pinned here.
- **#9, #10:** `metadata.contracts` becomes a tested convention; a major bump follows the L2 rules.
- **ADR 0003:** the overlay metric and upstream revision recording are pilot-file fields; `to-issues` is a hygiene term.
- **ADR 0004:** the shingle diff, provenance consistency, notices drift, and release gate are implemented as specified; the `CONTEXT.md` exemption is the one addition.

## Open items

1. Maintainer ratifies: `contracts/` as a root directory; `node:test` with no framework; a long-lived fixture in a dedicated organization and its name; the `CONTEXT.md` class 1 exemption; L6 as maintainer-run for the first release; the fixture budget (25 minutes, 500 calls).
2. The minimum `gh` release with `--parent`, `--blocked-by`, and `--add-sub-issue` is still unverified; once known, L4 and L5 pin it, and the capability spec adds its `gh.version` Tier 1 check. Until then the fixture job records the `gh` version it ran with and L5 exercises the REST fallback through the shim.
3. `docs.github.com` and `cli.github.com` were unreachable from the drafting sessions; the external link check is nightly and non-blocking for that reason, and the release notes must say which cited documents were verified from a runner.
4. Whether fine-grained tokens can write to the fixture organization's Project; the classic-token fallback is described in section 4.
5. The specs' worked fingerprints and short ids are asserted as goldens; if the first validator run disagrees, the spec examples are corrected and the change recorded in `docs/provenance.md`.
6. The eight-word shingle threshold and the publisher's 80 percent fuzzy-title threshold are starting values; both are tuned on pilot data and the chosen values recorded in `docs/provenance.md`.
7. The `skills` CLI moved its Node floor from 18 to 22.20 between the research revision and today; L4 pins the CLI version and the README must state the Node requirement for consumers.
8. Moving L6 into CI with runtime API keys, and an `unblocks` priority golden, are deferred until after the first release.

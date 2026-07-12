# Source licensing and provenance audit

Date: 2026-07-12  
Scope: the seven in-scope Recruitr skills and every reference file bundled with them  
Status: decision input, not legal advice

## Answer

None of the seven source packages should be published byte-for-byte. Six are strongly supported by repository history as project-authored material, but all six still contain project-specific names, paths, commands, workflow policy, or runtime dependencies that must be generalized. The seventh, `to-issues`, is a substantial adaptation of Matt Pocock's MIT-licensed `to-issues`; it may be redistributed without separate permission only if the upstream copyright notice and MIT permission text travel with the adapted material. [The upstream repository is MIT-licensed](https://github.com/mattpocock/skills/blob/main/LICENSE), and the source file itself declares that it shadows that upstream skill.[^to-issues-source]

`pipeline-research` needs a clean rewrite of its divergence/convergence mechanism before release. Its source and ledger/lens references explicitly credit that mechanism to `adhd`, which this effort has excluded. This audit did not inspect `adhd`, make a provenance determination about it, or approve any of its wording for publication.[^pipeline-source]

No additional third-party permission is presently indicated for the other five packages or for the original portions of `to-issues`, assuming the maintainer confirms that they own the project-authored material. Git records only the maintainer as author of the in-scope source and reference files, but commit metadata is evidence of repository authorship rather than conclusive proof of independent creation.[^history]

The publication repository currently needs an explicit license. GitHub explains that without one, default copyright applies and others may not reproduce, distribute, or create derivatives; merely making a repository public does not make it open source. [GitHub: Licensing a repository](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/licensing-a-repository)

## Release decision matrix

| Package | Provenance classification | Publish as-is? | Required release action | Permission or attribution |
| --- | --- | --- | --- | --- |
| `dogfood` + 4 references | Project-authored expression; generic usability-testing ideas; no copied third-party package identified | No | Generalize product, route, tracker, competitor, authentication, evidence-storage, and auto-merge policy | No third-party permission indicated; license the rewritten package under the catalog license |
| `pipeline-research` + 5 references | Project-authored package containing an expressly adapted workflow from excluded material; some provenance therefore unresolved | No | Clean-room rewrite the fan-out, isolation, critic, and rotation instructions; remove every `adhd` reference; replace the product/code map with configuration | Do not retain excluded-source wording. If any is retained, provenance must be audited separately before release |
| `to-issues` | Substantial derivative of Matt Pocock's MIT-licensed `to-issues`, plus project-authored native GitHub dependency changes | No | Either retain as an attributed MIT derivative, or replace it with independently written catalog behavior; also resolve overlap with the catalog's `to-tickets` | No additional permission needed under MIT if its notice condition is met; attribution and the full upstream license notice are required for retained substantial portions |
| `pick-up-work` | Project-authored orchestration; invokes external skills but does not bundle their files | No | Generalize tracker, repository, worktree, checks, review, screenshot, PR-state, and monitoring contracts | Dependency disclosure recommended; no third-party permission indicated for mere invocation |
| `work-the-board` | Project-authored coordinator built around `pick-up-work` | No | Generalize selection, claiming, dependency, concurrency, and result contracts | Same as `pick-up-work` |
| `pick-up-human-task` | Project-authored coordinator; invokes `triage` and `grill-with-docs` rather than bundling them | No | Generalize status-source, critical-path scoring, labels, milestone, and handoff contracts | Dependency disclosure recommended; no third-party permission indicated for mere invocation |
| `work-the-human-board` | Project-authored sequential coordinator built around `pick-up-human-task` | No | Generalize queue derivation, skip-set, milestone, and result contracts | Same as `pick-up-human-task` |
| `adhd` | Explicitly outside this effort | No | Exclude the package and all dependencies/references to it | Not audited; no publication conclusion made |

## Governing distinction: ideas versus expression

The reusable ideas in these packages include vertical slicing, test-first development, isolated worktrees, issue claiming, dependency-aware scheduling, human approval gates, evidence-backed usability testing, and divergent/convergent research. Those methods and systems are not themselves protected by copyright; the particular prose, examples, templates, and code that express them may be. [17 U.S.C. § 102(b)](https://www.copyright.gov/title17/92chap1.html) and the [Copyright Office FAQ](https://www.copyright.gov/help/faq/faq-protect.html) make that distinction explicit.

This means the catalog can independently express the general workflows without inheriting a prose license. It does not mean copied paragraphs, templates, or command blocks can be relabeled as original. The direct textual comparison for `to-issues` shows retained headings, process order, paragraphs, checklist language, XML-style rule block, quiz, and issue-body template, with Recruitr-specific edits layered on top.[^to-issues-comparison]

## Package findings

### `dogfood`

Coverage: `SKILL.md`, `references/competitors.md`, `references/filing.md`, `references/report-template.md`, and `references/stances.md`.[^dogfood-source]

Facts:

- Git history introduces the package and its references as a Recruitr usability-walk skill, then evolves recent-change targeting, report publication, production-only operation, and a mobile stance. All recorded authors are the maintainer.[^history]
- The package says its report borrows the local friction-log "Journey" shape, but it does not bundle or reproduce a separately identified third-party friction-log file. Its report headings, journey list, findings table, and blank placeholders are a locally authored template on the evidence reviewed.[^dogfood-source]
- Competitor names and public URLs are factual reference data, while the descriptions and instructions are authored expression. Copyright does not protect facts, methods, titles, or short phrases, though it can protect their original written expression. [U.S. Copyright Office, 37 C.F.R. § 202.1](https://www.copyright.gov/title37/202/37cfr202-1.html)

Inference and action:

- The package is publishable after a normal project-agnostic rewrite. Preserve the useful method: operate a real product through several behavioral stances, collect console/network/visual evidence, narrate the journey, deduplicate findings, and gate issue filing.
- Do not ship the current product URL, repository coordinates, route floor, competitor roster, private-repository screenshot rules, tracker policy, milestone, or automatic merge procedure as defaults. These are configuration or example content, not reusable provenance.
- No third-party attribution is required on the evidence found. Confidence is moderate, not absolute, because Git history cannot disprove unattributed influence or establish how every sentence was generated.

### `pipeline-research`

Coverage: `SKILL.md`, `references/filing.md`, `references/ledger.md`, `references/lenses.md`, `references/pipeline-areas.md`, and `references/report-template.md`.[^pipeline-source]

Facts:

- A single feature history introduces the skill, all five references, its rotating research lenses, product-code map, report template, ledger, and filing overlay under the maintainer's authorship.[^history]
- The skill says its load-bearing divergence/convergence steps mirror `adhd`; it also labels parallel isolation and rotating lenses as coming from that mechanism. The ledger and lenses repeat the attribution.[^pipeline-source]
- The rest of the package contains substantial Recruitr-specific original material: the job-pipeline taxonomy, concrete module map, three-score product constraint, DeepSeek-provider note, evidence scoring formula, brief schema, ledger schema, and tracker filing policy.[^pipeline-source]

Inference and action:

- Treat the fan-out/critic/rotation prose as uncertain provenance because its declared source is intentionally outside this audit. Do not copy it into the public catalog.
- Re-express the generic research method independently: configurable research areas; a source ledger and avoid-list; isolated research branches; a separate synthesis pass; evidence/applicability scoring; and explicit mapping to the consuming codebase. These are methods rather than protectable ownership of the idea. [Copyright Office Circular 31](https://www.copyright.gov/circs/circ31.pdf)
- Remove `adhd` from frontmatter, body, references, examples, and dependencies. Replace the Recruitr pipeline-area file with a user-supplied research map or a neutral example.
- The remaining project-authored references can be rewritten and licensed by the maintainer. If the excluded wording is retained instead, publication is blocked pending a separate provenance and permission audit.

### `to-issues`

Coverage: `SKILL.md`; it has no bundled reference files.[^to-issues-source]

Facts:

- The source labels itself a Recruitr copy that shadows `mattpocock-skills` `to-issues`.[^to-issues-source]
- Repository history says it was vendored and modified to replace Markdown blocker sections with GitHub-native dependency relationships.[^history]
- Comparison against Matt Pocock's April 2026 upstream version shows that most of the prose structure and wording remained while terminology, readiness labels, native dependency instructions, and an extra prototype-snippet exception were changed.[^to-issues-comparison]
- The upstream repository grants MIT permission to use, copy, modify, publish, distribute, sublicense, and sell the work, conditioned on including its copyright and permission notice in all copies or substantial portions. [Upstream MIT license](https://github.com/mattpocock/skills/blob/main/LICENSE)

Inference and action:

- The current file is an adapted work, not merely an implementation of an unprotectable idea. It needs attribution if substantial upstream wording or its template is retained.
- Safe retained-derivative path: add a `THIRD_PARTY_NOTICES.md` entry naming `mattpocock/skills`, Matt Pocock, the upstream `to-issues` source, MIT, and the local modifications; include the complete upstream MIT notice in the distributed repository or package. A top-level MIT license bearing only the catalog maintainer's copyright would not by itself preserve Matt Pocock's notice.
- Safe independent path: write a new skill from the requirements—vertical slices, human/agent disposition, approval, issue publication, and native dependencies—without consulting or reproducing the upstream prose during drafting. This avoids carrying copied expression, though provenance may still be acknowledged as project history.
- No separate permission request is needed for the retained-derivative path because MIT already grants permission when its condition is followed. The architecture decision about whether this should ship separately from `to-tickets` remains outside this audit.

### `pick-up-work` and `work-the-board`

Coverage: both `SKILL.md` files; neither has bundled references.[^agent-work-source]

Facts:

- Git history introduces both together as original repository orchestration and records later maintainer changes for native dependencies, issue-thread fidelity, review, screenshots, motion evidence, concurrency, and broader board scanning.[^history]
- `pick-up-work` names Superpowers capabilities for worktrees, TDD, and verification; `work-the-board` names its parallel-dispatch capability. The packages call those skills but do not include their `SKILL.md` files or source code.[^agent-work-source]
- The official Superpowers repository lists those capabilities and is MIT-licensed. [obra/superpowers](https://github.com/obra/superpowers)

Inference and action:

- Referencing or invoking an external skill is a dependency, not evidence that its source text was copied. No substantial Superpowers passage was identified in these packages.
- Publish after rewriting the orchestration around capability contracts and repository configuration. Document Superpowers as one compatible provider if desired, but do not make it the only runtime or silently vendor its files.
- If a future release bundles any Superpowers skill, that bundled content needs its own notice and MIT compliance review. The present conclusion covers only the seven source packages.

### `pick-up-human-task` and `work-the-human-board`

Coverage: both `SKILL.md` files; neither has bundled references.[^human-work-source]

Facts:

- Git history introduces each under the maintainer's authorship and describes the second as the sequential many-gate counterpart to the first.[^history]
- They invoke `triage` and `grill-with-docs`, and may hand off to `to-issues`; they do not bundle those external skill files.[^human-work-source]
- `triage` and `grill-with-docs` are distributed in Matt Pocock's MIT-licensed skills repository. [mattpocock/skills](https://github.com/mattpocock/skills)

Inference and action:

- The critical-path selection, live queue re-derivation, sequential human interaction, and skip-set coordinator can be retained as maintainer-authored concepts and rewritten in neutral prose.
- Treat external skills as replaceable capability dependencies. If the catalog later redistributes their files rather than merely invoking installed copies, preserve their upstream notices and audit the exact versions bundled.
- No additional permission is indicated for these two packages as currently constituted.

## Attribution and licensing plan

Before the first public release:

1. Choose and add an explicit catalog license. MIT is operationally simple and compatible with the known MIT-derived material, but the license choice belongs to the maintainer.
2. Add `THIRD_PARTY_NOTICES.md` even if `to-issues` is the only initial entry. Record upstream project, author, source link, license, retained files, and a short modification summary.
3. If retaining the derivative `to-issues`, include the complete Matt Pocock MIT notice with the distributed copy. Keep the attribution adjacent to the skill as well as in the repository notice file so selective installation does not strip it away.
4. Keep named external skills as dependencies/adapters unless their own licensed files are intentionally vendored. Installation documentation should distinguish "required separately" from "included here."
5. Require a release scan for source-project names, repository coordinates, personal filesystem paths, service URLs, tracker/project identifiers, field-option identifiers, and credentials. This audit intentionally reproduces none of the opaque configuration identifiers found in the sources.
6. Have the maintainer affirm ownership of the six project-authored packages and the Recruitr-authored modifications to `to-issues`. Commit metadata supports that conclusion but is not a rights assignment.

## Permission, rewrite, and exclusion summary

### Can be published after project-agnostic rewriting

- `dogfood`
- `pick-up-work`
- `work-the-board`
- `pick-up-human-task`
- `work-the-human-board`
- The independently authored product map, ledger, report, and filing ideas in `pipeline-research`, after the excluded mechanism is replaced
- Recruitr's original native-dependency additions to `to-issues`

### Must carry attribution if retained

- The substantial upstream wording, process structure, and template retained in `to-issues`: Matt Pocock / `mattpocock/skills`, MIT
- Any external skill files later bundled from Superpowers or Matt Pocock's catalog; mere named invocation does not bundle them

### Should be rewritten

- Every package's Recruitr-specific prose, defaults, paths, URLs, tracker details, and runtime/tool assumptions
- `pipeline-research`'s fan-out, isolation, critic, and lens-rotation instructions, written independently and without the excluded source
- `to-issues`, if the catalog prefers not to ship an attributed derivative or if `to-tickets` supersedes it

### Needs permission only if retained without an existing grant

- No reviewed component currently falls into this category. `to-issues` already has an MIT grant; the maintainer can license material they own.
- Any text ultimately found to come from the excluded source or from an unlicensed third party would enter this category. Until then, do not ship it.

### Exclude

- `adhd` itself
- Every dependency on or attribution to `adhd` in the public research skill; replace the mechanism rather than renaming it
- Source-project secrets, private filesystem paths, credentials, and opaque tracker identifiers

## Confidence and unresolved risk

Confidence is high for the `to-issues` derivative finding because the source declares the relationship, repository history describes the vendoring, and textual comparison confirms substantial overlap. Confidence is high that Superpowers, `triage`, and `grill-with-docs` are invoked rather than bundled in the reviewed packages.

Confidence is moderate for independent authorship of the remaining expression. Git history consistently identifies the maintainer and no other embedded source claim was found, but Git cannot establish whether prose was AI-generated, copied before commit, or influenced by an unrecorded source. That residual uncertainty is best managed by rewriting for project agnosticism, preserving known notices, and adding a maintainer ownership affirmation before release.

## Evidence inventory

The audit covered all 16 in-scope files:

| Skill | Files reviewed | History reviewed | Embedded source/dependency references checked |
| --- | ---: | --- | --- |
| `dogfood` | 5 | Introduction plus all later changes to skill/references | local friction-log shape; browser tooling; GitHub tracker/PR workflow |
| `pipeline-research` | 6 | Introduction plus later terminology change | excluded `adhd` references; `grill-with-docs`; `to-issues` |
| `to-issues` | 1 | Vendoring plus later terminology change | Matt Pocock upstream file and MIT license |
| `pick-up-work` | 1 | Introduction plus all later changes | Superpowers worktree/TDD/verification; `code-simplifier`; review capability |
| `work-the-board` | 1 | Introduction plus all later changes | Superpowers parallel dispatch; `pick-up-work` |
| `pick-up-human-task` | 1 | Introduction plus all later changes | `triage`; `grill-with-docs`; `to-issues` |
| `work-the-human-board` | 1 | Introduction plus later grill hardening | `pick-up-human-task`; `triage`; `grill-with-docs`; `to-issues` |

No source-project license, copying notice, or provenance file was found at the Recruitr repository root. That absence does not prevent its owner from relicensing material they own, but it does mean the source checkout itself supplies no public reuse grant. GitHub's no-license guidance therefore reinforces the need for an explicit license in the new catalog. [GitHub: Licensing a repository](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/licensing-a-repository)

[^dogfood-source]: Source snapshot reviewed: `recruitr/.claude/skills/dogfood/SKILL.md` and all four files under `dogfood/references/` (`competitors.md`, `filing.md`, `report-template.md`, `stances.md`). No absolute local path is recorded here.
[^pipeline-source]: Source snapshot reviewed: `recruitr/.claude/skills/pipeline-research/SKILL.md` and all five files under `pipeline-research/references/` (`filing.md`, `ledger.md`, `lenses.md`, `pipeline-areas.md`, `report-template.md`). `adhd` was not opened or analyzed.
[^to-issues-source]: Source snapshot reviewed: `recruitr/.claude/skills/to-issues/SKILL.md`. Upstream comparison source: [Matt Pocock's April 2026 `to-issues`](https://github.com/mattpocock/skills/blob/179a14e/skills/engineering/to-issues/SKILL.md).
[^agent-work-source]: Source snapshot reviewed: `recruitr/.claude/skills/pick-up-work/SKILL.md` and `recruitr/.claude/skills/work-the-board/SKILL.md`.
[^human-work-source]: Source snapshot reviewed: `recruitr/.claude/skills/pick-up-human-task/SKILL.md` and `recruitr/.claude/skills/work-the-human-board/SKILL.md`.
[^history]: Primary local evidence: `git log --follow`, introduction commit messages, author metadata, and later file history for each of the seven skills and both reference directories. Introduction dates are 2026-05-27 through 2026-05-31; recorded authors are Christian Dowell / `cdowell09`. The `to-issues` introduction explicitly says it was vendored from `mattpocock-skills`; the `pipeline-research` introduction explicitly describes its divergence/convergence mechanism as fused from the excluded source.
[^to-issues-comparison]: Primary comparison: a direct unified diff between the Recruitr source and the upstream April 2026 file. The diff changes the setup note, HITL/AFK vocabulary, readiness labels, dependency publication, and adds native GitHub commands plus a prototype-snippet exception; it leaves most earlier process prose and the issue template intact.

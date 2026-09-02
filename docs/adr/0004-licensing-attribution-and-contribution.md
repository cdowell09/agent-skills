# ADR 0004: Licensing, attribution, and contribution policy

Date: 2026-09-02
Status: Proposed (maintainer ratifies the license choice and the ownership affirmation)
Resolves: [#14 Set licensing, attribution, and contribution policy](https://github.com/cdowell09/agent-skills/issues/14)
Inputs: [Source licensing and provenance audit](../research/2026-07-12-source-provenance.md), [`to-issues` overlap audit](../research/2026-07-12-to-issues-overlap.md), [Source skill inventory](../research/2026-07-12-source-skill-inventory.md), [`npx skills` compatibility research](../research/2026-07-12-npx-skills-compatibility.md), [ADR 0001](0001-npx-skills-distribution-contract.md), [ADR 0002](0002-skill-boundaries-and-shared-contracts.md), [Capability configuration spec](../specs/capability-configuration.md)

Skill names used here (`pick-up-work`, `work-the-board`, `pick-up-human-task`, `work-the-human-board`, `dogfood`, `research-project`, `publish-findings`) are working names; a later naming pass may rename them.

## Context

The public catalog redistributes rewritten material whose provenance the audit classified three ways: six packages the repository history attributes to the maintainer alone; one package (`pipeline-research`, the source of `research-project`) whose divergence/convergence mechanism is expressly adapted from the excluded local skill and therefore has unresolved provenance; and one package (`to-issues`) that is a substantial derivative of Matt Pocock's MIT-licensed skill. The overlap audit retired `to-issues` as an independent skill and left issue #13 to choose between composing around upstream `to-tickets` and shipping a thin attributed extension.

Three distribution facts shape the policy. The repository has no license today, so under [GitHub's guidance](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/licensing-a-repository) nobody may reproduce or redistribute it yet. The installer copies exactly one `skills/<name>/` directory per selection, so a notice that lives only at the repository root does not reach an installed skill. And the ecosystem the catalog joins is uniformly MIT: the `skills` CLI ([Vercel, Inc.](https://github.com/vercel-labs/skills/blob/main/LICENSE)), Matt Pocock's catalog ([mattpocock/skills](https://github.com/mattpocock/skills/blob/main/LICENSE)), and Superpowers ([obra/superpowers](https://github.com/obra/superpowers/blob/main/LICENSE)) all carry MIT with a single copyright line.

The maintainer is one person. The policy must be enforceable by one reviewer and by the automation #15 builds, not by a governance process the catalog does not have.

## Decision

### 1. Catalog license: MIT

License the whole catalog under MIT (SPDX `MIT`). Maintainer ratifies.

Why MIT over the two credible alternatives:

| | MIT | Apache-2.0 | CC-BY-4.0 |
| --- | --- | --- | --- |
| Conditions ([choosealicense data](https://github.com/github/choosealicense.com/tree/gh-pages/_licenses)) | include copyright | include copyright, document changes, NOTICE handling | include copyright, document changes |
| Patent grant | none (implied only) | express | none; patent use excluded |
| Fit for skill content | prose plus shell/JS scripts | same | choosealicense marks it "not recommended for software"; skills ship `scripts/` |
| Mixing with the MIT-derived material and MIT ecosystem | identical terms; one notice format | compatible but two notice regimes, and consumers mixing MIT skills with an Apache catalog must carry a NOTICE file | compatible but a second, prose-oriented regime for the same kind of file |
| Primary user cost | copy one file | copy one file plus maintain NOTICE and change statements | attribution rules written for media, not installers |

MIT wins because every upstream the catalog touches already uses it, its sole obligation (keep the notice) is the same one the catalog must honour for retained third-party material, and the primary user already tolerates MIT dependencies. Apache-2.0's patent clause protects nothing here: the catalog is workflow prose and thin scripts. CC-BY-4.0 is wrong for a package that ships executable scripts.

Exact `LICENSE` file at the repository root: the unmodified MIT text with this copyright line, following the choosealicense placeholder format:

```text
MIT License

Copyright (c) 2026 Christian Dowell

Permission is hereby granted, free of charge, ... [remainder of the standard MIT text, verbatim]
```

The name matches the author recorded in the source project's git history per the provenance audit. The year is the first publication year; do not add ranges or "and contributors", because contributors keep their own copyright and are credited through git history.

Every `SKILL.md` declares the license in frontmatter using the Agent Skills specification's short form ([specification source](https://github.com/agentskills/agentskills/blob/main/docs/specification.mdx)):

```yaml
license: MIT
```

A skill that carries a third-party notice keeps `license: MIT` (the catalog's outbound license for the skill as a whole) and points to the notice through `metadata` (section 6). Do not write `license: MIT AND MIT` or bundle-file references in `license`; the installer only requires `name` and `description`, and a plain SPDX identifier is what tooling can check.

### 2. Attribution rules

**Repository-level record.** `THIRD_PARTY_NOTICES.md` at the repository root is the catalog's complete register of third-party material. It has two parts:

```markdown
# Third-party notices

## Register

| Project | Copyright holder | Source | License | Used in | Retained material | Modifications |
| --- | --- | --- | --- | --- | --- | --- |
| mattpocock/skills | Matt Pocock | https://github.com/mattpocock/skills/blob/<commit>/skills/engineering/to-issues/SKILL.md | MIT | skills/<name>/ | issue-body template; approval checklist wording | terminology; native GitHub dependency step; disposition mapping |

## Notices

### mattpocock/skills

[full upstream license text, verbatim, including its copyright line]
```

Every row records all seven fields: project, copyright holder as written upstream, source link pinned to a commit, SPDX identifier, the catalog skill directories that use it, what was retained (files, templates, wording), and a one-line modification summary. A row without a pinned commit or its verbatim notice below is a release blocker.

**Adjacent record.** Attribution must also live inside the skill's own directory, because selective installation copies only `skills/<name>/`. The mechanism is a `skills/<name>/NOTICE.md` file, not a section of `SKILL.md`. Reasons: it travels with every install mode (symlink and `--copy`) because the installer copies the directory recursively; it keeps license text out of the instruction context the agent loads; and `THIRD_PARTY_NOTICES.md` can be generated from the union of all `NOTICE.md` files and drift-tested, the same way shared protocol text is. `NOTICE.md` contains the same register row and the verbatim upstream notice. `SKILL.md` frontmatter points at it with `metadata.third-party-notices: NOTICE.md`.

**Retired `to-issues` derivative.** No ADR 0003 exists yet; #13 decides the shape. Both branches are settled here so #13 only has to pick one:

- *Composition branch (the overlap audit's preferred path):* the catalog recommends upstream `to-tickets` and expresses the agent-doable/human-gated disposition as its own policy, written without consulting the upstream file. No upstream wording is retained, so no license notice is required. Record the dependency in the disclosure table below, and record the project history (the source skill was vendored from `mattpocock/skills`) in `docs/provenance.md` as history, not as a notice.
- *Thin-extension branch:* if the extension retains any upstream wording, template, checklist text, or rule block, it is a derivative under MIT's notice condition. It ships `skills/<name>/NOTICE.md` with Matt Pocock's copyright line and the full MIT permission text, a `THIRD_PARTY_NOTICES.md` row, and `metadata.provenance: derivative`. The catalog-level `LICENSE` bearing only the maintainer's name does not satisfy the upstream condition on its own.

**Named-but-not-bundled external skills.** Upstream `to-tickets`, Superpowers capabilities (worktrees, TDD, verification, parallel dispatch), `triage`, and `grill-with-docs` are invoked by name, never copied. Invocation is a dependency, not a derivative, so they need a *dependency disclosure*, not a license notice: a `## External skills` table in `README.md`, mirrored in the `compatibility` field of each skill that uses one:

| External skill | Needed by | Required or optional | Install | License | Bundled here |
| --- | --- | --- | --- | --- | --- |
| `to-tickets` | `publish-findings` (complex-action handoff) | optional | `npx skills add mattpocock/skills --skill to-tickets` | MIT | No |
| Superpowers worktree/TDD/verification | `pick-up-work` | optional provider of required capabilities | per [obra/superpowers](https://github.com/obra/superpowers) | MIT | No |
| `triage`, `grill-with-docs` | `pick-up-human-task` | optional | `npx skills add mattpocock/skills --skill <name>` | MIT | No |

"Bundled here" is always "No" for the first release. Skills must not describe any external skill as the only way to satisfy a capability (ADR 0002: capabilities are semantic, providers are replaceable).

**Future vendoring.** A skill may copy third-party files only when all of these hold: the upstream license permits redistribution and modification (MIT, BSD, Apache-2.0, or CC-BY-4.0 for prose-only assets); the copy is pinned to an upstream commit; the verbatim upstream notice ships in `skills/<name>/NOTICE.md`; the register row exists; `metadata.provenance: derivative` is set; and review confirms nothing else in the directory came from an unrecorded source. Never vendor copyleft (GPL, AGPL, LGPL), share-alike, non-commercial, or unlicensed material. Apache-2.0 material additionally carries its upstream `NOTICE` contents and a statement of changes in `NOTICE.md`.

### 3. Provenance gate before first release

Three artefacts must exist before the first tag; #15 verifies them.

**Ownership affirmation.** The maintainer records the following, verbatim, in `docs/provenance.md` and quotes it in the first release pull request:

> I, Christian Dowell, affirm that I am the author of the source material from which the skills `pick-up-work`, `work-the-board`, `pick-up-human-task`, `work-the-human-board`, `dogfood`, and `research-project` were rewritten, and of the native-dependency and disposition modifications made to the source project's `to-issues` skill; that I hold the rights needed to license that material under the MIT License; that no part of the published catalog reproduces wording from the excluded local skill; and that every third-party passage I am aware of is recorded in `THIRD_PARTY_NOTICES.md`. Repository history supports this statement, but this affirmation, not the history, is the catalog's rights basis.

The audit was explicit that commit metadata is evidence of repository authorship rather than a rights assignment; this affirmation is what converts it into a license the catalog can grant.

**Clean-room rule for `research-project`.** The divergence/convergence mechanism (fan-out into isolated research branches, separate critic, lens rotation) is rewritten from a requirements list, not from the source text:

1. The #12 spec lists the behavioural requirements in its own words: configurable research areas, a source ledger and avoid-list, isolated divergent branches, a separate critic pass, evidence and applicability scoring, and mapping to the consuming codebase. These are methods, which copyright does not protect ([17 U.S.C. § 102(b)](https://www.copyright.gov/title17/92chap1.html)).
2. Whoever drafts the public `SKILL.md` and references works from that list and ADR 0002 only, without opening the source `pipeline-research` fan-out, isolation, critic, or rotation passages, its `lenses.md` or `ledger.md`, or the excluded skill; the draft's inputs are recorded in `docs/provenance.md`.
3. A second pass (the reviewer, or #15's automation) diffs the draft against the source package for shared runs of eight or more consecutive words and for the excluded skill's name; any hit is rewritten before merge.
4. `docs/provenance.md` records the date, the drafter, the inputs used, and the diff result.

The same rule applies whenever any other skill's rewrite touches a passage the audit marked as uncertain provenance.

**Release-hygiene scan.** A pre-release scan over `skills/` (strict) and `docs/` (identifiers, paths, endpoints, credentials only) fails the release on any hit in these classes:

| Class | What to detect | Examples of pattern shape |
| --- | --- | --- |
| Source-project names | the source project's name and product vocabulary; the excluded skill's name | denylist held outside the public tree (CI secret or git-ignored local file) so the catalog does not itself enumerate the vocabulary; the excluded skill name is always denied |
| Repository coordinates | any `owner/repo` other than `cdowell09/agent-skills` and the cited upstreams | `github.com/<owner>/<repo>`, `gh ... -R <owner>/<repo>` |
| Personal paths | absolute or home-relative workstation paths | `/Users/`, `/home/<name>/`, `~/` followed by a project path |
| Service URLs | production, staging, health, or API endpoints not on the documentation allowlist | any `https://` host outside `github.com`, `docs.github.com`, `agentskills.io`, and cited upstream hosts |
| Tracker/project identifiers | GitHub Project numbers and node IDs | `PVT_`, `PVTF_`, `PVTSSF_`, `PVTI_` prefixes; `projects/<number>` |
| Field-option identifiers | opaque single-select option IDs and status IDs | eight-character hexadecimal tokens adjacent to `optionId`, `fieldId`, or `status` |
| Credentials | tokens, keys, `.env` content | `ghp_`, `github_pat_`, `sk-`, `AKIA`, `Bearer `; any file named `.env*` |

`docs/research/` is exempt from the source-project-name class only, because the existing research already names the source project and that history is deliberate; it is not exempt from any other class. The scan is a #15 deliverable; until it exists, the maintainer runs the equivalent `rg` checks by hand and records the run in the release PR.

### 4. Contribution policy

`CONTRIBUTING.md` at the repository root, with these sections:

**Licensing of contributions.** Inbound equals outbound: by opening a pull request the contributor licenses the contribution under the catalog's MIT license, which is the default GitHub already applies to contributions to a licensed repository ([GitHub Terms of Service, D.6](https://docs.github.com/en/site-policy/github-terms/github-terms-of-service#6-contributions-under-repository-license)). No CLA. No DCO: a `Signed-off-by` trailer would add a git-level ritual and a bot check without addressing the catalog's actual risk, which is a contributor pasting a third-party or project-specific skill. The pull-request template's provenance declaration (below) targets that risk directly and is what the reviewer checks. Revisit DCO only if the catalog gains multiple maintainers or an organisational owner.

**Provenance declaration.** Every pull request that adds or changes a skill checks one of: "I wrote this material myself", "this material derives from `<project>` under `<license>` and `NOTICE.md` plus the register row are included", or "this material was produced with AI assistance from my own prompts and I have reviewed it for copied text". A PR without a checked box is not reviewed.

**AI-assisted contributions.** Permitted and expected. The contributor must say so in the declaration, must have read every line they submit, and remains responsible for it. Contributors must not paste another skill, README, or blog post as a prompt and submit the output; if a generation was seeded from third-party text, the third-party rules apply to the output.

**Rules for a contributed skill.** It must be self-contained under `skills/<name>/` with no runtime reference to sibling paths (ADR 0001); pass `skills-ref validate` and the catalog's own validation; be repository-agnostic, with every concrete label, repository, project field, path, command, and URL routed through capability configuration at `.agent-skills/config.yaml` rather than embedded as a default; pass the release-hygiene scan; declare `license`, `metadata.provenance`, and any `NOTICE.md`; express runtime needs as semantic capabilities marked required or optional with fallbacks; and, if it participates in a shared protocol, be generated from the canonical protocol source rather than hand-copied.

**Review expectations.** The maintainer reviews every PR; there is no second reviewer for the first release. Review covers the provenance declaration, the hygiene-scan output, spec validation, an install-and-run check in the pilot consumer for behavioural changes, and any `NOTICE.md`. First response within two weeks; a PR idle for sixty days may be closed. Skill-behaviour changes need a linked issue; documentation fixes do not.

**What maintainers will not accept.** Copied third-party skills without notices, or with notices but no pinned source; skills, references, or examples with project-specific defaults, real repository coordinates, opaque tracker identifiers, or personal paths; anything of unresolved provenance, including material described as "adapted from" an unnamed or unlicensed source; changes that weaken a core safety behaviour that ADR 0002 says configuration cannot remove; new catalog-level runtime files; a `SKILL.md` at the repository root; or copyleft, share-alike, non-commercial, or unlicensed material in any form.

### 5. Third-party-content boundaries

**May be included without a notice:** prose the maintainer or a contributor wrote; protocol text generated from the catalog's canonical source; scripts written for the catalog; factual reference data such as public product names and URLs used as neutral examples; methods (vertical slicing, worktree isolation, claim-and-release, divergent-then-convergent research) expressed in the catalog's own words; and standard tool invocations (`gh`, `git`, `npx skills`).

**Requires a notice (register row plus adjacent `NOTICE.md`):** any retained wording, headings-plus-structure, checklist language, rule block, quiz, issue-body template, report template, or code taken from a licensed source, including MIT sources; any file copied from an external skill; any prose generated by seeding a model with a third-party file. "Substantial" is judged by the upstream's notice condition, and the catalog resolves doubt in favour of adding the notice.

**Excluded outright:** the excluded local skill and every reference to, credit for, or dependency on it; any passage of uncertain provenance not yet rewritten under the clean-room rule; material whose license is missing, copyleft, share-alike, or non-commercial; and all credentials, personal paths, service endpoints, repository coordinates, and opaque tracker or field identifiers from the source project, even inside examples, comments, or fixtures.

### 6. Files added at implementation time

| File | Location | Content | Decision |
| --- | --- | --- | --- |
| `LICENSE` | root | Verbatim MIT with `Copyright (c) 2026 Christian Dowell` | Add. Root placement is what GitHub's license detection expects |
| `THIRD_PARTY_NOTICES.md` | root | Register table plus verbatim notices (section 2); generated from all `skills/*/NOTICE.md` and drift-tested | Add even if initially empty apart from a header explaining the format |
| `skills/<name>/NOTICE.md` | per derivative skill | Register row plus verbatim upstream notice | Add only where third-party material is retained |
| `CONTRIBUTING.md` | root | Section 4 outline | Add |
| `CODE_OF_CONDUCT.md` | root | Contributor Covenant 2.1, verbatim, with the enforcement contact filled in and the required attribution link kept ([Contributor Covenant 2.1 source](https://github.com/EthicalSource/contributor_covenant/blob/release/content/version/2/1/code_of_conduct.md)) | Adopt. It costs one file and one email address, GitHub's community profile expects it, and setting expectations for a public tracker up front is cheaper than after an incident. Enforcement contact: an address the maintainer ratifies |
| `SECURITY.md` | root | Minimal: what counts as a vulnerability in a skill (instructions that could exfiltrate data, run unreviewed commands, or bypass a configured approval gate); report through GitHub private vulnerability reporting; supported versions are the default branch and latest tag; no bounty; seven-day acknowledgement target | Add |
| `docs/provenance.md` | docs | Ownership affirmation; clean-room records; history note for the retired `to-issues` derivative; hygiene-scan runs before automation exists | Add |
| `.github/PULL_REQUEST_TEMPLATE.md` | `.github` | Provenance declaration checkboxes; hygiene-scan and validation checklist | Add |

`SKILL.md` frontmatter for every published skill, using only spec-defined fields (`metadata` is a string-to-string map):

```yaml
---
name: pick-up-work
description: ...
license: MIT
compatibility: GitHub repository with Issues and Projects; gh CLI; git worktrees. Optional providers: Superpowers worktree/TDD/verification skills.
metadata:
  author: Christian Dowell
  version: "0.1.0"
  provenance: original            # or: derivative
  third-party-notices: NOTICE.md  # present only when provenance is derivative
---
```

`metadata.provenance` takes exactly two values. `original` means every file in the directory is catalog-authored under the affirmation in section 3. `derivative` means the directory contains a `NOTICE.md` and a register row. Validation in #15 rejects a directory where these disagree (a `NOTICE.md` with `provenance: original`, or `provenance: derivative` without one).

## Alternatives considered

- **Apache-2.0.** Rejected: an express patent grant buys nothing for workflow prose, and it would force consumers to carry NOTICE handling for one dependency in an otherwise MIT install set.
- **CC-BY-4.0, or a dual "MIT for scripts, CC-BY for prose" split.** Rejected: skills mix prose and scripts in one directory, and a per-file split cannot be expressed in one `license` frontmatter value or checked by tooling.
- **Attribution as a `## Attribution` section inside `SKILL.md`.** Rejected in favour of `NOTICE.md`: license text in `SKILL.md` enters every agent context that uses the skill, and a section is easier to lose in a rewrite than a file the drift test enumerates.
- **DCO sign-off.** Rejected for a solo-maintained catalog; the trigger for reconsidering it is in section 4.
- **Skipping a code of conduct.** Rejected: the cost is one file and the issue tracker is already the intake surface for AI-authored contributions.
- **Relying on git history instead of an ownership affirmation.** Rejected: the audit says history is evidence, not a rights grant.
- **A public denylist of source-project terms inside the repository.** Rejected: it would embed the vocabulary the scan exists to remove.

## Consequences

- **#15 (validation and release):** owns the release-hygiene scan (section 3 table), the `metadata.provenance`/`NOTICE.md` consistency check, the generation and drift test of `THIRD_PARTY_NOTICES.md` from `skills/*/NOTICE.md`, the eight-word shingle diff for the clean-room rule, and a release gate that refuses to tag without `LICENSE`, the affirmation in `docs/provenance.md`, and a clean scan.
- **#17 (documentation and launch):** owns writing `LICENSE`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, the PR template, `docs/provenance.md`, and the README `## External skills` disclosure table; must present install commands that distinguish "included here" from "install separately".
- **#13 (ticket-decomposition provider):** picks one of the two `to-issues` branches in section 2; whichever it picks, the attribution consequence is already specified.
- **#12 (`research-project` spec):** must contain the requirements list the clean-room rule drafts from, and must not quote the source mechanism.
- **#8 (capability configuration):** inherits the rule that every concrete identifier lives in `.agent-skills/config.yaml`, which is what makes the hygiene scan's zero-tolerance rule for `skills/` achievable.
- **Every skill spec (#9, #10, #11, #18):** must include the frontmatter block in section 6 in its `SKILL.md` outline and list any external skill in the disclosure table.

## Open items

1. Maintainer ratifies MIT and the exact copyright line (section 1).
2. Maintainer records the ownership affirmation verbatim (section 3) before the first tag.
3. Maintainer supplies the enforcement contact for `CODE_OF_CONDUCT.md` and confirms GitHub private vulnerability reporting is enabled for the repository so `SECURITY.md` can point to it.
4. #13 chooses the `to-issues` branch; if the thin-extension branch, the pinned upstream commit for the register row must be the one the derivative was actually compared against, not `main`.
5. The eight-word shingle threshold for the clean-room diff is a starting point; #15 may tune it after the first run and should record the chosen value in `docs/provenance.md`.
6. If any future skill vendors Apache-2.0 material, extend `NOTICE.md` format with a changes statement before the first such vendoring, not after.

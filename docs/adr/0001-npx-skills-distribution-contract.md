# ADR 0001: Distribute through the existing `npx skills` ecosystem

Date: 2026-07-12
Status: Accepted
Resolves: [#6 Decide the npx skills distribution contract](https://github.com/cdowell09/agent-skills/issues/6)
Inputs: [`npx skills` compatibility research](../research/2026-07-12-npx-skills-compatibility.md)

## Context

The catalog must be discoverable, selectable, installable, updatable, and removable through the existing `skills` CLI for both Codex and Claude Code. The research found that the installer covers that lifecycle but has no dependency resolver, so anything one skill needs at runtime must live inside its own directory.

## Decision

Use the existing `npx skills` ecosystem and publish one flat, standards-compliant catalog. Build no custom installer.

### Repository contract

```text
agent-skills/
├── README.md
├── LICENSE
├── THIRD_PARTY_NOTICES.md
├── skills/
│   └── <skill-name>/
│       ├── SKILL.md
│       ├── references/   # optional, owned by this skill
│       ├── scripts/      # optional, owned by this skill
│       └── assets/       # optional, owned by this skill
├── docs/                 # catalog documentation
├── scripts/              # catalog maintenance only
└── tests/                # catalog and lifecycle verification
```

- Do not place a catalog-level `SKILL.md` at the repository root.
- Do not make installed skills reference catalog-level runtime files.
- Use specification-compliant `name` and `description` fields. Put real environment requirements in `compatibility`, include license and provenance metadata where applicable, and treat `metadata.version` as descriptive rather than a compatibility constraint.

### Discovery, selection, and installation

- Guarantee direct GitHub discovery through `npx skills add cdowell09/agent-skills --list`. A `skills.sh` listing is promotion, not part of the installation contract.
- Document exact selection with `--skill <name> --yes`. Never document bare `--yes`, because it selects the whole catalog.
- Project scope is the default so the consuming repository can version `skills-lock.json`. Document `--global` as an explicit convenience.
- Support Codex and Claude Code from the same portable `SKILL.md`, and target both explicitly in canonical commands:

```bash
npx skills add cdowell09/agent-skills \
  --skill <skill-name> \
  --agent codex \
  --agent claude-code \
  --yes
```

- Accept the installer's shared-copy-plus-symlink default. Document `--copy` as the portability fallback. Add no catalog-specific file-placement logic.
- Installation only installs skills. Repository capability configuration is discovered and validated at runtime, with safe setup guidance when required mappings are missing.

### Composition contract

- A public skill is self-contained by default: its runtime files live inside its own skill directory, and it has no implicit dependency on an uninstalled sibling.
- Two explicit worker/coordinator companion dependencies are allowed:
  - the parallel agent-board coordinator requires the single agent-work worker;
  - the sequential human-board coordinator requires the single human-gate worker.
- Coordinators remain independently selectable but are not usable without their worker. Documentation installs and updates each pair together, and each coordinator preflights its worker before claiming work or causing other side effects.
- The installer does not resolve these dependencies. Do not invent a dependency metadata field or rely on cross-skill filesystem paths.

### Update and removal

- Canonical unpinned installs track the repository default branch so `npx skills update` discovers published changes. Immutable tags are opt-in pins.
- Companion pairs are documented with paired update commands.
- Use upstream `skills remove` behavior. Pilot-test the possible stale project `skills-lock.json` entry; if confirmed, report it upstream and document safe cleanup rather than adding removal tooling.

### Release verification

The adjacent pilot repository gates releases on: Agent Skills specification validation; catalog discovery and exact selection; project installs for Codex and Claude Code; default symlink and `--copy` modes; default-branch updates; removal including the project-lock caveat; and companion preflight behavior for both pairs.

### Custom-tooling threshold

Build a companion installer only when the pilot demonstrates a reproducible failure against a non-negotiable requirement, the gap cannot reasonably be fixed upstream or handled through documentation, and it concerns dependency resolution, coordinated versioning, install-time configuration, runtime transformation, policy enforcement, or unsafe lifecycle behavior. No such gap is currently demonstrated.

## Consequences

- Every downstream specification must keep runtime files inside the skill directory and express shared behavior through generated, drift-tested copies rather than shared paths.
- Companion-pair documentation and preflight behavior become acceptance criteria for the work-execution and human-gate waves.
- The validation strategy owns the installer lifecycle tests listed above.

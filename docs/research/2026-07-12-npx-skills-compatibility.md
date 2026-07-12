# `npx skills` compatibility constraints for a public multi-skill catalog

## Research basis

This review covers the official `skills` CLI at version **1.5.16**, repository revision [`3176ae4`](https://github.com/vercel-labs/skills/tree/3176ae424e50bb7d3f20a7e085f31912b3f325d2), released July 10, 2026. The CLI package requires Node.js 18 or newer and is MIT-licensed. [Official package metadata](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/package.json#L1-L5) [Node and license metadata](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/package.json#L122-L143)

## Decision

**Use the existing `npx skills` ecosystem for the first release. Do not build a custom installer.**

It already supports a public repository containing independently selectable skills, explicit Codex and Claude Code targets, project or global installation, copy or symlink behavior, and update/removal commands. The catalog should conform to the standard layout and treat each independently installable skill directory as a self-contained distribution unit.

The one material design constraint is that `skills` has **no skill dependency resolver**. Shared behavior can be composed at the instruction level, but installing one skill does not automatically install another. A custom tool becomes justified only if automatic dependency/version resolution, transactional suite updates, or install-time project configuration becomes a real requirement.

## Repository layout and discovery

### Observed behavior

- A repository does not need to be published as an npm package. `skills add` accepts GitHub shorthand, a full GitHub URL, a direct path within a repository, GitLab, generic Git URLs, and local paths. [Official source-format documentation](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/README.md#L28-L48)
- The conventional catalog layout is `skills/<skill-name>/SKILL.md`. The CLI also walks one additional category level, so `skills/<category>/<skill-name>/SKILL.md` works. It searches several agent-specific skill directories too. A root `SKILL.md` is treated as a single direct skill and normally shadows nested discovery; `--full-depth` opts into a broader recursive search. [Discovery documentation](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/README.md#L369-L436) [Discovery implementation](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/skills.ts#L145-L289)
- A discovered `SKILL.md` must provide string-valued `name` and `description` fields. The CLI hides `metadata.internal: true` skills unless explicitly selected or `INSTALL_INTERNAL_SKILLS=1` is set. [Parser implementation](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/skills.ts#L65-L96)
- The CLI can also discover skill paths declared by Claude plugin manifests. In the installer, those declarations add discovery paths and optional display grouping; they do not create dependency semantics. [Plugin-manifest implementation](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/plugin-manifest.ts#L38-L177)
- `skills find` is separate from repository discovery: it queries the remote `skills.sh` search API. Direct installation by repository URL works independently of whether a skill appears in search results. [Search implementation](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/find.ts#L22-L83)

### Implication for this catalog

Use this layout:

```text
agent-skills/
├── README.md
├── LICENSE
└── skills/
    ├── pick-up-work/
    │   ├── SKILL.md
    │   ├── references/      # optional, owned by this skill
    │   ├── scripts/         # optional, owned by this skill
    │   └── assets/          # optional, owned by this skill
    └── work-the-board/
        └── SKILL.md
```

Do not put a catalog-level `SKILL.md` at the repository root. A Claude marketplace manifest may be added later for Claude plugin distribution or visual grouping, but it is not required for `npx skills add`.

The public README should advertise direct repository installation first. Search-directory presence is useful discovery, not a prerequisite for installation.

## Selection and installation behavior

### Observed behavior

- `npx skills add <owner/repo> --list` lists discovered skills. `--skill <name>` can be repeated to select exact names, and `--skill '*'` selects all. With multiple discovered skills and no selector, interactive mode prompts; `--yes` instead selects every discovered skill. [CLI options and examples](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/README.md#L50-L88) [Selection implementation](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/add.ts#L1264-L1348)
- The shorthand `<owner>/<repo>@<skill-name>` also selects a skill. Git refs use a URL fragment, such as `owner/repo#tag`, and can be combined with a skill selector as `owner/repo#tag@skill-name`. [Source parser](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/source-parser.ts#L117-L227) [GitHub shorthand parsing](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/source-parser.ts#L294-L324)
- Project scope is the default; `--global` installs at user scope. `--agent` accepts explicit runtime targets. [Scope documentation](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/README.md#L90-L95)
- Codex uses `.agents/skills/` in a project and `~/.codex/skills/` globally. Claude Code uses `.claude/skills/` in a project and its Claude configuration directory globally. Both are first-class target values: `codex` and `claude-code`. [Supported-agent table](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/README.md#L238-L263) [Agent definitions](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/agents.ts#L7-L13)
- The default multi-target method copies the selected skill directory into a canonical `.agents/skills/<name>` location and symlinks agent-specific locations to it. `--copy` creates independent copies instead, and a failed symlink automatically falls back to copying. [Installation-method documentation](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/README.md#L97-L103) [Installer implementation](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/installer.ts#L265-L410)
- Installation recursively carries files inside the selected skill directory, preserving scripts and resources. It omits `.git`, Python cache/package directories, and `metadata.json`; repository-level siblings are not part of the selected skill. [Directory-copy implementation](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/installer.ts#L423-L505)

### Recommended commands

```bash
# Inspect the catalog
npx skills add cdowell09/agent-skills --list

# Install one skill into the current project for both supported runtimes
npx skills add cdowell09/agent-skills \
  --skill pick-up-work \
  --agent codex \
  --agent claude-code \
  --yes

# Install globally instead
npx skills add cdowell09/agent-skills \
  --skill pick-up-work \
  --agent codex \
  --agent claude-code \
  --global \
  --yes
```

Always pair `--yes` with `--skill` in documentation unless the intent is to install the entire catalog.

## Metadata and portability

### Standard requirements

The Agent Skills specification defines a skill as a directory containing `SKILL.md`; `scripts/`, `references/`, and `assets/` are optional. Required frontmatter fields are `name` and `description`. Optional standard fields are `license`, `compatibility`, `metadata`, and experimental `allowed-tools`. The `name` must be lowercase kebab-case, no more than 64 characters, and match its directory; `description` is limited to 1,024 characters. [Agent Skills specification](https://agentskills.io/specification#directory-structure) [Frontmatter constraints](https://agentskills.io/specification#frontmatter)

### Compatibility boundary

Basic skills work in both Claude Code and Codex, but runtime-specific features do not have equal support. The CLI's current compatibility table marks `allowed-tools` as supported by both, while `context: fork` and hooks are supported by Claude Code but not Codex. [Official compatibility matrix](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/README.md#L460-L470)

### Implication for this catalog

- Keep the shared `SKILL.md` behavior within the basic Agent Skills specification.
- Use `compatibility` to declare real environment requirements such as GitHub, `gh`, Git, network access, or an agent capability. Do not use it as a dependency resolver.
- Prefer capability language in instructions over runtime-specific tool names. Put Codex- or Claude-specific guidance in small referenced adapter files only when behavior truly differs.
- Validate release candidates with the specification's `skills-ref validate` command; the installer itself is more permissive than the full naming constraints. [Specification validation guidance](https://agentskills.io/specification#validation)
- A `metadata.version` value is descriptive only. The current installer update model tracks source refs and folder hashes, not semantic version ranges declared by a skill.

## Update and removal behavior

### Updates

- Project installs create a repository-level `skills-lock.json` containing source, source type, optional ref, skill path, and a content hash. The file is intended for version control and is written deterministically. [Project lock schema](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/local-lock.ts#L8-L75)
- Global installs use a user-state lock containing source URL, optional ref, skill path, and the GitHub tree hash for the whole skill folder. [Global lock schema](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/skill-lock.ts#L10-L55)
- `skills update [names...]` can target project, global, or both. Project updates reinstall only the recorded skill path; global GitHub updates compare the recorded skill-folder hash. [Update documentation](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/README.md#L147-L182) [Project update implementation](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/update.ts#L487-L643)
- Local-path installs are not automatically updated. Some generic Git or legacy entries without a recorded skill path cannot be checked automatically. If an upstream skill disappears, interactive update offers removal; non-interactive update reports it and skips deletion. [Update limitations](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/update.ts#L166-L271)

This means releases may follow the repository default branch for normal “latest” behavior, or users may pin a tag/ref. The CLI does not resolve semver ranges between skills or atomically upgrade a compatible suite.

### Removal

`skills remove` supports name selection, agent selection, global scope, and all-skills removal. It removes agent-specific copies/symlinks and removes the canonical copy only when no remaining detected agent uses it. [Removal documentation](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/README.md#L184-L231) [Removal implementation](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/remove.ts#L25-L243)

One current v1.5.16 caveat should be tested in the pilot repository: removal updates the global lock but the removal implementation does not call the available project-lock removal helper. A project-scoped `skills-lock.json` may therefore retain a stale entry after removal. This is a candidate upstream bug or documentation issue, not a reason to fork the installer. [Removal lock handling](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/remove.ts#L231-L242) [Project-lock removal helper](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/local-lock.ts#L170-L183)

## Dependency and composition limits

### Facts

- The standard frontmatter has no dependency field. [Frontmatter fields](https://agentskills.io/specification#frontmatter)
- The CLI's skill and lock schemas have no dependency graph, version constraint, bundle, or prerequisite representation. [Skill model](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/types.ts#L75-L86) [Lock model](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/skill-lock.ts#L10-L55)
- Explicit selection filters the discovered skills by name; it does not expand prerequisites. [Selection filter](https://github.com/vercel-labs/skills/blob/3176ae424e50bb7d3f20a7e085f31912b3f325d2/src/skills.ts#L294-L310)
- Only files inside the selected skill directory are installed. A relative reference to a catalog-level sibling is therefore not independently portable.

### Design consequences

For an independently installable catalog:

1. Each skill must work alone or fail with a precise prerequisite message.
2. Files required at runtime must live inside that skill's own directory.
3. Small shared instructions may be duplicated or generated into each published skill during release.
4. A skill may recommend another skill by name, but it cannot assume the installer resolved or installed it.
5. If a workflow genuinely requires several skills, document a copy-pasteable multi-`--skill` install command. Treat the set as a documented bundle, not an enforced dependency graph.

Claude plugin manifests can group skills in prompts and listings, but that grouping should not be mistaken for composition or prerequisite installation.

## What would justify custom tooling

Build a custom CLI or a thin companion tool only if a validated requirement cannot be expressed through self-contained skill folders and documented `npx skills` commands. Genuine triggers are:

1. **Dependency resolution:** installing one skill must automatically resolve transitive skill prerequisites, conflicts, or optional features.
2. **Coordinated versioning:** several skills require compatible version ranges, atomic upgrades, rollback, or a catalog lock independent of Git refs.
3. **Install-time configuration:** installation must discover repository policy, generate project configuration, validate tracker mappings, migrate old config, or conduct a secure credential setup.
4. **Runtime transformation:** the same source must be compiled differently for Codex and Claude rather than using portable instructions plus small adapters.
5. **Policy enforcement beyond validation:** distribution requires organization-specific signature verification, provenance policy, or blocking security checks that the upstream installer does not provide.
6. **Unsupported lifecycle behavior:** pilot testing proves that update/removal semantics cannot be fixed upstream or safely documented.

None of these is yet established for the first release. Configuration discovery can remain runtime behavior inside the installed skills; it does not require an installer extension.

## Acceptance constraints for the publication plan

- Publish one public repository with self-contained skills under `skills/<name>/`.
- Keep every published skill individually selectable and usable.
- Target `codex` and `claude-code` explicitly in documented commands.
- Use the common Agent Skills frontmatter and validate it before release.
- Do not rely on cross-skill filesystem paths or implicit prerequisite installation.
- Test install, update, and remove in the adjacent pilot repository, including both symlink and `--copy` modes and the project-lock removal caveat.
- Reconsider custom tooling only when one of the explicit triggers above appears in a concrete pilot requirement.

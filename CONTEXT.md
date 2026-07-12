# Public Skills

This context defines the language for extracting reusable, open-source agent skills from project-local workflows.

## Language

**Repository-agnostic skill**:
A skill with no embedded product identity, repository coordinates, private paths, workflow IDs, or domain-specific defaults. It may depend on declared platform capabilities such as GitHub, `gh`, browser control, or subagents.
_Avoid_: Project-agnostic skill, universal skill

**Capability configuration**:
Repository-owned mappings and policies that connect a repository-agnostic skill to concrete tracker fields, labels, commands, paths, verification rules, and runtime tools.
_Avoid_: Hard-coded defaults, portability layer

**Skill family**:
A set of composable skills that share reusable workflow primitives while keeping single-item, batch, human-gated, dogfooding, and research responsibilities distinct.
_Avoid_: Monolithic skill, skill bundle

**Distribution package**:
A versioned public skill catalog compatible with the existing `npx skills` installer for discovering, selecting, installing, updating, and removing skills from supported agent runtimes. A custom installer is justified only by a demonstrated compatibility gap.
_Avoid_: Folder structure, source archive

**Public catalog**:
The single open-source repository that owns the skill family, shared capability configuration, documentation, and release metadata. Each skill remains independently installable.
_Avoid_: One repository per skill, all-or-nothing bundle

**Primary user**:
A solo developer or software team using GitHub with Codex, Claude, or both, who wants to reuse the workflows in another repository. The first release does not target non-software workflows or non-GitHub trackers.
_Avoid_: Every project type, every tracker

**Pilot consumer**:
A software repository adjacent to the public catalog that validates installation, configuration, and real workflow behavior before release. Recruitr supplies the source skills but will not consume or track the extracted versions.
_Avoid_: Recruitr migration, reference implementation

**Source skill**:
A project-specific local skill being evaluated for extraction into the public skill family. The local `adhd` skill is explicitly excluded from this set.
_Avoid_: Public skill, candidate by default

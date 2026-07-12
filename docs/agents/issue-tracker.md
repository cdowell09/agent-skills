# Issue tracker: GitHub

Issues and PRDs live in GitHub Issues at `cdowell09/agent-skills`.

Use `gh` by default. The connected GitHub plugin is the fallback when CLI authentication or capability coverage is unavailable.

## Conventions

- Create, read, comment on, label, and close work as GitHub issues.
- Infer the repository from the local `origin` remote.
- Search open and closed issues before creating potential duplicates.
- Pull requests are not a triage request surface.

## Wayfinding operations

- A map is an issue labelled `wayfinder:map`.
- Tickets are child issues labelled `wayfinder:research`, `wayfinder:prototype`, `wayfinder:grilling`, or `wayfinder:task`.
- Prefer GitHub-native sub-issue and blocking relationships.
- Fall back to `Part of #N` and `Blocked by: #N` body conventions when the available GitHub interface lacks those operations.
- Claim a ticket by assigning it before beginning work.
- Resolve it with an answer comment, close it, then append a linked gist to the map's Decisions-so-far section.

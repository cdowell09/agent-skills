# agent-skills

Reusable, repository-agnostic agent workflow skills for GitHub, Codex, and Claude.

This repository currently holds the planning record for the catalog. The skills themselves ship in the release waves described in [ADR 0007](docs/adr/0007-release-sequencing-and-readiness-gates.md); nothing is installable yet.

## Planning record

The planning map is [issue #1](https://github.com/cdowell09/agent-skills/issues/1). Its research tickets, decisions, and specifications live here:

### Research

- [Source skill inventory](docs/research/2026-07-12-source-skill-inventory.md)
- [`npx skills` compatibility constraints](docs/research/2026-07-12-npx-skills-compatibility.md)
- [`to-issues` overlap audit](docs/research/2026-07-12-to-issues-overlap.md)
- [Source licensing and provenance audit](docs/research/2026-07-12-source-provenance.md)

### Decisions

- [ADR 0001: Distribute through the existing `npx skills` ecosystem](docs/adr/0001-npx-skills-distribution-contract.md)
- [ADR 0002: Skill boundaries and shared contracts](docs/adr/0002-skill-boundaries-and-shared-contracts.md)
- [ADR 0003: Ticket decomposition provider](docs/adr/0003-ticket-decomposition-provider.md)
- [ADR 0004: Licensing, attribution, and contribution policy](docs/adr/0004-licensing-attribution-and-contribution.md)
- [ADR 0005: Validation and pilot strategy](docs/adr/0005-validation-and-pilot-strategy.md)
- [ADR 0006: Pilot consumer and acceptance scenarios](docs/adr/0006-pilot-consumer.md)
- [ADR 0007: Release sequencing and readiness gates](docs/adr/0007-release-sequencing-and-readiness-gates.md)

### Specifications

- [Capability configuration](docs/specs/capability-configuration.md): the repository-owned `.agent-skills/config.yaml` contract, validation receipts, and claim protocol
- [Work execution](docs/specs/work-execution.md): the single agent-work worker and its parallel board coordinator
- [Human gate](docs/specs/human-gate.md): the single human-gate worker and its sequential board coordinator
- [Findings publication](docs/specs/findings-publication.md): the findings artifact contract, deduplication, approval, and publication receipts
- [Dogfooding](docs/specs/dogfooding.md): product dogfooding that produces findings artifacts
- [Project research](docs/specs/project-research.md): repeated project research that produces findings artifacts

ADRs marked "Proposed" carry maintainer-ratification items; ADR 0007 collects them into one checklist.

## Working conventions

See `AGENTS.md`, `CONTEXT.md`, and `docs/agents/`.

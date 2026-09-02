# ADR 0002: Skill boundaries and shared contracts

Date: 2026-08-07
Status: Accepted
Resolves: [#7 Design skill boundaries and shared contracts](https://github.com/cdowell09/agent-skills/issues/7)
Inputs: [Source skill inventory](../research/2026-07-12-source-skill-inventory.md), [`to-issues` overlap audit](../research/2026-07-12-to-issues-overlap.md), [ADR 0001](0001-npx-skills-distribution-contract.md)

## Context

Given the fixed distribution contract, the seven source workflows must be divided into public skill units, explicit companion relationships, shared semantic protocols, repository-owned capability configuration, and runtime adapters, without hidden filesystem dependencies.

## Decision

### Public jobs

Publish seven independently installable, job-centered skills. These are roles; downstream specification tickets choose their public names:

1. Single agent-work-item worker
2. Parallel agent-board coordinator
3. Single human-gate worker
4. Sequential human-board coordinator
5. Product dogfooding
6. Repeated project research
7. Findings publication

The `to-issues` source does not define an eighth job. Issue #13 decides which external provider or thin extension supplies optional ticket decomposition and which differentiated routing behavior survives.

### Composition

The two worker/coordinator pairs are the only required companion relationships. A worker owns one issue end to end. Its coordinator preflights the worker before side effects, calculates the ready frontier, claims and dispatches work, reconciles receipts, reevaluates the queue, and recovers failures without duplicating worker behavior. Each pair uses one configured eligibility and priority definition.

Dogfooding and research produce conforming findings artifacts. They hand off automatically when the findings publisher is configured and available, but remain complete and independently installable without it. Publication failure is separate from discovery failure. The publisher also accepts any conforming artifact as a standalone input.

Every installed skill remains self-contained. Essential behavior has a self-contained baseline; external helpers may provide optional capabilities. Shared protocols have one canonical authoring source, are generated into participating packages, and are tested for drift. Runtime behavior never depends on an uninstalled sibling path.

### Shared contracts

- Setup validation is durable, fast, and repeatable. Initial setup records evidence tied to a capability-configuration and protocol-contract fingerprint. Missing or changed requirements invalidate it. Later runs compare the fingerprint cheaply and recheck volatile credentials, permissions, repository targets, and mutable GitHub identifiers before relevant mutations.
- A claim minimally records the issue, run owner, and claim time. Claims are idempotent for the same run, workers verify ownership, only the owner may release them, and abandoned claims follow an explicit recovery policy. Board status may reflect a claim but does not constitute one.
- Every worker returns a versioned receipt containing the issue, pair-specific outcome, claim disposition, and a relevant link or reason. Each worker family defines supported versions and outcomes. Coordinators may ignore additive fields and accept explicitly compatible forms, but stop on unsupported versions or unknown ownership and result semantics.
- Native GitHub dependency and sub-issue relationships are authoritative. Textual relationships are a fallback only when the active interface cannot perform native operations.
- Human decision or specification gates remain distinct from work that requires human execution.
- Findings artifacts use stable artifact and finding identities and contain evidence, dispositions, and Proposed Actions. Publication receipts record each finding ID, filing outcome, and issue URL or reason.

### Responsibility boundaries

The skill defines its workflow, safety rules, interaction points, recovery behavior, and result. Repository-owned capability configuration supplies concrete repositories, projects, labels, statuses, commands, paths, verification policy, concurrency, journeys, personas, research areas, and approval policy. Configuration cannot remove core safety behavior.

Skills request semantic capabilities. Runtime-specific notes explain only genuine Codex or Claude execution differences. Each capability is required or optional: a missing required capability stops preflight before side effects; a missing optional capability uses a documented fallback and reports the downgrade. Effective concurrency is the lower of repository policy and runtime capacity.

An agent-work worker completes when it creates a verified pull request. Monitoring is scheduled separately and included in its receipt; the work-execution contract must identify the durable lifecycle owner and claim behavior through merge, closure or rejection, failed checks, conflicts, and retries.

The findings publisher deduplicates against prior receipts and open or closed issues. It binds approval to the proposal revision and requires repository-configured approval with no catalog default. It files simple actions directly and may hand complex actions to configured ticket decomposition. When publication or decomposition cannot complete, it preserves the proposal and records a durable, retryable outcome.

## Consequences

- Issue #8 owns the concrete configuration schema, validation-receipt storage, and claim representation.
- Issues #9 and #10 own the companion-pair contracts.
- Issue #13 owns the ticket-decomposition provider decision.
- Issue #18 specifies findings publication; #11 and #12 specify its producer workflows.
- Issue #15 validates the resulting contracts and release behavior.

The original resolution was published as a [gist](https://gist.github.com/cdowell09/6cd8a4725f7f9cb24df0b0622753363b); this ADR is the in-repository copy.

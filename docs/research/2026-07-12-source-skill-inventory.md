# Source skill inventory

Date: 2026-07-12
Question: What do the seven in-scope Recruitr source skills contain, what do they assume, where do they overlap, and what must later extraction decisions account for?

## Executive answer

All seven are viable inputs to a repository-agnostic skill family, but none is ready to publish by simple copying.

- The four board skills already form two clear worker/coordinator pairs: one agent worker plus a parallel coordinator, and one human-gate worker plus a sequential coordinator.
- `dogfood` and `pipeline-research` share an evidence-to-report-to-approved-filing pipeline, but their target maps must remain project-supplied.
- `to-issues` is the common issue-slicing handoff, but it explicitly derives from Matt Pocock's upstream `to-issues`; its public disposition requires attribution and overlap resolution before implementation.
- The largest extraction seam is a shared GitHub capability contract: repository identity, labels, project fields, dependency queries, claiming, release, priority, evidence, review, and PR policies are currently embedded in prose and shell recipes.
- Runtime operations must be expressed as capabilities rather than named tools: isolated agents, interactive skill calls, worktrees, browser evidence, review, and PR monitoring.

The inventory found no embedded credentials or secrets. It did find repository coordinates, production endpoints, account names, opaque GitHub Project IDs, fixed paths, product terminology, and runtime-specific commands. These are configuration inputs or source-only evidence and must not ship as defaults.

## Scope and method

The review covered the complete `SKILL.md` for each of the seven source skills, all nine files under their `references/` directories, and the directly referenced tracker, product, architecture, ledger, and board-planning sources needed to verify their assumptions.

Local citations use paths relative to the Recruitr source repository, followed by line ranges. Absolute workstation paths, opaque IDs, and credentials are intentionally omitted from this public artifact. The excluded local skill was not inventoried; five textual dependencies on it are identified only as removal work.

## Portfolio at a glance

| Source skill | Core job | Natural public shape | Main extraction pressure |
| --- | --- | --- | --- |
| `dogfood` | Observe a deployed product as a real user, preserve evidence, and file approved findings | Product dogfooding worker | Target, auth, journey, evidence, filing, and publication configuration |
| `pipeline-research` | Research one project area through divergent lenses, distill it, and file approved actions | Project-grounded research worker | Project research map, persistent ledger, agent fan-out, scoring, and filing configuration |
| `to-issues` | Convert a plan into approved tracer-bullet issues and native dependency edges | Issue-slicing skill or upstream extension | Provenance, naming overlap, tracker adapter, and dependency representation |
| `pick-up-work` | Take one ready issue through implementation to a PR | Agent work worker | Excess policy breadth and hard-coded tracker/runtime/repository assumptions |
| `work-the-board` | Claim and dispatch a wave of ready issues | Parallel agent coordinator | Shared query/claim/result protocol and runtime-neutral fan-out |
| `pick-up-human-task` | Select and clear one high-leverage human decision gate | Human-gate worker | Human-gate vocabulary, leverage algorithm, status-source and triage adapters |
| `work-the-human-board` | Clear human gates serially in one sitting | Sequential human coordinator | Shared candidate/root/result logic, resumable skip state, and runtime-neutral skill invocation |

## Detailed inventory

### `dogfood`

The skill is a production-only observational workflow with a hard “observe, never fix” boundary. Its ten steps check deployment health, attach to a real authenticated browser, prioritize recently changed user flows, walk multiple behavioral stances, capture screenshots plus console and network evidence, optionally compare competitors, write a journey-first report, deduplicate findings, request approval, file selected issues, and publish the report through a docs-only PR. [`dogfood/SKILL.md:L8-L35`](source:.claude/skills/dogfood/SKILL.md#L8-L35), [`dogfood/SKILL.md:L37-L156`](source:.claude/skills/dogfood/SKILL.md#L37-L156), [`dogfood/SKILL.md:L158-L201`](source:.claude/skills/dogfood/SKILL.md#L158-L201)

Embedded project assumptions include the production and health endpoints, `main` as a proxy for the deployed revision, a fixed repository, real-user authentication, a fixed core journey, React/monorepo route heuristics, repository report paths, Project/Theme/milestone metadata, label routing, and an admin auto-merge convention. The supporting files add a fixed stance catalog, competitor roster, report schema, severity scale, filing rules, and web-oriented agent routing. [`dogfood/references/stances.md:L1-L64`](source:.claude/skills/dogfood/references/stances.md#L1-L64), [`dogfood/references/competitors.md:L1-L37`](source:.claude/skills/dogfood/references/competitors.md#L1-L37), [`dogfood/references/report-template.md:L1-L62`](source:.claude/skills/dogfood/references/report-template.md#L1-L62), [`dogfood/references/filing.md:L1-L104`](source:.claude/skills/dogfood/references/filing.md#L1-L104)

Required capabilities are browser attachment/control, user-mediated authentication, screenshot/console/network collection, web fetch, GitHub issue and PR operations, git, HTTP health checks, and repository writes. Improvements delegate to `to-issues`; bugs may file directly.

Portability seams:

- target endpoint, readiness check, authentication boundary, routes, core journeys, and recent-change-to-journey mapping;
- stance and competitor catalogs;
- evidence collectors, report location/schema, and severity vocabulary;
- deduplication, approval/AFK filing, labels, project metadata, and issue classifier;
- whether and how a report is committed, reviewed, merged, and retained.

Extraction risks: authenticated production actions can affect real data; a deployed commit is inferred rather than proven; “observe only” still authorizes issue, branch, PR, and merge mutations; browser commands differ by runtime; and the filing policy is duplicated between the main skill and its reference overlay.

### `pipeline-research`

This skill researches one project area per run. It grounds in product and code context, derives an avoid-list from an append-only ledger, fans out three or five isolated research agents under different lenses, applies a separate critic with a fixed weighted score, maps survivors back to project code and measurements, writes a dated brief, updates the ledger, and files only after approval. [`pipeline-research/SKILL.md:L19-L37`](source:.claude/skills/pipeline-research/SKILL.md#L19-L37), [`pipeline-research/SKILL.md:L39-L117`](source:.claude/skills/pipeline-research/SKILL.md#L39-L117), [`pipeline-research/SKILL.md:L119-L149`](source:.claude/skills/pipeline-research/SKILL.md#L119-L149)

Project assumptions include a job-pipeline taxonomy, fixed code paths and tables, product and architecture documents, an MVP priority, one LLM provider, a non-negotiable three-score model, a fixed ledger path, and the same tracker metadata as `dogfood`. The existing ledger proves the avoid-list is persistent workflow state rather than merely a proposed format. [`pipeline-research/references/pipeline-areas.md:L1-L95`](source:.claude/skills/pipeline-research/references/pipeline-areas.md#L1-L95), [`pipeline-research/references/ledger.md:L1-L53`](source:.claude/skills/pipeline-research/references/ledger.md#L1-L53), [`docs/research/LEDGER.md:L1-L27`](source:docs/research/LEDGER.md#L1-L27)

Required capabilities are primary-source web and GitHub research, isolated parallel subagents, code and documentation reading, structured report writes, persistent ledger updates, and GitHub issue/project operations. The lens contract, scoring model, report shape, and filing overlay live in four references. [`pipeline-research/references/lenses.md:L1-L74`](source:.claude/skills/pipeline-research/references/lenses.md#L1-L74), [`pipeline-research/references/report-template.md:L1-L55`](source:.claude/skills/pipeline-research/references/report-template.md#L1-L55), [`pipeline-research/references/filing.md:L1-L60`](source:.claude/skills/pipeline-research/references/filing.md#L1-L60)

Portability seams:

- project-owned research-area map, current-state sources, constraints, and code landing points;
- ledger path/schema and source/rejection identity rules;
- lens library, lens count, scoring weights/bar, and structured branch output;
- runtime-neutral isolated fan-out plus separate-critic capability;
- report persistence and the shared approval/deduplication/filing pipeline.

Extraction risks: the code map will drift; fixed five-agent fan-out may exceed runtime capacity; the main skill requests a durable brief without specifying a commit/PR step; board policies are repeated; and five references to the excluded local skill must be removed while retaining the independent algorithm: isolated divergent branches followed by a separate critic. Those references occur at `pipeline-research/SKILL.md:L34`, `L64`, `L73`, `references/lenses.md:L5`, and `references/ledger.md:L12`.

### `to-issues`

The skill gathers a plan, optionally reads the codebase and domain docs, drafts end-to-end tracer-bullet slices, separates agent-doable from human-gated work, quizzes the user until approval, creates issues from a compact template, and wires GitHub-native `blockedBy` edges. [`to-issues/SKILL.md:L12-L77`](source:.claude/skills/to-issues/SKILL.md#L12-L77), [`to-issues/SKILL.md:L79-L96`](source:.claude/skills/to-issues/SKILL.md#L79-L96)

Its assumptions are GitHub Issues and GraphQL, configured triage labels, a domain glossary and ADR layout, interactive approval, repository coordinates in the example mutation, and a downstream board planner that consumes native dependencies. GitHub documents `addBlockedBy` as the mutation that adds a blocking relationship, so this dependency choice is a supported platform capability rather than a Recruitr invention. [GitHub GraphQL issue mutations](https://docs.github.com/en/graphql/reference/mutations#addblockedby)

Portability seams are the tracker implementation, label role mapping, domain-doc discovery, dependency representation with a fallback, approval mode, and native parent/sub-issue policy. Extraction risks include body-only parent references, no duplicate search, and a GitHub-specific shell/GraphQL recipe.

Provenance is a release blocker, not a code-style note. The source explicitly calls itself a Recruitr copy that shadows Matt Pocock's upstream skill and changes dependency storage. [`to-issues/SKILL.md:L8-L10`](source:.claude/skills/to-issues/SKILL.md#L8-L10) The upstream skill contains the same purpose, workflow, and issue template, while using body-based blocker references. [Upstream `to-issues`](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-issues/SKILL.md) The upstream repository is MIT-licensed and requires preservation of its copyright and permission notice in substantial copies. [Upstream license](https://github.com/mattpocock/skills/blob/main/LICENSE)

Later design must therefore choose among contributing the native-dependency change upstream, shipping an attributed extension/fork, or avoiding a duplicate public skill by depending on the upstream one plus a GitHub adapter.

### `pick-up-work`

This is a broad single-issue implementation orchestrator: select a ready issue, read the full thread, obtain an issue-fidelity subreview, confirm interactively unless unattended, create an isolated worktree, claim the project item, implement test-first when behavioral, verify, capture UI evidence, simplify and review, open a ready PR, monitor it, and emit a machine-readable result. [`pick-up-work/SKILL.md:L8-L35`](source:.claude/skills/pick-up-work/SKILL.md#L8-L35), [`pick-up-work/SKILL.md:L47-L113`](source:.claude/skills/pick-up-work/SKILL.md#L47-L113), [`pick-up-work/SKILL.md:L115-L235`](source:.claude/skills/pick-up-work/SKILL.md#L115-L235)

It embeds repository/project identity, opaque project and status IDs, an MVP default, label names, issue-number priority, a base branch, a local board-planning script, npm-workspace commands, a specific frontend path, an authentication bypass, Playwright device and image conventions, private-repository image behavior, an evidence directory, optional FFmpeg installation, runtime-specific review commands, PR state and template rules, and indefinite monitoring. Its required skill capabilities are worktree creation, TDD, verification, simplification, issue-fidelity review, diff review, browser automation, and monitoring.

Portability seams:

- candidate query, priority, eligibility, claim/release, dependency, and result contracts;
- repository convention discovery and implementation/test policies;
- interactive versus unattended thin-spec behavior;
- worktree, browser, review, and monitoring capabilities;
- verification matrix, evidence type/location, public/private embedding, PR state/template, and completion boundary.

Extraction risks are unusually high because the skill combines selection, implementation, visual evidence, publication, and long-running ownership. Its stated MVP default does not reach the board-planning command, whose numeric milestone filter is optional; this can select all open issues. Claiming also occurs after interactive confirmation and worktree creation, leaving a double-pick window. Opaque IDs can drift, monitoring is unbounded, and the UI/GIF section overfits one private frontend repository. The planner verifies the mismatch: repository/project identity is hard-coded, milestone filtering only happens when explicitly passed, and dispatchability is otherwise label-based. [`scripts/board-plan.mjs:L36-L49`](source:scripts/board-plan.mjs#L36-L49), [`scripts/board-plan.mjs:L112-L133`](source:scripts/board-plan.mjs#L112-L133), [`scripts/board-plan.mjs:L160-L166`](source:scripts/board-plan.mjs#L160-L166)

### `work-the-board`

This coordinator builds the unblocked agent frontier, previews a wave, marks each issue in progress before dispatch, fans out at most `--max` parallel workers running unattended `pick-up-work`, collects result lines, releases failures, re-evaluates, and reports work still blocked at PR-merge boundaries. It also contains same-branch sequencing rules for post-PR fixes. [`work-the-board/SKILL.md:L8-L31`](source:.claude/skills/work-the-board/SKILL.md#L8-L31), [`work-the-board/SKILL.md:L33-L88`](source:.claude/skills/work-the-board/SKILL.md#L33-L88), [`work-the-board/SKILL.md:L90-L122`](source:.claude/skills/work-the-board/SKILL.md#L90-L122)

Its assumptions mirror `pick-up-work`: fixed project fields, runtime labels, the local board planner, issue-number priority, status as a claim lock, GitHub-native blockers that clear only when the issue closes, and children inheriting the coordinator's runtime. Required capabilities are GitHub query/mutation, bounded background fan-out, wait/collect, isolated worktrees, and result parsing.

Portability seams are the shared tracker/claim protocol, concurrency policy, fan-out adapter, merge-boundary semantics, filters, and result schema. Risks include runtime-specific “multiple Agent calls” language, inconsistent agent-label behavior between coordinator and worker, stranded claims after coordinator failure, width-only draining across human merge boundaries, and a post-PR section that may belong with monitoring rather than board dispatch.

### `pick-up-human-task`

This fully interactive worker finds the human-gated Todo issue with the greatest transitive leverage, using the latest Project status update or live board state, confirms the selection, delegates the discussion and state transition to triage/grilling skills, and normally produces `ready-for-agent`. [`pick-up-human-task/SKILL.md:L8-L44`](source:.claude/skills/pick-up-human-task/SKILL.md#L8-L44), [`pick-up-human-task/SKILL.md:L56-L127`](source:.claude/skills/pick-up-human-task/SKILL.md#L56-L127), [`pick-up-human-task/SKILL.md:L129-L169`](source:.claude/skills/pick-up-human-task/SKILL.md#L129-L169)

It assumes one owner/human, an MVP milestone, Todo status, specific human-gate labels, fixed Project status-update sections, native dependency connections, an overloaded `ready-for-human` label, path-based runtime routing, an AI disclaimer, and board status remaining Todo. Required capabilities are Project/GraphQL reads, transitive dependency reasoning, a live human, triage, grilling with domain-doc writes, issue splitting, and tracker mutation.

Portability seams are status-update versus live candidate sources, semantic human-gate roles, milestone policy, leverage scoring, triage/grilling interfaces, runtime routing, and result schema. Risks: the label overload is acknowledged design debt; leverage is a manual algorithm without a deterministic tie-breaking implementation; “today” lacks an explicit timezone; the interactive issue is not claimed; and dependent skills may mutate domain docs as well as tracker state.

### `work-the-human-board`

This is the sequential coordinator for the human worker. It previews current gates once, repeatedly re-derives the live frontier, selects the current highest-leverage root, delegates exactly one gate at a time to `pick-up-human-task`, maintains an in-memory skip set, stops on a cap or human tap-out, and hands agent-ready output to the agent board coordinator. [`work-the-human-board/SKILL.md:L8-L33`](source:.claude/skills/work-the-human-board/SKILL.md#L8-L33), [`work-the-human-board/SKILL.md:L35-L100`](source:.claude/skills/work-the-human-board/SKILL.md#L35-L100), [`work-the-human-board/SKILL.md:L102-L126`](source:.claude/skills/work-the-human-board/SKILL.md#L102-L126)

It repeats the single worker's Project, milestone, label, status, status-update, and dependency assumptions, adds a fixed board query limit, and treats one human as a hard sequential constraint. It needs serial interactive skill invocation and tracker queries, not parallel subagents.

Portability seams are a reusable sequential coordinator over configured candidate/root/result operations, milestone/source toggles, tap-out and cap behavior, and session-state handling. Risks: skip state disappears on interruption; concurrent sessions can duplicate a gate because there is no tracker claim; splitting can grow the queue; and candidate/root logic is duplicated from the worker.

## Shared primitives and duplication

The evidence supports a composable skill family with these shared contracts:

1. **Repository and tracker configuration** — repository, default branch, Project identity, field roles, option roles, labels, milestones, themes, and repository visibility.
2. **GitHub issue operations** — full-thread reads, open/closed deduplication, native dependencies, parent/sub-issue relation, board placement, metadata, comment, close, and PR-linked completion.
3. **Frontier and claim protocol** — eligibility, dependency resolution, priority, claim, release, failure recovery, and merge-boundary behavior.
4. **Worker result protocol** — structured outcomes for PR, needs-info, failure, cleared human gate, split, deferred, and none-ready.
5. **Runtime capability contract** — isolated parallel agents, sequential interactive invocation, worktrees, browser evidence, diff review, and monitoring.
6. **Evidence-to-action pipeline** — durable report/brief, deduplication, approval policy, classification, `to-issues` handoff, filing, and ledger/report persistence.
7. **Repository policy discovery** — commands, architecture docs, glossary, ADRs, verification matrix, UI surface, evidence rules, and PR policy.

The main duplication relationships are:

- `pick-up-work` is the worker for `work-the-board`; their issue query, claim, release, and result behavior should share one contract.
- `pick-up-human-task` is the worker for `work-the-human-board`; their candidate query, critical-root selection, skip/result behavior, and gate vocabulary should share one contract.
- Both board pairs traverse the same dependency graph and differ primarily in candidate roles and parallel versus live-human execution.
- `dogfood` and `pipeline-research` both create evidence, deduplicate, ask approval, classify actions, delegate improvements to `to-issues`, attach tracker metadata, and require their source artifact to remain durable.
- `to-issues` overlaps the catalog's existing `to-tickets` concept and an actively maintained upstream skill; a naming/disposition decision should precede any rewrite.

## Risks later decisions must resolve

| Risk | Consequence if ignored | Planning implication |
| --- | --- | --- |
| Opaque Project IDs and repository coordinates in prose | Immediate breakage or actions against the wrong repository | Discover IDs at runtime and map semantic roles in repository configuration |
| Labels used as both semantic roles and concrete strings | Portability failure and ambiguous human-gate behavior | Define canonical roles separately from repository mappings |
| Claims happen at different moments or are absent | Duplicate work and stranded In Progress items | Specify atomic claim/release ownership and crash recovery |
| Runtime tool and skill names are embedded | Codex/Claude divergence | Define capabilities and small runtime adapters |
| Long side-effect chains are hidden inside “research” or “observe” skills | Unexpected PRs, merges, installs, or indefinite monitoring | Make mutation/publication/monitoring policies explicit and separately confirmable |
| Product maps live inside generic workflow prose | Stale or irrelevant behavior in a new project | Make journey and research maps repository-owned inputs |
| Report persistence differs across discovery skills | Filed issues can point to unavailable evidence | Standardize durable artifact publication before filing |
| Parent and dependency semantics mix body text and native relationships | Inconsistent graph/frontier behavior | Choose native GitHub relations with an explicit capability fallback |
| Upstream-derived `to-issues` is republished without disposition | Attribution/license breach and catalog duplication | Resolve upstream contribution, attributed fork/extension, or dependency first |
| Planner and skill defaults disagree | Wrong work is selected | Put defaults in one executable/configured contract and test it |

## Evidence-led extraction order

1. Define capability configuration, semantic tracker roles, and runtime capability interfaces.
2. Extract/test the native dependency, frontier, claim/release, and result protocols from the current board planner and duplicated prose.
3. Resolve `to-issues` provenance, upstream relationship, and overlap with `to-tickets`.
4. Specify the single agent worker, then its parallel coordinator.
5. Specify the single human-gate worker, then its sequential coordinator.
6. Specify the shared evidence/approval/filing/publication layer, then apply it to dogfooding and project research with project-supplied maps.

This order is not an implementation plan by itself; it is the dependency structure the later architecture and per-skill disposition tickets should use.

## Source coverage

Reviewed in full:

- `.claude/skills/dogfood/SKILL.md` and its four reference files;
- `.claude/skills/pipeline-research/SKILL.md` and its five reference files;
- `.claude/skills/to-issues/SKILL.md`;
- `.claude/skills/pick-up-work/SKILL.md`;
- `.claude/skills/work-the-board/SKILL.md`;
- `.claude/skills/pick-up-human-task/SKILL.md`;
- `.claude/skills/work-the-human-board/SKILL.md`;
- `docs/agents/issue-tracker.md`, `docs/agents/triage-labels.md`, `scripts/board-plan.mjs`, `docs/research/LEDGER.md`, `PRODUCT.md`, `DESIGN.md`, `CLAUDE.md`, and `docs/adr/0001-mvp-agent-infrastructure.md` where referenced by the skills.

Public primary sources used for provenance/platform verification:

- [Matt Pocock's upstream `to-issues`](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-issues/SKILL.md)
- [Matt Pocock skills license](https://github.com/mattpocock/skills/blob/main/LICENSE)
- [GitHub GraphQL issue mutations](https://docs.github.com/en/graphql/reference/mutations#addblockedby)

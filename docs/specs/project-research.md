# Project research: `research-project`

Date: 2026-09-02
Question: How should the source project's `pipeline-research` become a generic, repeated research workflow with project-supplied research areas, code maps, rotating lenses, primary-source standards, an avoid-list ledger, reports, and evidence; what conforming Findings Artifact does it produce; and how does it hand off to `publish-findings` while staying successful when publication is unavailable or fails?
Resolves: [#12 Specify generic pipeline research](https://github.com/cdowell09/agent-skills/issues/12)
Inputs: [ADR 0001](../adr/0001-npx-skills-distribution-contract.md), [ADR 0002](../adr/0002-skill-boundaries-and-shared-contracts.md), [ADR 0003](../adr/0003-ticket-decomposition-provider.md), [ADR 0004](../adr/0004-licensing-attribution-and-contribution.md), [Capability configuration contract](capability-configuration.md), [Findings publication contract](findings-publication.md), [Source skill inventory](../research/2026-07-12-source-skill-inventory.md), [Source provenance audit](../research/2026-07-12-source-provenance.md)

Skill names used here (`pick-up-work`, `work-the-board`, `pick-up-human-task`, `work-the-human-board`, `dogfood`, `research-project`, `publish-findings`) are working names; a later naming pass may rename them. The other findings producer, `dogfood`, is specified concurrently in `dogfooding.md`; the two documents keep their producer-side structure parallel (artifact production, handoff line, result reporting) without depending on each other's text.

**Clean-room statement.** Per ADR 0004 section 3, the research mechanism in this document is written from a requirements list only: configurable research areas, a source ledger and avoid-list, isolated divergent research branches, a separate critique pass, evidence and applicability scoring, and explicit mapping to the consuming codebase. The source project's fan-out, isolation, critic, and lens-rotation passages, its `lenses.md` and `ledger.md`, and the excluded local skill were not opened or paraphrased. `docs/provenance.md` records this document as the drafting input for the public `SKILL.md`.

## Decision

- `research-project` researches **one configured area per run**, grounded in the repository's own documents and code, and produces exactly one **Findings Artifact** (`findings` v1). It changes no code and files nothing itself; it hands the artifact to `publish-findings` when that sibling is installed, else prints the install command and stops. Publication is reported on a separate line and never changes the research result.
- The **research map** is repository-owned: `research.areas_file` (addition for #8, default `docs/research/areas.yaml`) holds per-area questions, current-state sources, code landing points, constraints, and priority; the capability spec's inline `research.areas` stays valid as the minimal form, and the two are mutually exclusive. Selection is deterministic: argument, else least-recently-researched (from the ledger), then priority, then id.
- A **lens** is a research stance. Six neutral defaults ship in `references/lenses.md`; configuration adds or disables lenses. Each run uses `research.lens_count` lenses (default 3), those least recently used for the area; the ledger is the rotation state.
- The **ledger** at `research.ledger_path` (default `docs/research/LEDGER.md`, committed) is append-only Markdown: `## Sources` (skill-owned, one line per source, verdict `adopted | rejected | deferred`) and `## Publications` (publisher-owned). The avoid-list is derived from it: sources adopted or rejected for the same area within `research.avoid_list_expiry` (default 180 days).
- The **mechanism**: grounding pass; N isolated branches, one per lens, each seeing only the area brief, its lens, the avoid-list, and the source standards; fetch verification of every citation; a separate critique pass scoring evidence, applicability, and feasibility with configurable weights and bar (defaults 0.4/0.4/0.2, bar 3.5, evidence floor 2); mapping of survivors to landing points and measurable expectations; artifact; ledger; handoff. Isolated subagents are optional with two disclosed fallbacks.
- **Persistence**: artifact, ledger, and publication receipt are committed on a branch and opened as a docs PR (`research.persist: branch-pr`, never merged by the skill). **`receipt.research` v1** uses the catalog's receipt envelope with outcomes `researched | researched-empty | setup-required | failed`.

## Purpose and boundary

| | `research-project` |
| --- | --- |
| Owns | Area selection, grounding in the repository's docs and code, lens selection, bounded divergent research, citation verification, critique and scoring, mapping to landing points, the Findings Artifact, ledger updates, the docs branch/PR, the receipt, the publisher handoff |
| Never | Edits source code or the research map; files issues; merges anything; reads a sibling skill beyond its frontmatter; runs unattended setup; writes a candidate that failed the bar into the artifact; cites a URL it did not fetch |
| Done when | The artifact is written and validated, the ledger is appended, persistence has run, and the receipt is emitted; publication is reported separately |

## Preserve, remove, configure, delegate

Grounded in the inventory's `pipeline-research` entry and its portability seams.

| Disposition | Item |
| --- | --- |
| Preserve (as ideas, re-expressed) | One area per run; an area map; grounding in current-state documents and code before researching; an append-only ledger that yields an avoid-list; isolated divergent research branches; a critique pass separate from the branches; weighted scoring against a bar; mapping survivors to concrete code landing points with measurable expectations; a durable dated brief; filing only after approval |
| Remove | The job-pipeline taxonomy; fixed code paths and tables; product and architecture documents as inputs; the MVP priority; the LLM-provider note; the fixed three-score product model; the fixed ledger path; tracker metadata (Project, Theme, milestone, labels); the fixed five-agent fan-out; references to the excluded skill (mechanism replaced, not renamed) |
| Configure | Research map (`research.areas_file` or inline `areas`); per-area code map and grounding docs; repository-wide grounding docs; ledger path; branch count (`lens_count`, capped by budget and runtime capacity); lens library additions and disablements; scoring weights, bar, and evidence floor; avoid-list expiry; budget (branches, sources per branch, minutes); persistence mode and branch pattern; approval mode (`findings.approval.mode`, required, no default) |
| Delegate | Filing, deduplication against issues, approval binding, receipts → `publish-findings` (#18). Decomposition of complex actions → the `ticket-decomposition` capability through the publisher (ADR 0003). Setup and validation tiers → the capability spec's generated `references/setup.md` |

## Research map (`research_map` v1)

A YAML file at `research.areas_file`, repository-owned and edited only by humans. The skill reads it and never writes it.

```yaml
research_map: 1
grounding:                          # optional repository-wide current-state sources read every run
  docs: [CONTEXT.md, docs/adr/**]
areas:
  - id: ingestion                   # kebab-case, stable; becomes subject `area:ingestion`
    name: Upload ingestion
    priority: 1                     # 1 = highest; ties broken by id
    questions:
      - How do comparable systems make uploads safe to retry?
      - What validation must happen before a file is accepted into the queue?
    current_state:
      docs: [docs/adr/0003-ingestion.md, docs/ingestion.md]
      notes: "Single POST /uploads endpoint; 25 MB limit; synchronous scan before enqueue."
    landing_points:
      - { path: "apps/api/src/uploads/**", role: "HTTP handlers and request validation" }
      - { path: "packages/ingest/**", role: "queue producer and job schema" }
    constraints:
      - "No new infrastructure services this quarter"
      - "Public API stays backward compatible"
    last_researched: 2026-06-14     # optional seed; ledger entries win once present
  - id: search
    name: Full-text search
    priority: 2
    questions: ["What ranking and indexing approaches fit a single-node Postgres deployment?"]
    current_state: { docs: [docs/search.md] }
    landing_points: [{ path: "packages/search/**", role: "query builder and index maintenance" }]
    constraints: ["Postgres only; no separate search service"]
  - id: background-jobs
    name: Background job execution
    priority: 2
    questions: ["How should retries, backoff, and dead-lettering work for long-running jobs?"]
    current_state: { docs: [docs/adr/0005-jobs.md] }
    landing_points: [{ path: "packages/jobs/**", role: "worker loop and retry policy" }]
    constraints: []
```

Rules: `id`, `name`, and at least one `question` and one `landing_point` are required per area; `priority` defaults to 5; paths are repository-relative globs that may not escape the repository. A landing point that matches no file is a warning listed in the artifact Summary (the map drifts; the skill reports, never repairs). The inline `research.areas` form from the capability spec (`id`, `summary`, `code`, `docs`) maps to `name`/`questions: [summary]`/`landing_points`/`current_state.docs`; declaring both forms is a validation error.

**Area selection.** `--area <id>` selects explicitly (unknown id stops with the list of ids). Otherwise: order areas by last-researched date ascending, where the date is the newest `## Sources` ledger line for that area or, if none, `last_researched` from the map or, if absent, "never" (sorted first); then by `priority` ascending; then by `id`. The selection and its reason are printed and recorded in the receipt, so two machines with the same ledger choose the same area.

## Lens library

A lens is a research stance: which kinds of primary sources to seek and what question to put to them. Default set, shipped fresh in `references/lenses.md` (maintainer ratifies):

| id | Stance | Primary sources sought |
| --- | --- | --- |
| `standards` | What do specifications and protocol documents require or recommend? | RFCs, W3C/IETF/ISO documents, official protocol specs |
| `literature` | What has been measured or proven? | Peer-reviewed papers, theses, reproducible benchmarks |
| `open-source` | How do maintained projects with the same problem implement it? | Source code at a commit, design docs, changelogs of OSS projects |
| `competitors` | What do comparable products expose to their users? | First-party product docs, API references, release notes |
| `postmortems` | How has this failed for others? | Incident reports, CVE records, first-party postmortems, issue threads with maintainer responses |
| `cost-performance` | What does it cost and how does it scale? | Vendor pricing pages, published capacity and latency data, first-party benchmarks |

A lens entry has `id`, `stance`, `sources` (what counts as primary for it), a `question` template combining the area's questions with the stance, and `avoid` (shortcuts it must not take). `research.lenses.add` adds entries with the same fields; `research.lenses.disable` removes defaults.

**Rotation.** For the selected area, each enabled lens's last-used date is the newest ledger line with that area and lens; never-used lenses sort first, then oldest last-used, then library order. Take `research.lens_count` (default 3); `--lens <id>` (repeatable) overrides for one run. One branch runs per lens, so the effective branch count is `min(lens_count, research.budget.max_branches, runtime subagent capacity)`. A failed branch is not ledgered, so its lens is first in line next run.

## Primary-source standards (`references/source-standards.md`)

- **Primary**: official documentation; standards and specifications; peer-reviewed papers or their preprints; source code at a named commit; first-party announcements, changelogs, pricing pages, and postmortems; issue-tracker threads where a maintainer states a decision.
- **Secondary** (allowed only as a pointer to a primary source, never as the sole evidence): blog summaries, tutorials, aggregator posts, forum answers, model-generated text, and any page that restates another source without adding measurement or authority.
- **Disallowed**: uncited claims; claims whose only support is secondary; citations the skill did not fetch in this run; paywalled or login-gated content the skill could not read (record it as `deferred` with the reason instead).
- **Citation format** in the artifact prose: `<url> (accessed YYYY-MM-DD): <one-line relevance>`. The artifact's `evidence:` list carries the bare URLs, as the findings contract requires; accessed dates and relevance live in the finding prose and the ledger.
- **Source identity** for the ledger: a URL with fragment and query removed and host lowercased; `doi:<doi>` for papers; `<repository url>@<commit>` for code. Identity decides avoid-list matches, so two citations of the same page with different fragments are one source.
- **Verification rule**: before the artifact is written, every URL cited by a surviving candidate is fetched again in the main context and checked for the claimed content. A URL that fails to resolve, or whose content does not support the claim, is dropped; a candidate that thereby loses its last primary source falls below the evidence floor and goes to the ledger as `rejected` with reason `citation-unverified`. Nothing unverified reaches the artifact.

## Ledger (`ledger` v1, `research.ledger_path`)

Committed Markdown, append-only. The skill owns `## Sources`; `publish-findings` owns `## Publications` (created at the end of the file if absent, one line per terminal receipt row, as its contract states). No other section is ever edited or reordered.

```markdown
---
ledger_version: 1
---
# Research ledger

## Sources
- 2026-06-14 | ingestion | standards | adopted | https://www.rfc-editor.org/rfc/rfc9110 | F-71c0d2aa-2 | idempotent method semantics for retried requests
- 2026-09-02 | ingestion | postmortems | rejected | https://example.com/blog/why-our-uploads-broke | - | secondary summary; no first-party incident report found
- 2026-09-02 | ingestion | open-source | deferred | https://github.com/acme/uploader@4f1c2a9 | - | chunked-upload design needs a benchmark against the 25 MB limit before scoring
- 2026-09-02 | ingestion | standards | adopted | https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload | F-a3e489ef-2 | content-based file type validation

## Publications
- 2026-09-02 research-project-20260902-ingestion A-a3e489ef-1 filed https://github.com/acme/widgets/issues/312
```

Line format, seven pipe-separated fields in fixed order: `date | area | lens | verdict | source identity | finding ID or "-" | reason`. Verdicts: `adopted` (cited by a finding that passed the bar; the finding ID is required), `rejected` (evaluated and dismissed; reason required), `deferred` (relevant but not evaluable this run: paywalled, needs an experiment, out of budget). One line per source per run; a source cited by two findings gets two lines. Lines are written after critique and after the artifact is validated, in one append, so a partial write cannot leave the section inconsistent with the artifact.

**Avoid-list derivation.** For the selected area: every source with verdict `adopted` or `rejected` whose date is within `research.avoid_list_expiry` (default `180d`). Branches receive the list with each source's verdict and reason and must not return those sources; if one does, critique drops it with reason `on-avoid-list` and writes no new line. `deferred` sources are not avoided, so they can be revisited. Sources ledgered under a different area are not avoided: applicability differs per area. After expiry a source may be researched again; a re-evaluation writes a new line rather than editing the old one.

**Recovery.** If a run wrote the artifact but crashed before the ledger append (the receipt says `ledger: not-updated`), the next run for that area reads the artifact's evidence URLs and `## Findings` before grounding and appends the missing `adopted` lines first, dated with the artifact's `created_at`.

## Mechanism (written from requirements)

Budget: `research.budget.max_branches` (default 5), `max_sources_per_branch` (default 8), `max_minutes` (default 30, wall clock for the branch phase). Exceeding the time cap stops unfinished branches; their partial candidates are kept, the receipt records `budget.exhausted: true`, and the artifact Summary says which lenses were cut.

1. **Grounding pass** (main context). Load the area from the map; read `grounding.docs` and the area's `current_state.docs`; read the files under each landing point (headers, public interfaces, tests; not every line); derive the avoid-list. Write an **area brief** to the scratch directory: area name and questions, a current-state summary in the skill's own words with file references, constraints, landing points with the observed role of each, drift warnings, and the avoid-list. The brief is the only project context a branch sees.
2. **Isolated research branches**, one per selected lens. Each receives the area brief, its lens entry, the source standards, and the per-branch budget, and nothing else: not the other branches, not prior artifacts, not the full ledger. Each returns a structured candidate list, each candidate carrying `title`, `claim` (one paragraph), `sources` (identity, URL, accessed date, one-line relevance, primary yes/no), `applicability` (why this fits the area's current state and constraints), `suggested_landing` (landing point ids), and `suggested_expectation` (what would measurably change). Capability: **isolated subagents** (optional). Fallback 1: sequential branches, each in a fresh context where the runtime can start one, with the brief re-supplied; disclosed as `branches.mode: sequential`. Fallback 2: a single branch in the main context with one lens per run, rotating across runs; disclosed as `branches.mode: single-lens` in the receipt and the artifact Summary, and `lens_count` is reported as effectively 1.
3. **Citation verification** (main context) per the source standards. Every URL cited by every candidate is fetched; unsupported citations are dropped before critique so the critique never scores on a citation that does not exist.
4. **Critique pass**, separate from every branch: an isolated subagent when available, else a fresh context, else a procedural separation in the main context (candidates are written to a file first; the critique reads only that file, the brief, and the rubric; disclosed as `critique.mode: in-context`). It first merges candidates that make the same claim (union of sources, the stronger applicability note), then scores each merged candidate on three axes, 0–5 each, from `references/scoring-rubric.md`:
   - **evidence**: 0 no primary source; 2 one primary source supporting part of the claim; 4 two independent primary sources; 5 a standard or measurement that settles it;
   - **applicability**: 0 contradicts a constraint; 2 fits with substantial adaptation; 4 fits the current state directly; 5 fits and addresses a stated question;
   - **feasibility** (effort and risk, higher is easier): 0 needs new infrastructure or a breaking change; 2 multi-week, cross-module; 4 contained in one landing point; 5 small, reversible, testable.
   Score = `weights.evidence × evidence + weights.applicability × applicability + weights.feasibility × feasibility` with weights summing to 1 (defaults 0.4, 0.4, 0.2). A candidate **passes** when score ≥ `research.scoring.bar` (default 3.5) and evidence ≥ `research.scoring.evidence_floor` (default 2). Verdicts: `adopt`, `reject` (reason), `defer` (reason: needs an experiment, paywalled, budget). The critique also assigns each adopted candidate a **severity** meaning expected impact (`high` when it resolves a stated question or a known failure mode, `medium` by default, `low` for polish) and a disposition (`improvement`, or `question` when the right answer needs a human decision).
5. **Mapping.** For each adopted candidate: confirm the landing points against the grounding pass (paths that exist and match the role), name the concrete files or modules, and write one **measurable expectation** (a metric, a test that would pass, or a behaviour a reviewer can check). A candidate with no confirmable landing point becomes a `question` finding rather than an `improvement`.
6. **Artifact** written and validated with the publisher's rules (next section). **Deduplication** against prior artifacts and receipts runs before actions are drafted (below).
7. **Ledger** appended: `adopted` lines for every source cited by a finding, `rejected` and `deferred` lines for the rest, and lines for dropped citations with reason `citation-unverified`.
8. **Persist, hand off, report** per the sections below.

Stop points in interactive runs: after area selection (confirm or choose another), and before opening the docs PR. Unattended runs stop at neither and never enter setup.

## The artifact (`findings` v1)

Disposition mapping for research: adopted candidates with a confirmed landing point are `improvement`; candidates that need a human choice before they can land are `question`; `bug` is used only when a source proves the current state violates a standard the repository claims to follow; `note` and `wontfix` are not used (rejected candidates go to the ledger, not the artifact). Per the publisher's open item 5, confidence lives in the extension field `score`, never in `severity`, so `auto_file_severities` keeps meaning impact.

Actions: one `simple` `agent-doable` action per improvement whose expectation fits in one issue; one `complex` `human-decision` action when the improvement needs slicing or the finding is a `question` that gates work. Findings that only inform stay actionless. `labels` carry roles only (`improvement`, `finding`); disposition roles are the publisher's.

Complete example. `artifact_short_id` and every fingerprint below were computed with the publisher's formulas (`sha256("research-project-20260902-ingestion")` → `a3e489ef`; punctuation removed without substitution during normalization). The cited URLs are well-known public documents; they could not be fetch-verified from the drafting session because its egress proxy blocked those hosts, so the pilot (#16) re-verifies them under the verification rule before this example is used as a fixture.

````markdown
---
artifact_version: 1
artifact_id: research-project-20260902-ingestion
artifact_short_id: a3e489ef
producer: { skill: research-project, version: "0.1.0" }
created_at: 2026-09-02T15:42:10Z
subject: area:ingestion
revision: 1
ledger: docs/research/LEDGER.md
---

## Summary

Researched **Upload ingestion** (area `ingestion`, selected by rotation: last researched 2026-06-14, priority 1) through lenses `standards`, `postmortems`, `open-source` in three isolated branches. Grounding: `docs/adr/0003-ingestion.md`, `docs/ingestion.md`, `apps/api/src/uploads/**`, `packages/ingest/**`. Avoid-list: 4 sources. Branches returned 11 candidates, merged to 9; 7 citations verified, 0 dropped. Critique adopted 3, rejected 5, deferred 1 (see the ledger). Weights 0.4/0.4/0.2, bar 3.5. Previously found and already filed: none. Drift: none; all landing points resolved.

## Findings

### F-a3e489ef-1: Upload retries are not idempotent, so a client retry after a timeout creates duplicate ingest jobs
- severity: high
- disposition: improvement
- fingerprint: sha256:9c4a60b75a667c68
- evidence:
  - https://www.rfc-editor.org/rfc/rfc9110
  - https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/
- lens: standards
- score: 4.2 (evidence 4, applicability 5, feasibility 3)
- landing: apps/api/src/uploads/handler.ts, packages/ingest/src/enqueue.ts
- expectation: two POST /uploads requests with the same Idempotency-Key produce one job; covered by an integration test

`POST /uploads` is non-idempotent and the client retries on timeout (`apps/api/src/uploads/handler.ts`, `docs/ingestion.md` section "Retries"), so a slow scan yields duplicate jobs in `packages/ingest`. Sources: https://www.rfc-editor.org/rfc/rfc9110 (accessed 2026-09-02): defines idempotent methods and why POST is not one. https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/ (accessed 2026-09-02): specifies a request header and server behaviour for making POST safe to retry.

### F-a3e489ef-2: File type is trusted from the client Content-Type header instead of the file bytes
- severity: medium
- disposition: improvement
- fingerprint: sha256:6b2b360ba48b67b1
- evidence:
  - https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload
- lens: postmortems
- score: 3.8 (evidence 3, applicability 4, feasibility 5)
- landing: apps/api/src/uploads/validate.ts
- expectation: a renamed executable with an image Content-Type is rejected before enqueue; unit tests for the accepted types

Validation compares `Content-Type` with an allow-list (`apps/api/src/uploads/validate.ts`) and never inspects the bytes. Sources: https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload (accessed 2026-09-02): recommends content-based validation and treating client-supplied type and extension as untrusted.

### F-a3e489ef-3: Should ingestion support resumable uploads for files above the current size limit
- severity: medium
- disposition: question
- fingerprint: sha256:db82359b4b45fc7c
- evidence:
  - https://tus.io/protocols/resumable-upload
- lens: open-source
- score: 3.5 (evidence 3, applicability 3, feasibility 3)
- landing: apps/api/src/uploads/**, packages/ingest/src/schema.ts
- expectation: a decision recorded in an ADR; if adopted, uploads above 25 MB complete over interrupted connections

The 25 MB limit exists because a single request must finish before the synchronous scan (`docs/adr/0003-ingestion.md`). A resumable protocol would lift the limit but touches the public API, which the area's constraints protect. Sources: https://tus.io/protocols/resumable-upload (accessed 2026-09-02): an open protocol for resumable uploads with chunked PATCH semantics.

## Proposed Actions

### A-a3e489ef-1: Accept an Idempotency-Key on POST /uploads and deduplicate enqueue by key
- findings: [F-a3e489ef-1]
- kind: simple
- disposition: agent-doable
- labels: [improvement, finding]
- status: proposed

Read the header in `apps/api/src/uploads/handler.ts`; store key → job id with the existing scan-status record; on a repeated key return the stored job instead of enqueueing in `packages/ingest/src/enqueue.ts`. Keys expire with the scan record. Add an integration test that retries the same request twice and asserts one job. Expectation: duplicate jobs from client retries drop to zero in the ingest queue metrics.

### A-a3e489ef-2: Decide whether ingestion adopts resumable uploads, then slice the work
- findings: [F-a3e489ef-3, F-a3e489ef-2]
- kind: complex
- disposition: human-decision
- labels: [improvement, finding]
- status: proposed

Decision needed: keep the 25 MB limit, or adopt a resumable protocol despite the "public API stays backward compatible" constraint (additive endpoints may satisfy it). If adopted, slices span request handling, content validation at chunk assembly (F-a3e489ef-2 applies at that point), the ingest job schema, and client changes. Expectation: an ADR under `docs/adr/` records the decision and, if adopted, the first slice lands content-based validation for the assembled file.

## Publication
````

## Deduplication before Proposed Actions

After critique and before actions are drafted, each adopted candidate's fingerprint (computed exactly as the publisher does, from title, `area:<id>`, and disposition) is compared with every finding in artifacts under `findings.artifact_dir` whose `subject` matches, and with every `fingerprints` value in `.agent-skills/state/publications/*.yaml`. A hit whose newest receipt row is `filed`, `handed-off`, `duplicate`, or `rejected` means the candidate is **not emitted** as a new finding: the Summary lists it under "Previously found and already filed" with the prior finding ID and issue URL, and the ledger records its sources as `adopted` referencing the prior finding ID. A hit with no terminal row (an earlier artifact that was never published) is emitted again with a Summary note naming the earlier artifact, so the publisher's own deduplication decides. Producer-side deduplication only avoids re-proposing; `publish-findings` stays authoritative and runs its three layers regardless.

## Persistence

`research.persist` (addition for #8): `branch-pr` (default), `commit`, or `working-tree`. With `branch-pr`, after the handoff: require a clean working tree (else downgrade to `working-tree`, disclosed); create `research.branch_pattern` (default `research/{area}-{date}`) from `work.base_branch` or the discovered default branch; commit the artifact, the ledger, and, when the publisher ran, its receipt; push; `gh pr create` with the artifact Summary as body. One PR per run, never merged by the skill. `commit` stops after the commit; `working-tree` prints the paths to commit. Interactive runs confirm before the push.

## Handoff and result

Handoff follows the publisher's producer contract exactly: the artifact is written and validated with the same rules; `publish-findings` is located by the capability spec's sibling-skill discovery order, reading only its frontmatter; when found and the runtime supports sequential interactive skill invocation, it is invoked with the artifact path plus `--unattended` when this run is unattended, and its result line is awaited; the report ends with the separate `publication:` line in the publisher's vocabulary (`not-attempted (not-installed)` with the install command, `not-attempted (invocation-unavailable)` with the command for the user, `succeeded (...)`, `succeeded-with-failures (...)`, `failed (<outcome>: <reason>)`). One additive value is proposed for #18: `not-attempted (no-actions)` for a `researched-empty` artifact, so no empty receipt is created. Without the publisher the skill persists the artifact, prints the install command, and stops; it embeds no second filing path (accepting #18's recommendation). The research result is emitted before the handoff and is never changed by it.

````markdown
<!-- agent-skills:receipt research v1 -->
```yaml
receipt: 1
family: research
skill: research-project
skill_version: 0.1.0
owner: claude-code@buildbox/20260902T154210Z-8c21d0f4
area: ingestion
outcome: researched              # researched | researched-empty | setup-required | failed
selection: { by: rotation, last_researched: 2026-06-14, priority: 1 }
artifact: { id: research-project-20260902-ingestion, path: docs/findings/research-project-20260902-ingestion.md, revision: 1, findings: 3, actions: 2 }
lenses: [standards, postmortems, open-source]
branches: { mode: isolated-subagents, planned: 3, completed: 3, failed: [] }
critique: { mode: isolated-subagent, weights: { evidence: 0.4, applicability: 0.4, feasibility: 0.2 }, bar: 3.5 }
candidates: { received: 11, merged: 9, adopted: 3, rejected: 5, deferred: 1, previously_filed: 0 }
citations: { cited: 7, verified: 7, dropped: 0 }
ledger: { path: docs/research/LEDGER.md, appended: 9, status: updated }   # updated | not-updated
budget: { branches: "3/5", sources: "19/24", minutes: "14/30", exhausted: false }
persistence: { mode: branch-pr, branch: research/ingestion-20260902, pr: https://github.com/acme/widgets/pull/318 }
downgrades: []
reason: null
started_at: 2026-09-02T15:20:04Z
finished_at: 2026-09-02T15:49:31Z
protocols: { config: 1, findings: 1, receipt.research: 1 }
```
publication: succeeded (.agent-skills/state/publications/research-project-20260902-ingestion.yaml; 1 filed, 1 handed-off, 0 duplicate, 0 deferred)
````

Required fields: `receipt`, `family`, `skill`, `owner`, `area`, `outcome`, `protocols`; `artifact` for `researched` and `researched-empty`; `reason` for `failed`. The receipt is also written as JSON to `state/local/runs/<owner-id>.json`. There is no `claim` field: this family mutates no tracker item. Consumers ignore additive fields and stop on an unknown major or outcome. `researched-empty` is a success: the artifact exists with zero findings and the ledger records why every candidate was rejected. `failed` is reserved for errors before an artifact could be written; a budget cut, a failed branch, a dropped citation, or a failed publication never makes the run `failed`.

## Capabilities

| Capability | Required | Fallback |
| --- | --- | --- |
| Web fetch and search | yes | none; preflight stops |
| `git` and repository file writes (artifact, ledger, scratch brief) | yes | none |
| `gh` for the docs PR | required for `branch-pr`; otherwise optional | `persist` downgrades to `commit`, disclosed |
| Isolated subagents (branches and critique) | optional | sequential fresh-context branches; else single-lens branch with in-context critique; both disclosed |
| GitHub read via `gh api` for `open-source` sources at a commit | optional | plain web fetch of the file view |
| `publish-findings` sibling | optional | artifact produced; `publication: not-attempted (not-installed)` |
| Sequential interactive skill invocation | optional | `publication: not-attempted (invocation-unavailable)` with the command |
| `ticket-decomposition` | optional, via the publisher | complex actions deferred by the publisher (ADR 0003) |

Runtime notes, only where the runtimes differ: Claude Code runs branches as background subagents with their own context, so `branches.mode: isolated-subagents` and an isolated critique are the norm; Codex runs parallel work through its task facility under the cap in `capabilities.runtime_notes.codex`, and where that facility is unavailable the sequential fresh-context fallback applies. Web fetch and `gh` behave identically.

## Configuration keys consumed

```yaml
version: 1
research:                                   # required section for research-project
  areas_file: docs/research/areas.yaml      # ADDITION (#8): research map; mutually exclusive with inline `areas`
  # areas: [{ id, summary, code, docs }]    # capability spec's minimal inline form; one of the two is required
  ledger_path: docs/research/LEDGER.md      # capability spec key (the ticket's `ledger_file`)
  lens_count: 3                             # capability spec key; effective = min(this, budget.max_branches, runtime capacity)
  lenses:                                   # ADDITION
    add: []                                 # [{ id, stance, sources, question, avoid }]
    disable: []
  scoring:                                  # ADDITION
    weights: { evidence: 0.4, applicability: 0.4, feasibility: 0.2 }
    bar: 3.5
    evidence_floor: 2
  avoid_list_expiry: 180d                   # ADDITION
  budget: { max_branches: 5, max_sources_per_branch: 8, max_minutes: 30 }   # ADDITION
  persist: branch-pr                        # ADDITION: branch-pr | commit | working-tree
  branch_pattern: "research/{area}-{date}"  # ADDITION
findings:
  artifact_dir: docs/findings
  approval: { mode: interactive }           # REQUIRED, no default (capability spec)
work: { base_branch: main }                 # optional; default the discovered default branch
capabilities: { subagents: auto, interactive_skill_invocation: auto }
```

Additions flagged for #8: `research.areas_file`, `research.lenses.{add,disable}`, `research.scoring.{weights,bar,evidence_floor}`, `research.avoid_list_expiry`, `research.budget.*`, `research.persist`, `research.branch_pattern`. None is mutation-affecting in the spec's sense except `persist`, which guided setup should ask because it creates branches and PRs.

## Rough `SKILL.md` outline

```yaml
---
name: research-project
description: Research one configured area of this repository through rotating lenses in isolated branches, verify every citation, score candidates in a separate critique pass, and write a findings artifact mapped to code landing points, handing it to publish-findings when installed.
license: MIT
compatibility: Git repository; web fetch and search; gh CLI for the docs PR; .agent-skills/config.yaml with research.* and findings.approval.mode. Optional: isolated subagents (else sequential or single-lens fallback); publish-findings installed for filing.
metadata: { author: Christian Dowell, version: "0.1.0", provenance: original, contracts: "config=1 findings=1 receipt.research=1" }
---
```

Headings, one line per step, stop points marked:

1. **Purpose and boundary** — one area per run; artifact out; no code changes; filing only through `publish-findings`.
2. **Arguments** — `--area <id>`, `--lens <id>` (repeatable), `--unattended`, `--dry-run` (stops after the grounding brief).
3. **Preflight** — config tiers per `references/setup.md`; required capabilities; sibling discovery of the publisher; list downgrades; stop with guidance on missing keys or an unreadable map. **Stop point: `setup-required`.**
4. **Select the area** — argument or rotation per `references/research-map.md`; print the reason. **Stop point (interactive): confirm or choose another.**
5. **Ground** — read grounding docs, area docs, landing points; derive the avoid-list per `references/ledger.md`; write the area brief; report drift.
6. **Select lenses** — rotation per `references/lenses.md`; effective count and mode.
7. **Run research branches** — one per lens with brief, lens, standards, budget only; collect structured candidates; record failed or cut branches.
8. **Verify citations** — fetch every URL; drop unsupported ones per `references/source-standards.md`.
9. **Critique** — separate pass: merge, score, verdict, severity, disposition per `references/scoring-rubric.md`.
10. **Map** — landing points and measurable expectations; unmappable candidates become questions.
11. **Deduplicate** — prior artifacts and receipts; previously filed candidates go to the Summary and ledger only.
12. **Write and validate the artifact** — `references/artifact-template.md`, validated against `references/findings-artifact.md`.
13. **Append the ledger** — one append after validation.
14. **Hand off** — discovery, invocation, and the separate `publication:` line per `references/findings-artifact.md`.
15. **Persist** — branch, commit, PR per `research.persist`. **Stop point (interactive): confirm before push.**
16. **Report** — receipt block and run file per `references/receipt-research.md`.
17. **Safety rules** — never edit code or the map; never merge; never cite an unfetched URL; never put a failed candidate in the artifact; never enter setup unattended; never file directly.

`references/`:

- `research-map.md` — `research_map` v1 schema, inline-form mapping, selection rule, example.
- `lenses.md` — the six default lenses, entry fields, rotation rule, configuration hooks.
- `source-standards.md` — primary/secondary/disallowed, citation format, identity normalization, verification rule.
- `ledger.md` — `ledger` v1 format, verdicts, avoid-list derivation and expiry, recovery, the reserved `## Publications` heading.
- `scoring-rubric.md` — axes, anchors, weights, bar, floor, merge rule, severity and disposition assignment, critique isolation modes.
- `artifact-template.md` — the research-specific artifact skeleton and extension fields (`lens`, `score`, `landing`, `expectation`).
- `receipt-research.md` — `receipt.research` v1 fields and outcomes.
- `findings-artifact.md` — generated copy of the `findings` v1 contract, validation codes, and producer handoff contract; drift-tested.
- `setup.md` — generated setup and tiered-validation procedure from the capability spec.
- `protocol-contracts.md` — generated; majors produced (`receipt.research` 1, `findings` 1) and accepted (`config` 1).
- `runtime-adapters.md` — subagent and fresh-context mechanics per runtime; nothing else differs.

No `NOTICE.md`; the skill copies no third-party text. No external skills beyond the optional `publish-findings` sibling, which is a catalog skill.

## Acceptance scenarios

1. **No areas.** Given `config.yaml` has neither `research.areas_file` nor `research.areas`, when the skill runs, then it stops with the setup block naming the key, writes nothing, and records `setup-required`.
2. **Area rotation.** Given three areas where the ledger's newest lines are `search` 2026-08-20, `ingestion` 2026-06-14, and `background-jobs` has none, when no `--area` is passed, then `background-jobs` is selected with reason "never researched" and the receipt records the selection.
3. **Avoid-list respected.** Given the ledger rejected a source for `ingestion` 90 days ago, when a branch returns it anyway, then critique drops it with `on-avoid-list`, no new ledger line is written, and the artifact does not cite it.
4. **Avoid-list expiry.** Given the same source rejected 200 days ago under `avoid_list_expiry: 180d`, when a branch returns it, then it is scored fresh and a new ledger line records the new verdict; the old line is untouched.
5. **Subagents absent.** Given `capabilities.subagents: unavailable` and a runtime that can start a fresh context, when the run executes, then branches run sequentially, the receipt says `branches.mode: sequential`, and the artifact Summary discloses it.
6. **Single-lens fallback.** Given neither subagents nor fresh contexts, when the run executes, then one lens is used, the receipt says `branches.mode: single-lens` and `critique.mode: in-context`, and the next run rotates to the next lens.
7. **Critique rejects all.** Given every candidate scores below the bar, when the run completes, then the artifact exists with empty `## Findings` and `## Proposed Actions`, the ledger has a `rejected` line per source, the outcome is `researched-empty`, and the last line is `publication: not-attempted (no-actions)`.
8. **Branch failure.** Given three planned branches and the `literature` branch times out, when the run continues, then the artifact is built from the other two, the receipt lists `branches.failed: [literature]`, no ledger line names `literature`, and the next run for the area selects `literature` first.
9. **Ledger append after crash.** Given a run that wrote and validated the artifact but crashed before the ledger append, when the next run for the area starts, then it appends the missing `adopted` lines from the artifact before grounding and reports the recovery.
10. **Publisher absent.** Given `publish-findings` is not discoverable, when the run completes, then the outcome is `researched`, the docs PR is opened, and the last line is `publication: not-attempted (not-installed)` with the install command.
11. **Publisher fails.** Given the publisher returns `aborted` after a `gh` authentication error, when the run reports, then the outcome is still `researched`, the artifact and ledger are persisted unchanged, and the last line is `publication: failed (aborted: ...)` with the standalone command to retry.
12. **Duplicate across runs.** Given a prior artifact for `area:ingestion` whose finding fingerprint `sha256:9c4a60b75a667c68` has a `filed` receipt row, when critique adopts a candidate with that fingerprint, then it is not emitted as a finding, the Summary lists it with the issue URL, and the ledger's `adopted` line references the prior finding ID.
13. **Artifact validates.** Given the artifact of any `researched` run, when `publish-findings` validates it, then no `E-*` code is raised, `artifact_short_id` and every fingerprint recompute, numbering is dense, and `## Publication` is present and empty.
14. **Unverifiable citation.** Given a candidate whose only primary source fails to fetch, when verification runs, then the candidate is ledgered `rejected` with `citation-unverified` and never appears in the artifact.
15. **Budget exhausted.** Given `max_minutes: 10` and slow sources, when the cap is reached during the third branch, then that branch is cut, critique runs on the candidates returned so far, the receipt records `budget.exhausted: true`, and the Summary names the cut lens.
16. **Explicit lens.** Given `--lens cost-performance`, when the run executes, then exactly that lens runs regardless of rotation and the ledger lines carry it.

## Risks

- **Source drift.** Landing points and current-state docs go stale. Mitigation: the grounding pass reports unmatched paths in the Summary and receipt; the map is human-owned so the report is the repair signal; `question` findings absorb candidates that cannot be mapped.
- **Hallucinated or misattributed citations.** Mitigation: mandatory fetch verification of every cited URL before the artifact is written; secondary sources never suffice; the evidence floor; the ledger makes every verdict auditable.
- **Cost bounds.** Fan-out multiplies fetches and tokens. Mitigation: `budget.*`, effective branch count capped by runtime capacity, per-branch source cap, the avoid-list shrinking repeat work, and one area per run.
- **Non-reproducible searches.** Search results change between runs. Accepted: the ledger plus artifact make each run auditable even if not repeatable, and rotation ensures coverage over time rather than per run.
- **Constraint blindness in branches.** A branch that sees only the brief can propose something the wider repository forbids. Mitigation: constraints are in the brief, applicability scoring is in the critique with the grounding summary, and every action still passes human approval in the publisher.
- **Gate flooding.** Every `question` filed as a `human-decision` action lands on the human-gate frontier. Mitigation: questions are actionless unless they gate an improvement; the publisher's approval step is the second filter.

## Open items

1. Maintainer ratifies: the default weights (0.4/0.4/0.2), bar 3.5, evidence floor 2; the 180-day expiry; the six default lenses; `branch-pr` as the default persistence; the ledger line format; per-area (not global) lens rotation.
2. **#18** absorbs the additive `publication: not-attempted (no-actions)` line, or rules that empty artifacts are handed off anyway; either is workable, this document assumes the former.
3. **#18 open item 6** (a shared validator script) matters here: the fingerprint normalization ("removes punctuation") must be pinned to one implementation so producer and publisher agree; this document's example removed punctuation without substituting spaces.
4. Whether `deferred` sources should carry their own shorter re-visit window instead of being always revisitable; deferred to pilot data.
5. Whether the critique should be allowed to *raise* a candidate to `bug` on standards evidence, or whether research findings are always `improvement`/`question`; this document allows `bug` narrowly.

## Consequences

- **#8 (capability configuration):** absorb the flagged `research.*` additions; mark `research.persist` as asked during setup; confirm the inline `research.areas` form maps as described and that declaring both forms is an error.
- **#18 (findings publication):** the artifact above is the reference research fixture; the `ledger` frontmatter key is always set by this producer; `## Publications` stays append-only under this document's ledger format; the `not-attempted (no-actions)` addition.
- **#11 (dogfooding):** keep the producer-side structure parallel: artifact production with the same validation, the same `publication:` line, a `persist`-style policy for the docs PR, and a receipt in the same envelope with a `family` of its own.
- **#15 (validation and pilot strategy):** test the sixteen scenarios, the append-only ledger invariant under crash injection, rotation determinism across two machines sharing a ledger, fetch verification with a deliberately dead URL, drift of the generated `findings-artifact.md`, `setup.md`, and `protocol-contracts.md`, the ADR 0004 eight-word shingle diff of the public `SKILL.md` and references against the source package, and the hygiene-scan allowlist for this document's `example.com`, `acme/widgets`, `acme/uploader`, and the cited standards hosts.
- **#16 (pilot consumer):** the pilot repository needs an areas file with at least three areas and real landing points, a seeded ledger with entries inside and outside the expiry window, web egress, both runtimes (one with subagents disabled), and configurations with and without `publish-findings` installed; it re-verifies the example artifact's URLs.
- **#17 (documentation and launch):** README lists `research-project` with `publish-findings` as its optional companion for filing (not a required pair), documents the areas file and ledger as repository-owned inputs, and `docs/provenance.md` records this document as the clean-room drafting input with the shingle-diff result.

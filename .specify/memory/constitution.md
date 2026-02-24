<!--
SYNC IMPACT REPORT
==================
Version change  : 1.0.0 → 1.1.0  (MINOR — new Principle V added)
Modified principles : none
Added sections  :
  - Principle V: Documentation Language Standard (zh-TW)
  - Development Workflow: language mandate bullet
Removed sections : N/A
Templates reviewed :
  ✅ .specify/templates/plan-template.md   — Language mandate note added to header block
  ✅ .specify/templates/spec-template.md   — Language mandate note added to header block
  ✅ .specify/templates/tasks-template.md  — Language mandate note added; no structural changes needed
  ⚠  .specify/templates/commands/         — No command files detected; nothing to update
Deferred TODOs  : none
-->

# RagV1 Constitution

## Core Principles

### I. Code Quality (NON-NEGOTIABLE)

All production code MUST meet the following standards before it may be merged:

- **Style & formatting**: Every file MUST pass the project linter and formatter (no warnings
  suppressed without an inline justification comment).
- **Naming & clarity**: Identifiers MUST be self-describing; abbreviations are forbidden unless
  universally accepted in the domain (e.g., `llm`, `rag`, `api`).
- **No dead code**: Unused imports, variables, functions, and commented-out blocks MUST be
  removed prior to merge.
- **Documentation**: All public interfaces (classes, functions, API endpoints) MUST carry
  docstrings / JSDoc / XML-doc comments describing purpose, parameters, and return values.
- **Review gate**: Every pull-request MUST receive at least one peer approval. The author MUST
  NOT merge their own PR unless explicitly authorised by a maintainer.

**Rationale**: Consistent code quality is the primary defense against accumulating technical
debt. In a RAG system where data pipelines, retrieval logic, and generation components
interact, unreadable or inconsistent code dramatically increases the risk of silent
correctness bugs.

### II. Test-First Standards (NON-NEGOTIABLE)

Testing discipline follows strict Test-Driven Development (TDD):

- Tests MUST be written and reviewed **before** implementation code is produced
  (Red → Green → Refactor cycle is mandatory).
- Every feature MUST include:
  - **Unit tests** covering all pure functions and isolated components.
  - **Integration tests** covering component boundaries (e.g., retriever ↔ vector store,
    generator ↔ LLM client).
  - **Contract tests** for any external API or shared schema boundary.
- Minimum branch coverage: **80 %** enforced in CI; coverage regression blocks merge.
- Tests MUST be deterministic. Non-deterministic tests (e.g., LLM output comparison) MUST
  use snapshot-with-tolerance or mocked responses.
- No `skip`/`xfail` markers may be committed without a linked issue and expiry date.

**Rationale**: RAG pipelines are compositional and failure modes compound across retrieval,
ranking, and generation stages. Early test coverage ensures each layer's contract is
explicit and regressions are caught before they cascade.

### III. User Experience Consistency (NON-NEGOTIABLE)

All user-facing surfaces — API responses, CLI output, UI components, and error messages —
MUST conform to a single, unified interaction model:

- **API responses**: All REST/GraphQL responses MUST follow the project's canonical envelope
  schema (`{ data, error, meta }`). Deviations require a constitution amendment.
- **Error messages**: User-visible errors MUST be actionable, written in plain language, and
  free of internal stack traces or technical identifiers.
- **Terminology**: Domain terms (e.g., "document", "chunk", "query", "answer") MUST be used
  consistently across UI copy, API field names, documentation, and logs.
- **UI patterns**: Repeated interactions (search, filter, pagination, loading states) MUST
  reuse shared components; one-off implementations are prohibited.
- **Accessibility**: All UI surfaces MUST meet WCAG 2.1 AA requirements.

**Rationale**: Users interacting with a RAG-powered product expect predictable, coherent
behaviour. Inconsistent affordances erode trust and increase support burden.

### IV. Performance Requirements

Performance is a first-class feature, not a post-launch concern:

- **Baseline targets** — the following MUST be met in production-equivalent environments:
  | Operation | p50 | p95 | Hard limit |
  |---|---|---|---|
  | Query end-to-end (retrieval + generation) | ≤ 2 s | ≤ 5 s | 10 s |
  | Document ingestion (single chunk) | ≤ 200 ms | ≤ 500 ms | 1 s |
  | Vector similarity search (top-10, 10 k docs) | ≤ 50 ms | ≤ 150 ms | 300 ms |
- **Performance tests**: Every feature that touches a latency-sensitive path MUST include a
  performance benchmark in the test suite. Benchmarks run in CI on every PR to `main`.
- **Regression policy**: A PR that introduces a measurable regression (> 10 % increase at p95)
  relative to the `main` baseline MUST NOT merge until the regression is resolved or an
  exception is approved and documented.
- **Resource bounds**: Services MUST define explicit memory and CPU limits in deployment
  configuration. Exceeding limits in load tests blocks release.

**Rationale**: RAG workflows involve expensive vector operations and LLM inference.
Unmanaged latency accumulates across retrieval, reranking, and generation; disciplined
benchmarking prevents user-facing slowdowns before they reach production.

### V. Documentation Language Standard (NON-NEGOTIABLE)

All human-readable project artifacts MUST be authored in **Traditional Chinese (zh-TW)**:

- **Specifications** (`spec.md`): All sections — user stories, requirements, acceptance
  scenarios, success criteria — MUST be written in zh-TW.
- **Plans** (`plan.md`, `research.md`, `data-model.md`, `quickstart.md`): All narrative,
  decision rationale, complexity tracking, and structured summaries MUST be in zh-TW.
- **User-facing documentation**: README files, quickstart guides, API usage guides, release
  notes, and in-product help text MUST be in zh-TW.
- **Task descriptions** (`tasks.md`): Task titles and checkpoint descriptions MUST be in
  zh-TW; file paths and identifiers remain in English.
- **Exceptions** — the following MAY remain in English:
  - Source code identifiers (class names, method names, variable names).
  - Inline code comments where the comment is tightly coupled to an English identifier.
  - Commit messages (English is conventional for tooling compatibility).
  - API field names, schema keys, and contract definitions.
  - Third-party configuration file keys.
- **Mixed content**: When a document contains both prose and code blocks, prose MUST be in
  zh-TW; code blocks may use English as per the exceptions above.

**Rationale**: The primary audience and the core team operate in Traditional Chinese.
 Authoring all specifications and plans in zh-TW eliminates translation ambiguity, reduces
 misunderstandings during review, and ensures that user-facing documentation is immediately
 accessible without a translation step.

## Technology & Stack Constraints

- The primary language for all backend and pipeline code is **C# (.NET 9+)**.
- Infrastructure-as-code configurations MUST be version-controlled alongside application code.
- Third-party dependencies MUST be pinned to explicit versions in `Directory.Packages.props`
  (central package management is mandatory; floating versions are forbidden).
- LLM provider integrations MUST be abstracted behind an interface; no direct SDK calls in
  business logic.
- Vector store integrations MUST be abstracted behind a repository interface; swapping
  providers MUST require changes only in the infrastructure layer.

## Development Workflow

- **Branching**: Feature branches follow `###-short-description` naming (e.g.,
  `001-document-ingestion`).
- **Spec-first**: No implementation branch may be opened without an approved spec
  (`.specify/specs/###-feature/spec.md`).
- **CI gates**: All PRs MUST pass: linting, formatting, unit tests, integration tests,
  branch coverage threshold, and performance benchmarks before a review may be approved.
- **Release**: Releases follow Semantic Versioning. Release notes MUST reference spec numbers
  for every included feature.
- **Debt management**: Technical debt items MUST be logged as tracked issues and prioritised
  in the next planning cycle; undocumented debt is a constitution violation.
- **Language**: All specifications, plans, and user-facing documentation MUST be written in
  Traditional Chinese (zh-TW) per Principle V. Violation of this rule blocks merge.

## Governance

This constitution supersedes all other practices, guidelines, or conventions within the
RagV1 project. In the event of a conflict, the constitution takes precedence.

**Amendment procedure**:
1. Open a discussion issue proposing the change and referencing the affected principle(s).
2. Obtain approval from at least two maintainers.
3. Update this file, increment the version per the versioning policy below, and update
   `LAST_AMENDED_DATE`.
4. Provide a migration plan for any existing code that no longer complies.
5. Update all affected templates and agent guidance files within the same PR.

**Versioning policy**:
- MAJOR — backward-incompatible governance changes: principle removal or redefinition.
- MINOR — new principle or section added; materially expanded guidance.
- PATCH — clarifications, wording improvements, typo fixes.

**Compliance reviews**: Compliance with this constitution MUST be verified on every
pull-request via the "Constitution Check" gate defined in the plan template. Quarterly
deeper reviews are RECOMMENDED to ensure the constitution remains current.

**Version**: 1.1.0 | **Ratified**: 2026-02-25 | **Last Amended**: 2026-02-25

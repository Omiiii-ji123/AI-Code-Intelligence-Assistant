# Software Requirements Specification

## 1. Introduction

### Problem Statement

Developers joining unfamiliar codebases spend disproportionate time understanding architecture, dependencies, and execution flow before contributing meaningfully. Current AI coding assistants often provide conversational, ad-hoc responses but lack structured, repository-level engineering insight. This project builds a platform combining static analysis, embeddings, and LLM reasoning to produce structured, grounded software engineering insight — not conversational AI.

### In-Scope (MVP)

- Ingest a public Git repository (Python only)
- Parse code structure via Tree-sitter/AST behind a single extensible parser interface
- Generate embeddings, index in ChromaDB for semantic search
- Answer structural questions about the repo (architecture, dependencies, module purpose), grounded in parsed data
- Basic code smell / refactor suggestions
- Persist repository metadata and analysis results so previously analyzed repositories do not require full reprocessing
- Web UI to explore results

### Out-of-Scope

- General-purpose chatbot behavior
- Multi-language parsing (architecture must not block it later, but nothing beyond Python ships in MVP)
- Real-time collaborative editing
- CI/CD integration
- Auto-fixing code (suggest only, never auto-apply)

### Functional Success Criteria

- MVP deployed and publicly accessible on free-tier infra
- Demonstrably answers structural questions about a real Python open-source repo with grounded, non-hallucinated output
- Codebase clean enough to walk an interviewer through end-to-end
- Every design decision can be explained without notes

### Non-Functional Success Criteria

- Modular architecture with clear separation of concerns
- Clean and documented REST APIs
- Unit tests for core backend modules
- Dockerized deployment on Render
- Reasonable response latency for semantic search on free-tier infrastructure

## 2. User Roles

### Anonymous / Unauthenticated User

- Can submit a public Git repository URL for analysis
- Can perform semantic search and explore analysis results within a temporary session
- Session-scoped only — no persistence across devices/browsers, no saved history
- Anonymous session data lifecycle (expiration, cleanup) is defined in Functional Requirements, FR Group 6

### Authenticated User (Firebase Auth)

- All Anonymous capabilities, plus:
  - Save analyzed repositories for future reference
  - View history of past analyses
  - Reopen a previously completed analysis without reprocessing
  - Delete saved analyses

### Admin Role

- Explicitly excluded from MVP
- No role tables, permission matrices, admin enums, or generic policy engines are introduced to support this role
- Authentication and authorization logic shall remain modular enough to extend later, without a generalized RBAC system built in advance
- The system only distinguishes between unauthenticated (Anonymous) and authenticated access for the MVP

## 3. Functional Requirements

### FR Group 1: Repository Ingestion

| ID | Requirement |
|---|---|
| FR-001 | Repository URL Submission — The system shall accept a public GitHub repository URL as input for analysis. |
| FR-002 | Repository URL Format Validation — The system shall validate that the submitted URL is a well-formed GitHub repository URL before attempting any network operation. |
| FR-003 | Repository Accessibility Verification — The system shall verify that the validated URL points to an existing, publicly accessible GitHub repository (distinct from format validation — this requires a network check). |
| FR-004 | Repository Cloning — The system shall clone the verified repository into a temporary, isolated workspace for processing. |
| FR-005 | Python File Discovery — The system shall identify and analyze only `.py` files within the cloned repository, ignoring all other file types. |
| FR-006 | Empty/No-Python Handling — The system shall detect repositories containing no Python source files and return a clear error rather than proceeding with analysis. |
| FR-007 | Ingestion Failure Reporting — The system shall report ingestion failures (malformed URL, non-GitHub host, private/inaccessible repo, clone failure) with clear, user-facing error messages. |
| FR-008 | Analysis Scope Summary — The system shall provide the user a clear summary of analysis scope (e.g., number of Python files analyzed, confirmation that non-Python files were excluded) rather than an individual per-file report. |

**Traceability**

| FR ID | Traces to Charter Item |
|---|---|
| FR-001 | In-Scope: Ingest a public Git repository (Python only) |
| FR-002 | Constraint: GitHub-only scope |
| FR-003 | In-Scope: Repository ingestion |
| FR-004 | In-Scope: Repository ingestion |
| FR-005 | Constraint: Python-only analysis for MVP |
| FR-006 | Out-of-Scope: Multi-language parsing boundary |
| FR-007 | Functional Success Criteria: grounded, reliable behavior |
| FR-008 | Non-Functional Success Criteria: clean, transparent UX |

### FR Group 2: Code Parsing & Structural Analysis

| ID | Requirement |
|---|---|
| FR-009 | Python Source Parsing — The system shall parse each discovered Python file into a structured representation capturing modules, classes, functions, and imports. |
| FR-010 | Function-Level Metadata Extraction — For each function, the system shall extract its signature, docstring (if present), source file path, and line range. |
| FR-011 | Class and Module Metadata Extraction — For each class and module, the system shall extract its name, source file path, and line range. |
| FR-012 | Dependency Graph Construction — The system shall construct a graph of intra-repository import relationships between modules. |
| FR-013 | Parse Failure Handling — The system shall handle files that fail to parse (syntax errors, encoding issues) gracefully, excluding them from analysis without aborting the entire run. |
| FR-014 | Parser Extensibility — The system shall support adding parsers for additional programming languages in the future without requiring changes to repository ingestion or downstream analysis logic. |
| FR-015 | Repository Metadata Extraction — The system shall extract repository-level metadata, including the repository name, directory structure, and file hierarchy, to support repository understanding and navigation. |

**Explicit MVP exclusions:** call graphs, control-flow analysis, data-flow analysis, symbol resolution, type inference.

**Traceability**

| FR ID | Traces to Charter Item |
|---|---|
| FR-009 | In-Scope: parser behind extensible interface |
| FR-010 | In-Scope: parsing; supports semantic search |
| FR-011 | In-Scope: parsing |
| FR-012 | In-Scope: "Answer structural questions about the repo (architecture, dependencies, module purpose), grounded in parsed data" |
| FR-013 | Functional Success Criteria: grounded, reliable output |
| FR-014 | Constraint: extensible parser interface without major refactoring |
| FR-015 | In-Scope: "Web UI to explore results" (repository metadata supports navigation of results) |

### FR Group 3: Embeddings & Semantic Search

| ID | Requirement |
|---|---|
| FR-016 | Embedding Generation — The system shall generate vector embeddings for parsed code entities (functions, classes, modules) sufficient to support semantic retrieval. |
| FR-017 | Vector Index Storage — The system shall store generated embeddings in ChromaDB, associated with their source repository. |
| FR-018 | Semantic Search Query — The system shall accept a natural-language query and return the most semantically relevant code entities from the analyzed repository. |
| FR-019 | Search Result Presentation — The system shall return search results with sufficient context (file location, line range, signature/docstring) for the user to locate and understand the match. |
| FR-020 | Search Result Traceability — The system shall ensure that every semantic search result references its originating repository entity and source location so users can verify the relevance of the returned information. |

**Traceability**

| FR ID | Traces to Charter Item |
|---|---|
| FR-016 | In-Scope: embeddings/ChromaDB |
| FR-017 | In-Scope: embeddings/ChromaDB |
| FR-018 | In-Scope: "Generate embeddings, index in ChromaDB for semantic search" |
| FR-019 | Functional Success Criteria: usable output |
| FR-020 | Functional Success Criteria: non-hallucinated output |

### FR Group 4: LLM-Grounded Insights

| ID | Requirement |
|---|---|
| FR-021 | Architecture Summary Generation — The system shall generate, on user request, a natural-language summary of the repository's overall architecture, grounded in previously extracted structural metadata. |
| FR-022 | Module Explanation — The system shall generate, on user request, a natural-language explanation of a specific module's purpose and responsibilities, grounded in its parsed structure. |
| FR-023 | Dependency Explanation — The system shall generate, on user request, a natural-language explanation of dependency relationships between specified modules, grounded in the dependency graph. |
| FR-024 | Code Smell Detection — The system shall detect code smells (e.g., overly long functions, excessive parameters, missing docstrings) using deterministic, rule-based static analysis performed during repository ingestion. The LLM shall not determine whether a smell exists. |
| FR-025 | Refactoring Suggestion Generation — The system shall generate, on user request, natural-language explanations and refactoring suggestions for code smells already identified by the rule engine, grounded strictly in the specific smell(s) detected. |
| FR-026 | LLM Output Grounding Constraint — The system shall constrain LLM-generated responses to information derivable from parsed repository data and shall not permit free-form conversational responses unrelated to the analyzed repository. |

**Traceability**

| FR ID | Traces to Charter Item |
|---|---|
| FR-021 | In-Scope: "Answer structural questions about the repo (architecture, dependencies, module purpose), grounded in parsed data" |
| FR-022 | In-Scope: "Answer structural questions about the repo…" |
| FR-023 | In-Scope: "Answer structural questions about the repo…" |
| FR-024 | In-Scope: "Basic code smell / refactor suggestions"; Constraint: correctness/explainability priority |
| FR-025 | In-Scope: "Basic code smell / refactor suggestions" |
| FR-026 | Problem Statement: "structured, grounded software engineering insight — not conversational AI" |

### FR Group 5: Persistence & Caching

| ID | Requirement |
|---|---|
| FR-027 | Analysis Result Persistence — The system shall persist repository metadata, parsed structural data, and generated embeddings so that a previously analyzed repository does not require full reprocessing on subsequent requests. |
| FR-028 | Cache Validity Check — The system shall determine whether a cached analysis corresponds to the latest available commit of the target repository before reusing it. |
| FR-029 | Cache Invalidation on Repository Change — The system shall invalidate a cached analysis whenever the repository's current version no longer matches the version associated with the persisted analysis. |

**Explicit MVP exclusion:** Manual "Force Re-analyze" — logged as a candidate Future Enhancement, not a Functional Requirement.

**Traceability**

| FR ID | Traces to Charter Item |
|---|---|
| FR-027 | In-Scope: persist metadata/results |
| FR-028 | In-Scope: avoid full reprocessing |
| FR-029 | In-Scope: avoid stale results |

### FR Group 6: Anonymous Session Handling

| ID | Requirement |
|---|---|
| FR-030 | Anonymous Session Creation — The system shall create a temporary, unauthenticated session for any user who submits a repository for analysis without signing in. |
| FR-031 | Session-Scoped Access — The system shall scope an anonymous session's access, history, and record of its analysis and search activity to that session only, with no access from other sessions or devices. This scoping applies to the session's access relationship to analyses, not to the underlying shared repository analysis cache, which remains governed by FR-027–FR-029 and may be reused across sessions when its cached version remains valid. |
| FR-032 | Session Expiration and Data Removal — The system shall expire anonymous sessions after a configurable period of inactivity, and shall remove all session-scoped metadata, access, and history associated with an expired anonymous session. Expiration shall not delete the underlying repository analysis cache governed by FR-027–FR-029, which is independent of any individual session and may be reused by subsequent analyses while its cached version remains valid. |

**Explicit MVP exclusion:** Claiming anonymous session data after sign-up — logged as a Future Enhancement, not a Functional Requirement.

**Traceability**

| FR ID | Traces to Charter Item |
|---|---|
| FR-030 | Section 1: Anonymous User role — analyze without account |
| FR-031 | Section 1: Anonymous User role — session-scoped only, no persistence |
| FR-032 | Section 1: temporary session; Constraint: minimize free-tier storage cost |

### FR Group 7: Authenticated User Data Management

| ID | Requirement |
|---|---|
| FR-033 | User Authentication — The system shall authenticate users via Firebase Authentication before granting access to persistent data features. |
| FR-034 | Save Analyzed Repository — The system shall allow an authenticated user to save a repository analysis to their account for future access. If the user attempts to save a repository analysis already saved to their account, the system shall reuse or update the existing saved record rather than creating a duplicate. |
| FR-035 | View Analysis History — The system shall allow an authenticated user to view a list of their previously saved repository analyses. |
| FR-036 | Reopen Saved Analysis — The system shall allow an authenticated user to reopen a previously saved analysis without triggering reprocessing, provided the cached analysis is still valid (per FR-028). This validity check is shared functionality, not duplicated between authenticated and anonymous workflows. |
| FR-037 | Delete Saved Analysis — The system shall allow an authenticated user to delete a saved analysis from their own account. Deletion shall remove the analysis only from that user's account and shall not affect analyses or shared cached analysis data associated with other users. |
| FR-038 | Authenticated Session Repository Submission — The system shall allow an authenticated user to submit a new repository for analysis directly, with results eligible for saving under FR-034. |
| FR-039 | User Data Isolation — The system shall ensure that authenticated users can access, modify, and delete only their own saved-analysis records and account-associated history. This isolation applies to the user's saved-analysis records, not to the underlying shared repository analysis cache, which remains governed by FR-027–FR-029 and may be reused across users when its cached version remains valid. |

**Traceability**

| FR ID | Traces to Charter Item |
|---|---|
| FR-033 | Section 1: Authenticated User role |
| FR-034 | Section 1: Save analyzed repositories |
| FR-035 | Section 1: View analysis history |
| FR-036 | Section 1: Reopen previous analyses; In-Scope: caching/persistence |
| FR-037 | Section 1: Delete saved analyses |
| FR-038 | Section 1: Authenticated User capabilities |
| FR-039 | Constraint: correctness/explainability; implicit security requirement of multi-user persistence |

### FR Group 8: Web UI / Presentation

| ID | Requirement |
|---|---|
| FR-040 | Repository Submission Interface — The system shall provide a UI for submitting a public GitHub repository URL for analysis. |
| FR-041 | Analysis Progress Indication — The system shall indicate analysis progress or status to the user while repository analysis (ingestion, parsing, and embedding generation) is in progress. |
| FR-042 | Structural Overview Display — The system shall display the repository's structural overview (modules, classes, functions, dependency graph) in a navigable UI. |
| FR-043 | Semantic Search Interface — The system shall provide a UI for submitting natural-language search queries and viewing results with source context (per FR-019). |
| FR-044 | On-Demand Explanation Interface — The system shall provide a UI mechanism for requesting architecture, module, and dependency explanations on demand (per FR-021–023). |
| FR-045 | Code Smell & Refactoring Display — The system shall display detected code smells and their associated refactoring suggestions in the UI. |
| FR-046 | Authenticated User Dashboard — The system shall provide an authenticated user a dashboard listing their saved analyses, with options to reopen or delete each (per FR-035–037). |
| FR-047 | User Feedback and Error Presentation — The system shall present user-friendly success, warning, and error messages for repository ingestion, authentication, semantic search, analysis, and other user-initiated operations. |
| FR-048 | Empty State Presentation — The system shall present appropriate empty-state messages when no repository has been analyzed, no semantic search results are found, or an authenticated user has no saved analyses. |

**Traceability**

| FR ID | Traces to Charter Item |
|---|---|
| FR-040 | In-Scope: "Web UI to explore results" |
| FR-041 | In-Scope: "Web UI to explore results" |
| FR-042 | In-Scope: "Web UI to explore results"; In-Scope: "Answer structural questions about the repo (architecture, dependencies, module purpose), grounded in parsed data" |
| FR-043 | In-Scope: "Generate embeddings, index in ChromaDB for semantic search"; In-Scope: "Web UI to explore results" |
| FR-044 | In-Scope: "Answer structural questions about the repo (architecture, dependencies, module purpose), grounded in parsed data"; In-Scope: "Web UI to explore results" |
| FR-045 | In-Scope: "Basic code smell / refactor suggestions"; In-Scope: "Web UI to explore results" |
| FR-046 | Section 1: Authenticated User capabilities |
| FR-047 | In-Scope: "Web UI to explore results" |
| FR-048 | In-Scope: "Web UI to explore results" |

### Consolidated MVP Exclusions (Functional Requirements)

- Non-Python file analysis
- Call graphs, control-flow analysis, data-flow analysis, symbol resolution, type inference
- Manual "Force Re-analyze"
- Claiming anonymous session data after sign-up
- Admin role / RBAC

## 4. Non-Functional Requirements

### NFR Group 1: Performance & Resource Constraints

| ID | Requirement |
|---|---|
| NFR-001 | Repository Size Constraint — The system shall define and enforce an operational maximum repository size (file count and/or total size) to ensure ingestion, parsing, and embedding generation complete successfully within free-tier compute and memory limits. *Acceptance check:* the system rejects or gracefully handles repositories beyond the configured limit, rather than failing silently or crashing. *Not fixed in this SRS:* the exact numeric limit — an implementation/configuration decision, to be set empirically during implementation. |
| NFR-002 | Semantic Search Responsiveness — The system shall return semantic search results with responsiveness suitable for interactive use on free-tier infrastructure, without unexplained delay that would make search feel broken or unusable. *Acceptance check:* manual/exploratory verification that a user can issue a query and receive results without an indefinite or unexplained wait. *Not fixed in this SRS:* a numeric latency target — to be measured from real free-tier performance during testing/implementation. |
| NFR-003 | Long-Running Operation Feedback — The system shall provide progress or status feedback (per FR-041) for any analysis operation that runs long enough that a user could reasonably believe the system has stalled or failed. *Acceptance check:* no operation leaves the user without any status indication for an extended, unexplained period. *Not fixed in this SRS:* the specific duration threshold defining "long-running" — to be derived from observed pipeline timing during implementation/testing. |

**Traceability**

| NFR ID | Traces to Charter Item |
|---|---|
| NFR-001 | Constraint: "Budget: minimize cost; free-tier infra (Render, Firebase, Gemini) accepted as a known limitation, not a surprise — expect to hit rate/sleep limits and plan around them in HLD" |
| NFR-002 | Non-Functional Success Criteria: "Reasonable response latency for semantic search on free-tier infrastructure" |
| NFR-003 | FR-041 (Analysis Progress Indication); Constraint: "Budget: minimize cost; free-tier infra (Render, Firebase, Gemini) accepted as a known limitation, not a surprise — expect to hit rate/sleep limits and plan around them in HLD" |

### NFR Group 2: Reliability & Data Integrity

| ID | Requirement |
|---|---|
| NFR-004 | Fault Containment in Analysis Pipeline — A failure in one stage of the analysis pipeline (parsing, embedding generation, LLM-grounded insight generation) shall not corrupt the system's state or cause unrelated data (e.g., other users' sessions, other repositories' cached analyses) to become invalid or inconsistent. *Acceptance check:* a failure during one repository's analysis has no observable effect on any other repository's data, session, or previously completed analysis. |
| NFR-005 | No Partial Analysis Persisted as Valid Cache — The system shall not persist an incomplete analysis run as a complete, reusable cached result. If an analysis fails partway through, the incomplete run shall be discarded rather than marked as valid, and the next request for that repository shall restart the full analysis pipeline from the beginning. *Acceptance check:* a repository whose analysis previously failed partway through triggers a full re-analysis on the next request — no partial state is served or resumed. *Explicit MVP exclusion:* partial-stage persistence and resume/retry from a failed stage — logged as a Future Enhancement. |
| NFR-006 | Grounding Traceability Integrity — Every LLM-generated insight persisted or displayed by the system shall remain traceable to the specific parsed repository data it was grounded in, consistent with FR-020 and FR-026. *Acceptance check:* it is always possible to identify which parsed entity/data an insight was derived from — no orphaned or unattributable LLM output. |
| NFR-007 | Reliable Session Data Cleanup — Expired anonymous session data (per FR-032) shall be reliably removed such that expired session data does not persist indefinitely or accumulate unbounded on free-tier storage. *Acceptance check:* no expired anonymous session's data remains accessible or occupies storage beyond a bounded, verifiable cleanup process. |

**Traceability**

| NFR ID | Traces to Charter Item / FR |
|---|---|
| NFR-004 | Constraint: "Prioritize correctness, maintainability, and explainability over feature breadth — every feature must be fully understood, tested, and documented before new functionality is added" |
| NFR-005 | FR-027 (Analysis Result Persistence); FR-028 (Cache Validity Check); Constraint: correctness/consistency over feature breadth |
| NFR-006 | FR-020 (Search Result Traceability); FR-026 (LLM Output Grounding Constraint) |
| NFR-007 | FR-032 (Session Expiration and Data Removal); Constraint: "Budget: minimize cost; free-tier infra (Render, Firebase, Gemini) accepted as a known limitation, not a surprise — expect to hit rate/sleep limits and plan around them in HLD" |

### NFR Group 3: Security & Access Control

| ID | Requirement |
|---|---|
| NFR-008 | Authenticated Endpoint Protection — Any system operation that reads, modifies, or deletes data associated with an authenticated user's account shall require valid authentication before executing. *Acceptance check:* an unauthenticated request to any such operation is rejected, never partially executed. |
| NFR-009 | Cross-User Data Isolation Enforcement — The system shall enforce, at the point of every data access, that an authenticated user cannot read, modify, or delete another user's saved analyses — consistent with FR-039. *Acceptance check:* a request for another user's data, even if the resource ID is known or guessed, is rejected. |
| NFR-010 | Cross-Session Anonymous Isolation Enforcement — The system shall enforce that one anonymous session cannot access another anonymous session's scoped data — consistent with FR-031. *Acceptance check:* a request using a different or absent session identifier cannot retrieve another session's results. |
| NFR-011 | Secrets and Credential Protection — The system shall not expose API keys, database credentials, or other secrets in client-facing responses, logs, or error messages. *Acceptance check:* inspecting any client-facing response, log, or error message never reveals a credential or key value. |
| NFR-012 | Non-Execution of Untrusted Repository Content — The system shall treat all repository content as untrusted data and shall never execute, `eval`, or otherwise run code from an analyzed repository. *Acceptance check:* analysis of a repository containing malicious or arbitrary code does not result in that code executing within the system. |
| NFR-013 | Resource Abuse Prevention — The system shall apply reasonable limits to repository analysis submissions to prevent a single user or anonymous session from exhausting free-tier compute or API resources. *Acceptance check:* repeated, rapid repository submissions from a single user/session are constrained rather than processed without limit. *Not fixed in this SRS:* the specific numeric limits and enforcement mechanism — deferred to HLD/implementation. |

**Traceability**

| NFR ID | Traces to Charter Item / FR |
|---|---|
| NFR-008 | FR-033: User Authentication; Approved User Roles: Authenticated User |
| NFR-009 | FR-039: User Data Isolation |
| NFR-010 | FR-031: Session-Scoped Access |
| NFR-011 | Derived security requirement necessary to safely operate approved Firebase and Gemini integrations |
| NFR-012 | In-Scope: Public repository ingestion; Derived security requirement: third-party repository content is untrusted |
| NFR-013 | Constraint: Budget/free-tier infrastructure limitations |

### NFR Group 4: Maintainability & Architecture Quality

| ID | Requirement |
|---|---|
| NFR-014 | Separation of Concerns — The system's ingestion, parsing, embedding, persistence, and presentation logic shall be organized into distinct modules, each with clearly defined primary ownership of its responsibility, without unnecessary duplication of that responsibility across modules. *Acceptance check:* a reviewer can identify, for any major responsibility, which module owns it, and no responsibility is redundantly reimplemented in multiple modules without justification. |
| NFR-015 | Parser Interface Isolation — Consistent with FR-014, the language-parsing layer shall be isolated behind a defined interface such that Python-specific parsing logic does not leak into ingestion, persistence, or presentation layers. *Acceptance check:* ingestion/persistence/presentation code contains no direct dependency on Python-specific parsing internals. |
| NFR-016 | Documented Public Interfaces — All REST API endpoints and core module-level interfaces shall be documented with their purpose, inputs, and outputs. *Acceptance check:* a developer unfamiliar with the codebase can determine an endpoint's/module's contract from documentation alone, without reading its implementation. |
| NFR-017 | Consistent Code Style — The codebase shall follow a single, consistently applied code style and formatting standard across all modules. *Acceptance check:* an automated formatter/linter run against the codebase reports no style violations. |
| NFR-018 | Solo-Maintainability Constraint — The architecture shall avoid patterns, abstractions, or infrastructure components (e.g., microservices, message brokers, generic plugin frameworks) that are not justified by an actual approved FR or NFR. Every significant architectural component shall be justifiable by a specific requirement it exists to satisfy, with the sole explicit exception of the parser interface abstraction required by FR-014, which is permitted specifically to support future language extensibility. *Acceptance check:* for any significant architectural component under review, a reviewer can name the specific FR or NFR it satisfies; no component exists solely for hypothetical future flexibility outside the FR-014 exception. |

**Traceability**

| NFR ID | Traces to Charter Item / FR |
|---|---|
| NFR-014 | Non-Functional Success Criteria: "Modular architecture with clear separation of concerns" |
| NFR-015 | FR-014: Parser Extensibility |
| NFR-016 | Non-Functional Success Criteria: "Clean and documented REST APIs" |
| NFR-017 | Derived maintainability requirement supporting solo-developer explainability (Constraint: correctness/maintainability/explainability priority) |
| NFR-018 | Derived architectural governance requirement supporting Charter constraints: correctness, maintainability, and explainability over feature breadth; FR-014 parser-interface exception |

### NFR Group 5: Testability

| ID | Requirement |
|---|---|
| NFR-019 | Unit Test Coverage for Core Modules — Core backend modules (ingestion, parsing, embedding generation, persistence/caching logic, session handling) shall have unit tests covering their primary behavior and key documented edge/failure cases, prioritizing meaningful coverage over a numeric coverage target. *Acceptance check:* each core module has an associated test suite that exercises its main success path and at least its documented failure modes (e.g., FR-007, FR-013, NFR-005). |
| NFR-020 | Deterministic Component Testability — Deterministic system components (parsing, rule-based code smell detection, cache validity checks) shall be testable without requiring live network access, live LLM calls, or a live database connection. *Acceptance check:* the deterministic-component test suite runs successfully in isolation (e.g., in CI, offline), without external service dependencies. |
| NFR-021 | LLM-Dependent Component Test Isolation — Components that depend on the LLM API (architecture/module/dependency explanations, refactoring suggestions) shall be designed such that grounding data selection, input/context construction, traceability of data supplied to the LLM, and failure handling around LLM integration can each be tested independently of the LLM's actual generated wording. *Acceptance check:* it is possible to verify what data was sent to the LLM and how integration failures are handled, without asserting on the specific content of the LLM's response. |

**Traceability**

| NFR ID | Traces to Charter Item / FR |
|---|---|
| NFR-019 | Non-Functional Success Criteria: "Unit tests for core backend modules" |
| NFR-020 | Non-Functional Success Criteria: "Unit tests for core backend modules"; FR-024 (Code Smell Detection — deterministic, rule-based) |
| NFR-021 | FR-026 (LLM Output Grounding Constraint); FR-020 (Search Result Traceability); Derived testability requirement given non-deterministic external LLM behavior |

### NFR Group 6: Deployability

| ID | Requirement |
|---|---|
| NFR-022 | Containerized Deployment — The system shall be deployable via Docker, with all backend dependencies defined in a container image such that the application can be built and run consistently across environments. *Acceptance check:* the application starts and runs correctly from a fresh container build, without manual environment setup steps beyond configuration values. |
| NFR-023 | Configuration via Environment Variables — Environment-specific values (API keys, database connection strings, service URLs) shall be supplied via environment variables or an external configuration mechanism, not hardcoded into source code. *Acceptance check:* no credential or environment-specific value appears as a literal in source code; the same container image can run against different environments purely by changing configuration. |
| NFR-024 | Free-Tier Platform Compatibility — The system shall be deployable within the operational constraints of the approved free-tier infrastructure (Render, Firebase, Gemini), including tolerance for platform behaviors such as service sleep/cold-start on inactivity. *Acceptance check:* the deployed application recovers correctly from a cold start without requiring manual intervention. |
| NFR-025 | Deployment Reproducibility — The system's deployment process shall be documented as a manual, repeatable procedure such that the application can be redeployed from source without undocumented manual steps. Automated CI/CD pipeline tooling is not required for MVP. *Acceptance check:* following the documented deployment steps from a clean environment results in a working deployment, with no undocumented manual steps. |

**Explicit Future Enhancements (not MVP):** automated CI/CD deployment pipelines, blue/green or zero-downtime deployments, multi-region deployment, horizontal scaling.

**Traceability**

| NFR ID | Traces to Charter Item |
|---|---|
| NFR-022 | Non-Functional Success Criteria: "Dockerized deployment on Render" |
| NFR-023 | Derived deployment/security requirement supporting free-tier multi-environment deployment and NFR-011 (Secrets Protection) |
| NFR-024 | Constraint: "Budget: minimize cost; free-tier infra (Render, Firebase, Gemini) accepted as a known limitation, not a surprise — expect to hit rate/sleep limits and plan around them in HLD" |
| NFR-025 | Non-Functional Success Criteria: "Dockerized deployment on Render"; Out-of-Scope: "CI/CD integration" (defines the manual-process boundary) |

### NFR Group 7: Usability & Accessibility

| ID | Requirement |
|---|---|
| NFR-026 | Core Workflow Discoverability — A first-time user shall be able to identify how to submit a repository for analysis without requiring external instructions, using UI affordances alone (per FR-040). *Acceptance check:* the repository submission entry point is visible and identifiable on the initial page load, without scrolling or navigation. |
| NFR-027 | Consistent Feedback Across Workflow States — Every core workflow action (submission, progress, search, insight request, save/delete) shall present the user with a clear outcome — success, in-progress, empty, or error — consistent with FR-041, FR-047, and FR-048. *Acceptance check:* no core action leaves the UI in an ambiguous state where the user cannot tell whether it succeeded, failed, or is still running. |
| NFR-028 | Baseline Keyboard and Screen-Reader Accessibility — Core interactive elements (repository submission, search input, navigation between saved analyses) shall be operable via keyboard and shall expose accessible labels compatible with screen readers, using standard semantic HTML/ARIA practices. *Acceptance check:* core workflow actions can be completed using keyboard navigation alone, and interactive elements have non-empty accessible names. |
| NFR-029 | Responsive Layout for Common Viewport Sizes — The web UI shall remain usable at common desktop and tablet viewport widths without broken or inaccessible layout elements. *Acceptance check:* the core workflow can be completed without horizontal scrolling or overlapping elements at standard desktop and tablet breakpoints. |

**Explicit Future Enhancements (not MVP):** full WCAG conformance audit, mobile-first responsive design, internationalization/localization, theming/dark mode.

**Traceability**

| NFR ID | Traces to Charter Item / FR |
|---|---|
| NFR-026 | FR-040 (Repository Submission Interface) |
| NFR-027 | FR-041 (Analysis Progress Indication); FR-047 (User Feedback and Error Presentation); FR-048 (Empty State Presentation) |
| NFR-028 | Derived usability requirement supporting In-Scope: "Web UI to explore results" |
| NFR-029 | Derived usability requirement supporting In-Scope: "Web UI to explore results" |

## 5. Assumptions & Dependencies

### Assumptions

- **AS-001**: Target repositories submitted for analysis are reasonably sized for free-tier compute, consistent with the operational limit defined in NFR-001 (not itself a new limit — this assumption exists so NFR-001 is understood as protecting against, not solving, unbounded input size).
- **AS-002**: Users access the system via a modern web browser with JavaScript enabled and a stable internet connection.
- **AS-003**: Free-tier service quotas (Render, Firebase, Gemini) are sufficient to support demonstration-level and portfolio-review usage, not sustained production-scale traffic.
- **AS-004**: Users submit publicly accessible repositories, and the system's MVP scope is limited to validating public accessibility rather than independently determining repository licensing or broader usage permissions.
- **AS-005**: Submitted Python source files are syntactically valid in the general case; files that are not are handled per FR-013 (Parse Failure Handling), not assumed away entirely.

### Dependencies

- **DEP-001 — GitHub**: Availability and accessibility of GitHub for repository cloning and validation (FR-001–FR-004).
- **DEP-002 — Gemini API**: Availability of the Gemini API for embedding generation and LLM-grounded insight generation (FR-016, FR-021–FR-025).
- **DEP-003 — ChromaDB**: Availability of ChromaDB as the vector index for semantic search (FR-017).
- **DEP-004 — Firebase Authentication**: Availability of Firebase Authentication for authenticated user identity (FR-033).
- **DEP-005 — Render**: Availability of Render as the hosting/deployment platform (NFR-022, NFR-024).
- **DEP-006 — Persistent Data Store**: Availability of a persistent data store for repository metadata, analysis results, and authenticated user data. The specific database technology shall be selected during High-Level Design.

## 6. Constraints

### 6.1 Charter-Inherited Constraints

- **Timeline**: Multi-month, solo build, no external review — code and docs must be self-explanatory enough to defend solo in an interview.
- **Budget**: Minimize cost; free-tier infra (Render, Firebase, Gemini) accepted as a known limitation, not a surprise — expect to hit rate/sleep limits and plan around them in HLD.
- **Language Scope**: Python-only analysis for MVP; parser layer designed behind one clean interface (not a plugin framework) to allow future languages without rewrites.
- **Engineering Priority**: Prioritize correctness, maintainability, and explainability over feature breadth — every feature must be fully understood, tested, and documented before new functionality is added.

### 6.2 Requirements-Derived Constraints

- **Repository host scope**: Limited to public GitHub repositories only (FR-002, FR-003).
- **Access model**: MVP access is limited to unauthenticated and authenticated users; no admin-specific functionality, role tables, permission matrices, or RBAC system is implemented in the MVP.
- **Numeric operational thresholds deferred**: Repository size limits, latency targets, feedback-trigger durations, and rate-limiting thresholds are configuration values set empirically during implementation/testing (NFR-001, NFR-002, NFR-003, NFR-013).
- **No partial-resume processing**: Failed analyses restart from the beginning; partial-stage persistence and resume logic excluded from MVP (NFR-005).
- **No automated CI/CD**: Deployment is a documented, repeatable manual process (NFR-025, Charter Out-of-Scope: "CI/CD integration").
- **Database technology deferred**: The specific persistent data store technology is deferred to High-Level Design (DEP-006).
- **Advanced static analysis excluded**: Call graphs, control-flow, data-flow, symbol resolution, type inference excluded (FR Group 2 exclusions).
- **Advanced UX/accessibility excluded**: Full WCAG conformance, mobile-first design, i18n/l10n, theming excluded (NFR Group 7 exclusions).

## 7. Open Items

*None currently open. All items previously raised during Requirements Analysis (anonymous session lifecycle; database technology selection; audit-identified wording ambiguities in FR-031, FR-034, FR-039, FR-041) have been resolved and incorporated above. Database technology selection remains a deliberate deferral to High-Level Design (see DEP-006, Section 6.2) — not an open requirements gap.*

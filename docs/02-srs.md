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
| FR-022 | In-Scope: "Answer structural questions about the repo (architecture, dependencies, module purpose), grounded in parsed data" |
| FR-023 | In-Scope: "Answer structural questions about the repo (architecture, dependencies, module purpose), grounded in parsed data" |
| FR-024 | In-Scope: "Basic code smell / refactor suggestions"; Constraint: "Prioritize correctness, maintainability, and explainability over feature breadth — every feature must be fully understood, tested, and documented before new functionality is added" |
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
| FR-031 | Session-Scoped Access — The system shall scope an anonymous session's analysis results and search activity to that session only, with no access from other sessions or devices. |
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
| FR-034 | Save Analyzed Repository — The system shall allow an authenticated user to save a repository analysis to their account for future access. |
| FR-035 | View Analysis History — The system shall allow an authenticated user to view a list of their previously saved repository analyses. |
| FR-036 | Reopen Saved Analysis — The system shall allow an authenticated user to reopen a previously saved analysis without triggering reprocessing, provided the cached analysis is still valid (per FR-028). This validity check is shared functionality, not duplicated between authenticated and anonymous workflows. |
| FR-037 | Delete Saved Analysis — The system shall allow an authenticated user to delete a saved analysis from their own account. Deletion shall remove the analysis only from that user's account and shall not affect analyses or shared cached analysis data associated with other users. |
| FR-038 | Authenticated Session Repository Submission — The system shall allow an authenticated user to submit a new repository for analysis directly, with results eligible for saving under FR-034. |
| FR-039 | User Data Isolation — The system shall ensure that authenticated users can access, modify, and delete only repository analyses associated with their own account. |

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
| FR-041 | Analysis Progress Indication — The system shall indicate analysis progress or status to the user while ingestion and parsing are in progress. |
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

## 4. Non Functional Requirements

*Pending — to be completed in the next Requirements Analysis session.*

## 5. Assumptions

*Pending.*

## 6. Constraints

*Pending — Non-Functional and cross-cutting constraints from the approved Project Charter will be incorporated here.*

## 7. Open Items

*None currently open within the finalized Functional Requirements. Prior open item (anonymous session lifecycle) was resolved in FR Group 6 (FR-030–FR-032).*

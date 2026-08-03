Problem Statement
Developers joining unfamiliar codebases spend disproportionate time understanding architecture, dependencies, and execution flow before contributing meaningfully. Current AI coding assistants often provide conversational, ad-hoc responses but lack structured, repository-level engineering insight. This project builds a platform combining static analysis, embeddings, and LLM reasoning to produce structured, grounded software engineering insight — not conversational AI.

In-Scope (MVP)

Ingest a public Git repository (Python only)
Parse code structure via Tree-sitter/AST behind a single extensible parser interface
Generate embeddings, index in ChromaDB for semantic search
Answer structural questions about the repo (architecture, dependencies, module purpose), grounded in parsed data
Basic code smell / refactor suggestions
Persist repository metadata and analysis results so previously analyzed repositories do not require full reprocessing
Web UI to explore results

Out-of-Scope

General-purpose chatbot behavior
Multi-language parsing (architecture must not block it later, but nothing beyond Python ships in MVP)
Real-time collaborative editing
CI/CD integration
Auto-fixing code (suggest only, never auto-apply)

Constraints

Timeline: multi-month, solo build, no external review — code and docs must be self-explanatory enough to defend solo in an interview
Budget: minimize cost; free-tier infra (Render, Firebase, Gemini) accepted as a known limitation — expect to hit rate/sleep limits and plan around them in HLD
Language: Python-only analysis for MVP; parser layer designed behind one clean interface (not a plugin framework) to allow future languages without rewrites
Prioritize correctness, maintainability, and explainability over feature breadth — every feature must be fully understood, tested, and documented before new functionality is added

Functional Success Criteria

MVP deployed and publicly accessible on free-tier infra
Demonstrably answers structural questions about a real Python open-source repo with grounded, non-hallucinated output
Codebase clean enough to walk an interviewer through end-to-end
You can explain every design decision without notes

Non-Functional Success Criteria

Modular architecture with clear separation of concerns
Clean and documented REST APIs
Unit tests for core backend modules
Dockerized deployment on Render
Reasonable response latency for semantic search on free-tier infrastructure


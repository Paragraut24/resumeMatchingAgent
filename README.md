# Pathfinder — AI Job-Matching Agent

> **Status:** Under Active Development (Milestone M0/M1). Full functionality, live demo endpoints, and measured evaluation metrics will be updated as build milestones progress.

Pathfinder is an AI-powered job-matching agent. It reads a candidate's resume (PDF or text), extracts a structured candidate profile via schema-constrained LLM parsing, finds relevant jobs using semantic vector retrieval (pgvector), and ranks them using a transparent hybrid scoring function. Candidates can chat with an agent that uses tools to refine search results, inspect job postings, and provide grounded match rationales and skill gap analyses.

---

## Project Status

This repository is currently in the initial setup and data preparation phase (**Milestone M0 / M1**).
- Architecture and specification documents are finalized in docs `01` through `10`.
- PostgreSQL + pgvector schema, migrations, and job ingestion pipelines are in progress.
- Evaluation harnesses and benchmark baselines are scheduled for implementation in Milestone M4.

---

## Planned Architecture & How It Works

1. **Ingestion & Profiling:** Jobs and candidate resumes are parsed with schema-constrained LLM tool calls (`record_job_info`, `record_profile`). Unstructured descriptions are transformed into normalized profiles and skills.
2. **Embeddings & Vector Search:** Profiles and job postings are embedded using OpenAI `text-embedding-3-small` (1536 dimensions) and indexed in PostgreSQL using `pgvector` with HNSW indexing.
3. **Deterministic Hybrid Re-Ranking:** Retrieves the top 50 candidates by cosine similarity, then computes a composite score:
   $$\text{Final Score} = 0.60 \times \text{Semantic} + 0.25 \times \text{Skill} + 0.15 \times \text{Meta}$$
   Matches include transparent breakdowns, matched skills, and gap lists.
4. **Tool-Calling Agent:** A hand-rolled agent loop (without third-party agent framework bloat) with bounded turns (max 6 calls), short-term message memory, schema validation, and strict groundedness checks ensuring no invented job IDs.
5. **Systematic Evals:** Evaluated on a labeled evaluation set measuring Recall@10, Precision@5, nDCG@10, and MRR across Vector-Only (Config A) vs. Hybrid Re-Rank (Config B).

---

## Tech Stack

- **Backend:** Python 3.12, FastAPI, Uvicorn, Pydantic v2
- **Database & Storage:** PostgreSQL with `pgvector`, `psycopg` 3
- **LLM & Embeddings:** Anthropic API (Claude), OpenAI API (`text-embedding-3-small`)
- **Frontend:** React + Vite + TypeScript
- **Testing & Tooling:** Pytest, Ruff, Docker, GitHub Actions CI

---

## Roadmap & Milestones

- [x] **M0 — Architecture & Specifications:** PRD, schema, API contract, agent loop design, and eval plans finalized.
- [ ] **M1 — Data Ingestion:** Job snapshot pipeline, LLM skill extraction, and pgvector upserts.
- [ ] **M2 — Retrieval, Ranking & Core API:** Resume parsing, hybrid scoring, and FastAPI match endpoints.
- [ ] **M3 — Agent Implementation:** Tool execution, session persistence, and groundedness guards.
- [ ] **M4 — Evaluation Suite:** Labeled case evaluation, nDCG@10 benchmarking, and agent safety checks.
- [ ] **M5 — Deployment:** Docker containerization, cloud hosting, and React UI.
- [ ] **M6 — Polish & Reporting:** Real benchmark numbers and final documentation.

---

## Specifications & Documentation

The complete engineering design is documented in the repository:
- `01-project-brief-prd.md`: Project Brief & PRD
- `02-scope-and-non-goals.md`: Scope & Non-Goals
- `03-tech-stack-and-architecture.md`: Tech Stack & Architecture
- `04-data-schema.md`: Database Schema & Queries
- `05-api-contract.md`: REST API Specifications
- `06-agent-design.md`: Agent Loop & Tool Architecture
- `07-eval-plan.md`: Evaluation Framework & Quality Metrics
- `08-build-plan.md`: Phased Build Plan & Roadmap
- `09-devlog.md`: Devlog & Decision Records
- `10-README-template.md`: Release README Template

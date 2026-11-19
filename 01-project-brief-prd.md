# 01 — Project Brief / PRD: Pathfinder

**Status:** v1.0 · **Owner:** Parag · **Related docs:** 02 Scope, 03 Architecture, 04 Schema, 05 API, 06 Agent, 07 Evals, 08 Build Plan, 09 Devlog, 10 README

---

## 1. One-line summary

Pathfinder is an AI job-matching agent. A user submits a resume; the system parses it, finds relevant jobs with semantic search, ranks them with a transparent scoring function, and lets the user chat with an agent that can refine the search and explain every match. Retrieval quality is measured with a labeled evaluation set.

## 2. Why this project exists

The goal is a resume project that proves the exact skills a small AI-career-product team screens for: LLM agents with tool use and structured output, embeddings and semantic search, a ranking/recommendation pipeline, evaluation of LLM-system quality, a Python backend with SQL, and a deployed service used by real people. Every design choice should be explainable in an interview.

## 3. Target users

| User | Need |
|---|---|
| Student or early-career job seeker (primary) | Find relevant roles without reading hundreds of postings, and understand why a role fits. |
| Reviewer / recruiter reading the repo (secondary) | Quickly see working software, clean architecture, and measured results. |

## 4. Problem statement

Keyword search on job boards misses relevant roles whose wording differs from the resume, and gives no explanation of fit. Pathfinder matches on meaning, shows a score breakdown, and states skill gaps.

## 5. Goals

1. Upload a resume (PDF or pasted text) and receive a structured candidate profile.
2. Return the top matching jobs from a fixed jobs index using semantic retrieval followed by re-ranking.
3. Provide a chat agent that uses tools (search, job detail, candidate profile, scoring) and returns structured recommendations with rationales.
4. Accept like / dislike / applied feedback and store it.
5. Report retrieval quality with numbers from a labeled eval set (see doc 07).
6. Run as a Dockerized service in the cloud and be used by real people.

## 6. Non-goals

See doc 02. In short: no user accounts, no large-scale scraping, no fine-tuning, no agent framework, no resume rewriting.

## 7. Core user flows

1. **Onboard:** user uploads a resume → API parses it → profile returned and stored.
2. **Browse matches:** user opens matches → top 10 jobs with scores, matched skills and missing skills.
3. **Chat:** user asks "show me remote backend roles", "why is job X a fit?", "only entry-level" → agent calls tools and returns structured recommendations.
4. **Feedback:** user likes / dislikes / marks applied → stored for later analysis.
5. **Delete data:** user can delete their candidate record and all related data.

## 8. Functional requirements

| ID | Requirement |
|---|---|
| FR-1 | Accept resume as PDF (max 5 MB) or text (max 20,000 characters). |
| FR-2 | Extract a structured profile (skills, seniority, titles, education, summary, preferred locations) via a schema-constrained LLM call. |
| FR-3 | Embed jobs and candidate profiles with one embedding model and one fixed dimension (1536). |
| FR-4 | Retrieve the top 50 jobs by cosine similarity, then re-rank and return the top N (default 10, max 25). |
| FR-5 | Every match returns a score breakdown (semantic, skill, meta) and matched/missing skills. |
| FR-6 | The agent loop supports at most 6 model calls per user message and must end with a validated response. |
| FR-7 | Recommended job IDs must come from tool results in the same turn (no invented jobs). |
| FR-8 | Session history persists so context carries across messages (last 20 messages are sent to the model). |
| FR-9 | Feedback is stored per (candidate, job) with one row per pair. |
| FR-10 | `DELETE /candidates/{id}` removes the candidate and all dependent rows. |
| FR-11 | An offline eval script computes Recall@10, Precision@5, nDCG@10 and MRR for two required pipeline configurations (A: vector only, B: hybrid re-rank) and an optional third (C: B + LLM re-rank). |

## 9. Non-functional requirements

| Area | Requirement |
|---|---|
| Latency (targets, to be measured) | `GET /candidates/{id}/matches` p95 under 1.5 s (no LLM call). Agent reply p95 under 20 s. |
| Reliability | Retries with backoff on upstream API errors; clear 502 on persistent failure. |
| Security | Secrets only in environment variables; upload limits; rate limiting; untrusted text treated as data (see doc 06). |
| Privacy | No name/email stored in the parsed profile; resume text deletable on request. |
| Cost | Daily LLM call cap (`DAILY_LLM_CALL_CAP`) so the public demo cannot run up a bill. |
| Reproducibility | Frozen jobs snapshot for evals; pinned dependencies; one command to run locally. |

## 10. Success metrics

| Metric | Target | How measured |
|---|---|---|
| Retrieval quality | Config B (hybrid re-rank) beats Config A (vector only) on held-out cases for nDCG@10 | `evals/run_eval.py` (doc 07) |
| Groundedness | 100% of recommended job IDs exist in that turn's tool results | Agent eval checks + unit test |
| Schema validity | 100% of agent responses pass the response schema (after at most one retry) | Agent eval checks |
| Real usage | At least 5 people outside the developer complete the flow | Feedback table + notes in devlog |
| Delivery | Public URL, Docker image builds from clean checkout, README accurate | Manual checklist in doc 08 |

Do not promise numeric quality targets before the baseline exists. Record the baseline first, then report the improvement honestly.

## 11. Assumptions and risks

| Risk | Mitigation |
|---|---|
| Job data terms of use or rate limits block ingestion | Verify the terms of the chosen source before bulk ingest; keep a static fallback dataset; freeze a snapshot for evals. |
| Small eval set gives noisy comparisons | Report counts, keep a dev/test split, treat tiny differences as noise (doc 07). |
| LLM output drifts or breaks schema | Forced tool-call schemas, validation, one repair retry, temperature 0 for parsing. |
| Scope creep | Doc 02 is the contract; new ideas go to the devlog backlog. |
| Time | Milestone gates in doc 08; ship a working retrieval demo first. |

## 12. Glossary

- **RAG:** retrieve relevant documents first, then let the LLM reason over them.
- **Semantic score:** cosine similarity between candidate and job embeddings, min-max normalized within the retrieved pool.
- **Recall@k / Precision@k / nDCG@k / MRR:** standard ranking metrics (defined in doc 07).
- **Job key:** stable identifier `"{source}:{source_id}"`, used in eval labels because database UUIDs change between environments.

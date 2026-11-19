# 02 — Scope & Non-Goals

**Status:** v1.0 · **Purpose:** the contract for what gets built in the 2–3 week version. If it is not in "In scope", it is not built until the devlog records a deliberate change.

---

## 1. In scope (must ship)

| # | Item | Doc |
|---|---|---|
| 1 | Jobs ingestion CLI that loads a frozen snapshot (JSONL) and optionally pulls from a live source | 03, 04 |
| 2 | LLM-based skill extraction for jobs at ingest time | 04, 06 |
| 3 | Embedding of jobs and candidate profiles (text-embedding-3-small, 1536 dimensions) | 03, 04 |
| 4 | Resume parsing (PDF text extraction + schema-constrained LLM call) | 05, 06 |
| 5 | Semantic retrieval (top 50) + hybrid re-ranking (weights 0.60 / 0.25 / 0.15) | 03, 06 |
| 6 | REST API with the endpoints listed in doc 05 | 05 |
| 7 | Hand-rolled agent loop with five tools and structured final output | 06 |
| 8 | Session memory (persisted messages, 20-message window) | 04, 06 |
| 9 | Feedback capture (like / dislike / applied) | 05 |
| 10 | Eval harness with two required configurations (A, B), an optional third (C), and four metrics | 07 |
| 11 | Minimal React UI (upload, matches, chat, feedback) | 03 |
| 12 | Dockerfile, CI (lint + tests), cloud deploy, README with real measured results | 08, 10 |

## 2. Stretch (only after everything above is done and the eval report exists)

- Optional LLM re-rank stage as Config C (it is part of the eval comparison, so build it only if time allows; if skipped, report A vs B only and say so).
- Cover-letter generator tool.
- Learning a ranking-weight adjustment from feedback data.
- Saved searches / email alerts.

## 3. Out of scope (explicit non-goals)

| Non-goal | Reason |
|---|---|
| User accounts, login, OAuth | Not what the role screens for; adds security surface. Candidates are identified by an unguessable UUID. |
| Large-scale scraping or building a crawler | Fragile, possible terms-of-service issues, distracts from agent and ranking work. |
| Fine-tuning or training models | Out of the time budget; embeddings + prompting are sufficient. |
| LangChain / LlamaIndex / agent frameworks | A hand-rolled loop shows understanding of the mechanics. |
| Resume rewriting or ATS scoring | Different product; not required to demonstrate the target skills. |
| Mobile app, notifications | Not needed. |
| Payments, multi-tenancy, admin dashboard | Not needed. |
| Heavy frontend polish | The role is backend-heavy; UI only needs to be usable. |
| Multi-language resumes | English only in v1. |
| Storing candidate name, email, phone | Reduces privacy risk; not needed for matching. |

## 4. Constraints

- Time budget: 20 working days (see doc 08).
- Cost: free tiers where possible; hard daily LLM call cap on the public demo.
- One embedding model and one dimension for the whole system. Changing either requires re-embedding every row and is a recorded decision.
- Jobs index size: 200–500 postings for evals (frozen), more allowed for the live demo.

## 5. Change-control rule

When a new idea appears:
1. Write it under "Backlog" in doc 09.
2. Do not build it unless it replaces something of equal size from "In scope" or all in-scope items are done.
3. If scope changes, update this document and note the reason in the devlog.

## 6. Definition of done (project level)

- [ ] All 12 in-scope items work end to end on the deployed service.
- [ ] Eval report exists with baseline and improvement numbers from the held-out cases.
- [ ] At least 5 outside users have tried it; feedback rows exist.
- [ ] Docker build works from a clean checkout; CI is green.
- [ ] README matches reality and contains real numbers, not placeholders.
- [ ] Devlog contains the major decisions and what changed.

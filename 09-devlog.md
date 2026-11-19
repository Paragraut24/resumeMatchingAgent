# 09 — Progress Log / Devlog

**Purpose:** the single place that prevents losing context mid-build. Update it at the end of every working session. Newest entries go at the top of the "Daily log" section.

---

## How to use this file

1. **Daily log:** one entry per working session using the template below. Keep it short but specific.
2. **Decision log:** every non-trivial decision gets an ID (D-xxx) with context, options, decision and consequences. Never edit a past decision; add a new one that supersedes it.
3. **Doc change log:** when you change another doc, note which one and why.
4. **Backlog:** ideas that are out of scope go here, not into the code.
5. **Open questions / blockers:** anything unresolved.

### Daily entry template

```
### YYYY-MM-DD — Milestone Mx — Day N
**Done:**
- ...
**Decisions:** (reference D-xxx IDs)
**Blockers / surprises:**
- ...
**Docs changed:** (e.g. 05 API: added field X because Y)
**Next:**
- first thing to do tomorrow
```

---

## Daily log

### (Start here) — Milestone M0 — Day 0
**Done:**
- Planning documents 01–10 written.

**Next:**
- Create repo, database, API keys; verify DB, embeddings and LLM connectivity (M0 exit gate).

---

## Decision log

### D-001 — Build an AI job-matching agent as the resume project
- **Context:** the target role screens for LLM agents, embeddings/semantic search, recommender/ranking, RAG/evals, Python backend, SQL, deployment and real users.
- **Options:** a generic AI app; a job-matching agent that mirrors the product domain.
- **Decision:** job-matching agent.
- **Consequences:** every in-scope item maps to a stated requirement; scope must stay tight (doc 02).

### D-002 — Hand-rolled agent loop, no framework
- **Context:** demonstrate understanding of tool use.
- **Decision:** direct Anthropic SDK calls with a loop of at most 6 model calls.
- **Consequences:** more code to test (covered with a fake LLM client); no framework lock-in.

### D-003 — Postgres + pgvector for relational data and vectors
- **Context:** the role lists SQL and embeddings.
- **Decision:** one database, `vector(1536)` columns, cosine distance, HNSW index.
- **Consequences:** changing the embedding model or dimension requires a migration and re-embedding everything.

### D-004 — Embeddings: `text-embedding-3-small`, 1536 dimensions
- **Decision:** configured via `EMBEDDING_MODEL` and `EMBEDDING_DIM`; startup check verifies the dimension matches the schema.
- **Consequences:** an OpenAI API key is needed in addition to the Anthropic key.

### D-005 — Two-stage ranking: vector top 50, then deterministic hybrid score
- **Decision:** `final = 0.60 * semantic + 0.25 * skill + 0.15 * meta` with min-max normalized semantic score (doc 06).
- **Consequences:** weights are tunable on the dev split only; the baseline (vector only) stays available for the eval comparison.

### D-006 — Frozen jobs snapshot for evals; labels keyed by job key
- **Decision:** labels use `"{source}:{source_id}"`; the snapshot is never modified after labeling starts.
- **Consequences:** a new snapshot invalidates old labels.

### D-007 — Structured output through forced tool calls validated with Pydantic
- **Decision:** `record_profile`, `record_job_info` and `submit_recommendations` define the output schemas; one repair retry on validation failure.

### D-008 — No user accounts in v1; candidates are identified by random UUIDs; delete endpoint provided
- **Consequences:** anyone with a candidate UUID can read that profile. Acceptable for a demo; documented in the README.

### D-009 — Vite + React + TypeScript for the minimal frontend
- **Decision:** static build, deployed separately from the API; CORS restricted to `ALLOWED_ORIGINS`.

*(Add D-010 onward as you go.)*

---

## Doc change log

| Date | Doc | Change | Reason |
|---|---|---|---|
| | | | |

---

## Backlog (out of scope until approved)

- Optional LLM re-rank stage (Config C) if not already built.
- Cover-letter generator tool.
- Use feedback signals to adjust ranking.
- Automatic purge of candidates after 30 days.
- Saved searches and alerts.

---

## Open questions / blockers

- Which jobs source and what are its terms, rate limits and redistribution rules? (resolve in M1 before bulk ingest)
- Confirm current model IDs for `ANTHROPIC_MODEL` and `ANTHROPIC_PARSE_MODEL` against the provider docs (resolve in M0).
- Who will be the second labeler for the agreement check? (resolve before M4)

---

## Eval run log

| Date | Split | Configs | nDCG@10 | Recall@10 | Precision@5 | MRR | Notes / commit |
|---|---|---|---|---|---|---|---|
| | | | | | | | |

---

## User feedback log (from real users)

| Date | Who (anonymous label) | What they tried | What went wrong / what they liked | Action |
|---|---|---|---|---|
| | | | | |

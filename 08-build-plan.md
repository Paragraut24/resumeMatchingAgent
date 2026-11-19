# 08 — Phased Build Plan / Roadmap

**Status:** v1.0 · **Duration:** 20 working days (Day 0 is setup) · **Related:** 02 Scope, 09 Devlog

Each milestone has deliverables, an exit gate, and the docs that must be updated. Do not start the next milestone until the gate passes. If a milestone slips, cut stretch items first, then reduce UI scope; never cut evals.

---

## Application timing (do this first)

Apply to the Elip role now instead of waiting for the project to finish. Describe the project as in progress, link the repository, and update your application or follow up when milestone M2 is working. The job posting was about a week old at the time this plan was written, and roles like this can fill early.

## Dependency overview

```
M0 Setup → M1 Data → M2 Retrieval + API → M3 Agent → M4 Evals → M5 Deploy + Users → M6 Polish
                         ↘ (apply with repo link here)
```

## M0 — Setup (Day 0)

Deliverables:
- Repo initialized, `.gitignore` (includes `.env`), `.env.example`, `requirements.txt` placeholder.
- Python 3.12 virtual environment.
- Neon or Supabase Postgres created; `CREATE EXTENSION vector` verified.
- Anthropic and OpenAI API keys obtained and stored locally in `.env`.
- Docs 01–08 copied into `docs/`.
- GitHub repository with a basic CI workflow (`ruff`, `pytest`).

Exit gate: a script can connect to the database, call the embeddings API once, and call the LLM API once.

## M1 — Data (Days 1–3)

Deliverables:
- `migrations/001_init.sql` applied via `scripts/migrate.py`.
- Jobs source decided after reading its terms and rate limits; static fallback ready.
- `data/jobs_snapshot.jsonl` with 200–500 postings.
- `data/skill_aliases.json` (start small and grow it).
- `scripts/ingest_jobs.py`: normalize, skill extraction, seniority, embeddings, upsert with `content_hash` skip.
- Unit tests for text building, hashing and alias normalization.

Exit gate: the database holds the snapshot with embeddings; re-running ingest makes no new embedding calls; spot-check 20 jobs for sane `required_skills` and `seniority`.

Risks: source terms or rate limits (fall back to a static dataset); noisy skill extraction (improve the prompt and alias map).

Docs to update: 04 (if the schema changed), 09.

## M2 — Retrieval, ranking, core API (Days 4–7)

Deliverables:
- Resume parsing (PDF + text) with forced-tool schema and Pydantic validation.
- Candidate embedding and storage.
- Retrieval query, scoring (`ranking.py`) and `GET /candidates/{id}/matches`.
- `GET /jobs/search`, `GET /jobs/{id}`, `GET /health`, `DELETE /candidates/{id}`.
- Error format and request IDs.
- Tests: scorer (fixed inputs), min-max edge cases, API tests with a fake embedding client.
- A first manual look: five test resumes produce plausible matches.

Exit gate: upload a resume and receive plausible ranked matches in under the latency target, using only code you can explain line by line. **This is the "working retrieval demo" to reference in applications.**

Docs to update: 05, 06 (if scoring changed), 09.

## M3 — Agent (Days 8–12)

Deliverables:
- Tool definitions and executors with validation.
- Agent loop with the 6-call cap, error `tool_result` handling, history trimming and groundedness guard.
- Sessions and messages persistence; `POST /sessions`, `POST /sessions/{id}/messages`.
- Fake-LLM tests for loop behavior.
- Feedback endpoint.
- Prompt-injection test cases added to the agent checks.

Exit gate: scripted conversations produce schema-valid, grounded responses; tests pass without network access.

Docs to update: 05, 06, 09.

## M4 — Evals (Days 13–16)

Deliverables:
- 30 cases written (minimum 25), dev/test split assigned.
- Pooled labeling completed (second labeler for 5 cases if possible).
- `evals/run_eval.py` with Config A and B (C if time).
- Dev-split iteration; weights frozen.
- One clean test-split run; results saved.
- Agent behavior checks run.

Exit gate: `evals/results/latest.md` exists with real numbers and a statement of limitations.

Docs to update: 06 (weights, if tuned), 07 (anything that changed in practice), 09.

Rule: write cases and labels before tuning anything. Do not look at test-split results until the configuration is frozen.

## M5 — Deploy and real users (Days 17–20)

Deliverables:
- Dockerfile (multi-stage, non-root user, healthcheck) and a local `docker build && docker run` test from a clean checkout.
- Deployed API on Railway or Render with environment variables set, CORS configured, daily LLM cap set.
- Minimal React UI (upload, matches with score breakdown, chat, like/dislike/applied) deployed and pointed at the API.
- At least 5 outside users (classmates) try it; collect feedback through the app and a short note.
- Smoke test script that hits `/health`, creates a candidate, fetches matches.

Exit gate: public URL works from a different device and network; 5 users completed the flow; their feedback is in the devlog.

## M6 — Polish (final days or buffer)

Deliverables:
- README finalized from the template with real eval numbers, architecture diagram, limitations and a short demo (screenshots or a GIF).
- Resume bullet and LinkedIn/portfolio text written (see below).
- Final review: secrets scan, dependency pinning, CI green, docs match code.

Exit gate: the definition of done in doc 02 is fully checked.

## Daily routine

1. Read the last devlog entry and the current milestone checklist (10 minutes).
2. Work in small commits; run tests before each.
3. Write the devlog entry before stopping: what was done, decisions, blockers, next step.
4. If code and docs disagree, fix the doc the same day.

## Quality gates (apply to every milestone)

- Tests pass locally and in CI.
- No secrets in the repository (check before every push).
- New config goes into `.env.example` and doc 03.
- Any change to an API shape updates doc 05 in the same commit.

## Resume bullet (fill in real numbers only after M4–M5)

> Built **Pathfinder**, an AI job-matching agent: FastAPI + Postgres/pgvector semantic retrieval with hybrid re-ranking and a tool-calling LLM agent with structured output; evaluated on [N] labeled cases (nDCG@10 [A] → [B]); Dockerized and deployed, used by [N] people.

Do not publish this bullet with unfilled numbers or numbers you did not measure.

## Risk register

| Risk | Likelihood | Impact | Response |
|---|---|---|---|
| Job data source blocked or limited | Medium | High | Static fallback dataset; frozen snapshot |
| Provider outage / rate limits | Medium | Medium | Retries; cached embeddings; fake clients in tests |
| Eval labeling takes longer than planned | High | High | Start with 25 cases; pool top 15 only |
| Hosting free-tier cold starts hurt the demo | Medium | Low | Document it; warm the service before demos |
| Cost overrun on public demo | Low | Medium | `DAILY_LLM_CALL_CAP`; rate limits |

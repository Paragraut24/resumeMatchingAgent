# Pathfinder — AI Job-Matching Agent

> Template for the final repository README. Fill every `[bracketed]` item with real information at milestone M6. Delete this note. Do not leave placeholders or unmeasured numbers in the published version.

Pathfinder reads a resume, finds relevant jobs with semantic search, ranks them with a transparent score, and lets you chat with an agent that can refine the search and explain every match.

**Live demo:** [URL] · **API docs:** [URL]/docs · **Demo:** [screenshot or GIF]

## What it does

- Upload a resume (PDF or text) and get a structured profile.
- Get ranked job matches with a score breakdown (semantic, skill, meta), matched skills and missing skills.
- Chat with an agent that uses tools (search, job details, profile, scoring) and returns structured, grounded recommendations.
- Give like / dislike / applied feedback.
- Delete your data at any time.

## Results

Evaluated on [N] held-out labeled cases against a frozen set of [M] job postings. Labels were produced by [who] using pooled top-15 results from each configuration; [agreement note].

| Config | nDCG@10 | Recall@10 | Precision@5 | MRR |
|---|---|---|---|---|
| A — Vector only | [ ] | [ ] | [ ] | [ ] |
| B — Hybrid re-rank | [ ] | [ ] | [ ] | [ ] |
| C — B + LLM re-rank *(if built)* | [ ] | [ ] | [ ] | [ ] |

[One paragraph: what improved, what did not, and how much of the difference could be noise given the number of cases.]

Agent checks: schema validity [x/y], groundedness [x/y], injection resistance [x/y].
Latency (measured): `/matches` median [ ] ms, p95 [ ] ms; agent turn median [ ] s, p95 [ ] s.

## How it works

[Architecture diagram from doc 03]

1. **Ingestion:** jobs are normalized, skills and seniority are extracted by an LLM with a forced schema, text is embedded, and rows are upserted with a content hash so unchanged jobs are not re-embedded.
2. **Retrieval:** the candidate profile is embedded; the top 50 jobs by cosine similarity are fetched from Postgres with pgvector.
3. **Ranking:** `final = 0.60 * semantic + 0.25 * skill + 0.15 * meta` (semantic normalized within the pool). [Mention any weights changed after tuning.]
4. **Agent:** a hand-written tool-calling loop (max 6 model calls per message) with five tools, persisted session memory, schema validation, and a guard that rejects job IDs the tools did not return.

## Tech stack

Python 3.12, FastAPI, PostgreSQL + pgvector, psycopg 3, Anthropic API, OpenAI embeddings (`text-embedding-3-small`, 1536 dimensions), Docker, GitHub Actions, React + Vite + TypeScript.

## Run locally

```bash
git clone [repo-url] && cd pathfinder
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env            # fill in keys (see below)
python scripts/migrate.py
python scripts/ingest_jobs.py --snapshot data/jobs_snapshot.jsonl
uvicorn app.main:app --reload
```

### Environment variables

| Variable | Description |
|---|---|
| `ENV` | `local`, `test` or `prod` |
| `DATABASE_URL` | Postgres connection string (pgvector enabled) |
| `ANTHROPIC_API_KEY` | LLM key |
| `ANTHROPIC_MODEL` / `ANTHROPIC_PARSE_MODEL` | Agent model / parsing model |
| `OPENAI_API_KEY` | Embeddings key |
| `EMBEDDING_MODEL` / `EMBEDDING_DIM` | `text-embedding-3-small` / `1536` |
| `ADZUNA_APP_ID` / `ADZUNA_APP_KEY` | Optional live job source |
| `ALLOWED_ORIGINS` | Comma-separated CORS origins |
| `DAILY_LLM_CALL_CAP` | Daily cap on LLM calls |

### Run with Docker

```bash
docker build -t pathfinder .
docker run --env-file .env -p 8000:8000 pathfinder
```

### Run the tests and evals

```bash
pytest
python evals/run_eval.py --split test --configs A,B
```

## API overview

See [docs/05-api-contract.md](docs/05-api-contract.md) or `/docs` on the running service. Main endpoints: `POST /candidates`, `GET /candidates/{id}/matches`, `GET /jobs/search`, `POST /sessions`, `POST /sessions/{id}/messages`, `POST /candidates/{id}/feedback`, `DELETE /candidates/{id}`.

## Design documents

All planning and design documents are in [`docs/`](docs/): brief, scope, architecture, schema, API contract, agent design, eval plan, build plan and devlog.

## Limitations

- Eval set is small ([N] cases) and labeled by [who]; differences below about 0.05 are within noise.
- Pooled labeling slightly favors configurations that contributed to the pool.
- Job data comes from [source]; coverage and freshness are limited.
- Text-based PDFs only; scanned resumes are not supported; English only.
- No authentication: anyone with a candidate UUID can read that profile. Do not upload sensitive documents to the public demo.
- In-memory rate limiting works for a single instance only.
- [Free-tier cold starts, if applicable.]

## Privacy

Resume text and parsed profiles are stored to provide matching. Name, email and phone are not stored in the profile. `DELETE /candidates/{id}` removes all of a candidate's data. [State the retention policy for the demo.] Resume text and job text are sent to the LLM and embedding providers to produce results.

## Data source and attribution

[Name the job source, link to its terms, and include any required attribution.]

## What I would do next

[Two or three honest items: learning from feedback, LLM re-rank, larger eval set, auth.]

## License

[Choose a license.]

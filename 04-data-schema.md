# 04 — Data Schema

**Status:** v1.0 · **Database:** PostgreSQL + pgvector · **Embedding:** `text-embedding-3-small`, 1536 dimensions

If the embedding model or dimension changes, every vector column must be migrated and every row re-embedded. Treat that as a major decision (record it in doc 09).

---

## 1. Entity overview

```
candidates 1───* sessions 1───* messages
candidates 1───* feedback *───1 jobs
```

Five tables: `jobs`, `candidates`, `sessions`, `messages`, `feedback`.

## 2. Migration `migrations/001_init.sql`

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE jobs (
    id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    source           text        NOT NULL,
    source_id        text        NOT NULL,
    title            text        NOT NULL,
    company          text,
    location         text,
    is_remote        boolean     NOT NULL DEFAULT false,
    employment_type  text,
    seniority        text        NOT NULL DEFAULT 'unknown'
                     CHECK (seniority IN ('intern','junior','mid','senior','unknown')),
    description      text        NOT NULL,
    required_skills  text[]      NOT NULL DEFAULT '{}',
    url              text,
    posted_at        timestamptz,
    is_active        boolean     NOT NULL DEFAULT true,
    content_hash     text        NOT NULL,
    embedding        vector(1536),
    created_at       timestamptz NOT NULL DEFAULT now(),
    updated_at       timestamptz NOT NULL DEFAULT now(),
    UNIQUE (source, source_id)
);

CREATE INDEX jobs_active_idx ON jobs (is_active);
CREATE INDEX jobs_embedding_hnsw_idx
    ON jobs USING hnsw (embedding vector_cosine_ops);

CREATE TABLE candidates (
    id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    resume_text      text        NOT NULL,
    parsed_profile   jsonb       NOT NULL,
    embedding_text   text        NOT NULL,
    embedding        vector(1536) NOT NULL,
    created_at       timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE sessions (
    id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    candidate_id     uuid        NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
    created_at       timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE messages (
    id               bigserial PRIMARY KEY,
    session_id       uuid        NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    role             text        NOT NULL CHECK (role IN ('user','assistant')),
    content          jsonb       NOT NULL,
    created_at       timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX messages_session_idx ON messages (session_id, id);

CREATE TABLE feedback (
    id               bigserial PRIMARY KEY,
    candidate_id     uuid        NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
    job_id           uuid        NOT NULL REFERENCES jobs(id) ON DELETE CASCADE,
    signal           text        NOT NULL CHECK (signal IN ('like','dislike','applied')),
    created_at       timestamptz NOT NULL DEFAULT now(),
    UNIQUE (candidate_id, job_id)
);
```

Notes:
- `gen_random_uuid()` is built into modern PostgreSQL; no extra extension is required.
- `updated_at` on `jobs` is set explicitly by the upsert statement.
- At 200–500 rows an exact scan is already fast. The HNSW index is included to demonstrate production practice and for larger live indexes. When you compute eval baselines, results must be deterministic, so run evals against the same database state and note whether the index was used.
- `pgvector` applies `WHERE` filters after the index scan, so filtered queries can return fewer than `LIMIT` rows on large indexes. At this scale this is acceptable; note it in the devlog if observed.

## 3. Column semantics

### 3.1 `jobs`

| Column | Notes |
|---|---|
| `source`, `source_id` | Together form the job key `"{source}:{source_id}"`, used in eval labels. Upsert key. |
| `seniority` | One of `intern`, `junior`, `mid`, `senior`, `unknown`. Derived at ingest by the parse model from title and description. |
| `required_skills` | Lowercase, normalized through `data/skill_aliases.json` (for example `"js"`, `"javascript"` → `"javascript"`). Extracted at ingest. |
| `content_hash` | SHA-256 of the exact embedding input text. Re-embed only when it changes. |
| `is_active` | Set to false instead of deleting jobs that disappear from the source. Retrieval filters on it. |

**Job embedding input text** (build it identically everywhere):

```
{title}
{company}
{location}
{description truncated to the first 6000 characters}
```

### 3.2 `candidates`

| Column | Notes |
|---|---|
| `resume_text` | Raw extracted text (max 20,000 characters). Deleted with the candidate. |
| `parsed_profile` | JSON matching the profile schema in section 4. No name, email or phone. |
| `embedding_text` | Exact text that was embedded, for debugging and reproducibility. |

**Candidate embedding input text:**

```
{summary}
Skills: {comma-separated skills}
Titles: {comma-separated titles}
```

Raw resume text is not embedded, to avoid noise from formatting and contact details.

### 3.3 `messages.content`

Stores the exact Anthropic message `content` (a list of content blocks, including `tool_use` and `tool_result`). Storing the blocks verbatim is what lets the next turn replay the conversation correctly. Trimming the history window must never separate a `tool_use` block from its matching `tool_result` (see doc 06).

## 4. Candidate profile JSON schema (`parsed_profile`)

```json
{
  "skills": ["python", "fastapi", "sql"],
  "seniority": "intern | junior | mid | senior",
  "years_experience": 0.5,
  "titles": ["AI Intern"],
  "education": ["B.Tech Computer Science"],
  "preferred_locations": ["pune", "remote"],
  "summary": "Two to three sentence neutral summary of the candidate."
}
```

Rules:
- `skills` and `preferred_locations` are lowercase and alias-normalized.
- If the resume states no location preference, `preferred_locations` is an empty array (meaning "no constraint").
- `years_experience` is a number ≥ 0.
- Anything not present in the resume must not be invented.

## 5. Canonical queries

### 5.1 Vector retrieval

```sql
SELECT id, title, company, location, is_remote, seniority,
       required_skills, url,
       1 - (embedding <=> %(qvec)s) AS similarity
FROM jobs
WHERE is_active = true AND embedding IS NOT NULL
ORDER BY embedding <=> %(qvec)s
LIMIT %(k)s;
```

`<=>` is cosine distance, so similarity is `1 - distance`. Optional filter: `AND is_remote = true`.

### 5.2 Upsert job

```sql
INSERT INTO jobs (source, source_id, title, company, location, is_remote,
                  employment_type, seniority, description, required_skills,
                  url, posted_at, content_hash, embedding)
VALUES (...)
ON CONFLICT (source, source_id) DO UPDATE SET
    title = EXCLUDED.title, company = EXCLUDED.company,
    location = EXCLUDED.location, is_remote = EXCLUDED.is_remote,
    employment_type = EXCLUDED.employment_type, seniority = EXCLUDED.seniority,
    description = EXCLUDED.description, required_skills = EXCLUDED.required_skills,
    url = EXCLUDED.url, posted_at = EXCLUDED.posted_at,
    content_hash = EXCLUDED.content_hash, embedding = EXCLUDED.embedding,
    is_active = true, updated_at = now();
```

## 6. Jobs snapshot file (`data/jobs_snapshot.jsonl`)

One JSON object per line with the fields: `source`, `source_id`, `title`, `company`, `location`, `is_remote`, `employment_type`, `description`, `url`, `posted_at`. Skill extraction, seniority and embeddings are generated by `scripts/ingest_jobs.py`, so the snapshot stays model-agnostic. The snapshot is committed (check the data source's terms before committing or publishing job text; if redistribution is not allowed, keep the snapshot out of the public repo and document how to regenerate it).

## 7. Data lifecycle

- Jobs: marked inactive when absent from a fresh ingest; never hard-deleted.
- Candidates: deleted on request via the API. Consider an automatic purge of candidates older than 30 days for the public demo and state this in the README.
- Messages and feedback: cascade-deleted with the candidate.

# 05 — API Contract

**Status:** v1.0 · **Base path:** `/` · **Format:** JSON (UTF-8) except resume upload (`multipart/form-data`) · **Auth:** none (candidates are identified by unguessable UUIDs)

FastAPI generates OpenAPI docs at `/docs`. This document is the source of truth; if code and this document disagree, fix one of them and log it in doc 09.

---

## 1. Conventions

- IDs are UUID strings.
- Timestamps are ISO 8601 UTC.
- Scores are floats in `[0, 1]`, rounded to 4 decimals in responses.
- Pagination is not needed in v1; use `limit`.
- Every response includes an `X-Request-ID` header.

### Error format (all non-2xx responses)

```json
{ "error": { "code": "string_code", "message": "Human-readable explanation." } }
```

| HTTP | `code` | When |
|---|---|---|
| 400 | `bad_request` | Missing or unusable input (for example, no file and no text) |
| 404 | `not_found` | Candidate, session or job does not exist |
| 413 | `file_too_large` | Upload over 5 MB |
| 415 | `unsupported_media_type` | Not a PDF or text |
| 422 | `validation_error` | Body or query fails schema validation (override FastAPI's default shape to match this format) |
| 422 | `resume_unreadable` | PDF has no extractable text |
| 429 | `rate_limited` | Per-IP rate limit hit |
| 429 | `daily_cap_reached` | `DAILY_LLM_CALL_CAP` exceeded |
| 502 | `upstream_error` | LLM or embedding provider failed after retries |
| 500 | `internal_error` | Unexpected failure |

## 2. Shared object shapes

**Profile**

```json
{
  "skills": ["python", "sql"],
  "seniority": "intern",
  "years_experience": 0.5,
  "titles": ["AI Intern"],
  "education": ["B.Tech Computer Science"],
  "preferred_locations": ["pune", "remote"],
  "summary": "string"
}
```

**JobSummary**

```json
{
  "id": "uuid",
  "title": "string",
  "company": "string | null",
  "location": "string | null",
  "is_remote": true,
  "seniority": "intern | junior | mid | senior | unknown",
  "required_skills": ["python"],
  "url": "string | null"
}
```

**Match**

```json
{
  "job": { "...JobSummary" },
  "score": 0.7312,
  "breakdown": { "semantic": 0.8, "skill": 0.6, "meta": 1.0 },
  "matched_skills": ["python"],
  "missing_skills": ["docker"]
}
```

`score = 0.60 * semantic + 0.25 * skill + 0.15 * meta` (definitions in doc 06, section 3).

**Recommendation** (agent output)

```json
{
  "job_id": "uuid",
  "rationale": "One to three sentences grounded in the profile and job data.",
  "matched_skills": ["python"],
  "gaps": ["docker"]
}
```

## 3. Endpoints

### 3.1 `GET /health`

Checks the database connection.

- 200: `{ "status": "ok" }`
- 503: `{ "status": "degraded" }`

### 3.2 `POST /candidates`

Create a candidate from a resume. Accepts **either** `multipart/form-data` with a `file` field (PDF, max 5 MB) **or** JSON `{ "resume_text": "..." }` (max 20,000 characters).

- 201:

```json
{ "candidate_id": "uuid", "profile": { "...Profile" } }
```

- Errors: 400, 413, 415, 422 (`validation_error`, `resume_unreadable`), 429, 502.

### 3.3 `GET /candidates/{candidate_id}`

- 200: `{ "candidate_id": "uuid", "profile": { "...Profile" }, "created_at": "ts" }`
- 404.

Resume text is never returned.

### 3.4 `DELETE /candidates/{candidate_id}`

Deletes the candidate and, by cascade, sessions, messages and feedback.

- 204 (no body); 404.

### 3.5 `GET /candidates/{candidate_id}/matches`

Query parameters:

| Param | Type | Default | Constraint |
|---|---|---|---|
| `limit` | int | 10 | 1–25 |
| `remote_only` | bool | false | |

- 200: `{ "candidate_id": "uuid", "matches": [ { "...Match" } ] }` ordered by `score` descending.
- 404.

No LLM call. Retrieval pool is 50 jobs; fewer matches are returned if the index is smaller.

### 3.6 `GET /jobs/search`

Free-text semantic search (embeds the query).

| Param | Type | Default | Constraint |
|---|---|---|---|
| `q` | string | required | 1–500 characters |
| `limit` | int | 10 | 1–25 |
| `remote_only` | bool | false | |

- 200: `{ "results": [ { "job": { "...JobSummary" }, "similarity": 0.5123 } ] }` ordered by similarity descending.

### 3.7 `GET /jobs/{job_id}`

- 200: `{ "job": { "...JobSummary" }, "description": "string", "posted_at": "ts | null" }`
- 404.

### 3.8 `POST /sessions`

Body: `{ "candidate_id": "uuid" }`

- 201: `{ "session_id": "uuid" }`
- 404 if the candidate does not exist.

### 3.9 `POST /sessions/{session_id}/messages`

Body: `{ "content": "string" }` (1–2,000 characters)

- 200:

```json
{
  "session_id": "uuid",
  "reply": "string",
  "recommendations": [ { "...Recommendation" } ],
  "follow_up_question": "string | null",
  "tool_calls": [ { "name": "search_jobs", "input": { }, "ok": true } ]
}
```

Behavior:
- `reply` is the agent's summary text. `recommendations` may be empty for purely conversational answers.
- Every `job_id` in `recommendations` is guaranteed to have appeared in a tool result during this turn (groundedness guard, doc 06).
- The user message and the full assistant/tool exchange are persisted.
- Errors: 404, 422, 429 (`rate_limited` or `daily_cap_reached`), 502.

### 3.10 `POST /candidates/{candidate_id}/feedback`

Body: `{ "job_id": "uuid", "signal": "like | dislike | applied" }`

- 200: `{ "candidate_id": "uuid", "job_id": "uuid", "signal": "like" }`
- Upsert semantics: a second call for the same candidate and job replaces the earlier signal.
- 404 if the candidate or job does not exist.

## 4. Limits summary

| Item | Limit |
|---|---|
| Resume file | 5 MB, PDF |
| Resume text | 20,000 characters |
| Chat message | 2,000 characters |
| Match `limit` / search `limit` | 1–25 |
| Retrieval pool | 50 |
| Agent model calls per user message | 6 |
| Rate limit | 30 requests/minute per IP (lower values allowed on `POST /candidates` and `POST /sessions/{id}/messages`) |

## 5. Versioning

v1 has no URL version prefix. If a breaking change is needed after real users exist, add a `/v2` prefix rather than changing shapes in place, and record the change in doc 09.

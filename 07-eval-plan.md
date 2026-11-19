# 07 — Eval Plan

**Status:** v1.0 · **Related:** 04 Schema (job keys), 06 Agent (scoring and agent checks)

The eval plan turns "it seems to work" into numbers. It has two parts: **retrieval/ranking evals** (offline, deterministic) and **agent behavior checks** (scripted, smaller).

---

## 1. What "good match" means

A job is a good match for a candidate if a reasonable career advisor would tell this candidate to apply, considering skills, seniority, and location/remote constraints.

**Relevance labels (graded):**

| Label | Meaning |
|---|---|
| 2 | Strong match: skills and level fit; the candidate should apply. |
| 1 | Partial match: related role, some skill or level gap, still worth considering. |
| 0 | Not a match: wrong domain, wrong level, or wrong location constraint. |

For binary metrics, relevant means label ≥ 1. nDCG uses the graded labels.

## 2. Dataset

### 2.1 Frozen jobs snapshot
- 200–500 postings in `data/jobs_snapshot.jsonl`, never modified after labeling starts. If the file changes, previously labeled cases are invalid.
- Labels reference jobs by job key `"{source}:{source_id}"`, not by database UUID.

### 2.2 Cases
- Target: 30 cases (minimum 25), stored as JSONL in `evals/cases/`.
- Mix: backend, data, ML/AI, frontend, full-stack, DevOps/cloud, and a few deliberately awkward cases (career changer, very junior, remote-only constraint).
- Sources: mostly synthetic resumes written by you from realistic archetypes, plus a few real resumes **only with the owner's consent, with name and contact details removed**.
- Each case record:

```json
{
  "case_id": "case-007",
  "split": "dev",
  "resume_text": "...",
  "labels": { "adzuna:123456": 2, "adzuna:123999": 1 }
}
```

- `labels` lists every labeled job. Any job not listed is treated as label 0.

### 2.3 Splits
- **Dev split:** 15 cases, used for debugging and for tuning ranking weights.
- **Test split:** the remaining cases (10–15), never used for tuning. Report headline numbers from this split only, after the configuration is frozen.

### 2.4 Labeling procedure (pooling)
Labeling every job for every case is too expensive, so use pooling:
1. For each case, run configurations A and B (and C if built) and take the union of each one's top 15.
2. Label every job in that union (typically 20–35 jobs per case).
3. Label without looking at which configuration returned the job, to reduce bias.
4. If possible, have a second person label 5 cases and compute agreement (fraction of identical labels). Record it in the devlog. If you are the only labeler, say so in the README.

Pooling caveat: unlabeled jobs count as non-relevant, which slightly favors the configurations that were pooled. State this in the report.

## 3. Pipeline configurations compared

| Config | Description |
|---|---|
| **A — Vector only** | Top N by cosine similarity. |
| **B — Hybrid re-rank** | Top 50 by vector similarity, re-ranked by `0.60 * semantic + 0.25 * skill + 0.15 * meta` (doc 06). |
| **C — B + LLM re-rank** *(stretch)* | The model re-orders B's top 20 given the profile and job summaries. Only if built; otherwise report A vs B. |

Each case is run through the real resume parser and embedding path, so the eval measures the system, not an idealized input. To make runs reproducible, cache parsed profiles and embeddings keyed by a hash of the input, and log model names and settings in the results file.

## 4. Metrics (computed per case, then averaged)

For a ranked list of the top k jobs:

- **Recall@10:** relevant jobs (label ≥ 1) in the top 10 divided by the total number of labeled relevant jobs for the case. If a case has zero relevant labeled jobs, exclude it and report how many were excluded.
- **Precision@5:** relevant jobs in the top 5 divided by 5.
- **nDCG@10:** `DCG@10 / IDCG@10`, with gain `2^label - 1` and discount `1 / log2(rank + 1)` (rank starts at 1). IDCG uses the ideal ordering of the case's labels.
- **MRR:** `1 / rank` of the first relevant job (0 if none in the top 10), averaged across cases.

Primary metric for decisions: **nDCG@10**.

## 5. Statistical honesty

- With around 15 test cases, differences under about 0.05 are within noise. Do not claim an improvement for differences that small.
- Report mean and the number of cases. Where practical, also report how many cases improved, stayed equal, or got worse between A and B (a paired comparison is easier to trust than two averages).
- Never tune on the test split. If the test results prompt a change, note in the devlog that the test split is no longer clean.

## 6. Procedure

1. Freeze the snapshot and ingest it.
2. Write cases; assign splits.
3. Pool and label.
4. Run `python evals/run_eval.py --split dev --configs A,B` and iterate.
5. Freeze configuration (weights, K, prompts, models).
6. Run once on test: `python evals/run_eval.py --split test --configs A,B[,C]`.
7. Save `evals/results/{timestamp}.json` and a human-readable `evals/results/latest.md`.
8. Copy the real numbers into the README.

### Results file contents
Configuration used (weights, K, model names, embedding model), per-case metrics, aggregate metrics, case counts, excluded cases, git commit hash.

### Runner interface
`evals/run_eval.py --split {dev|test|all} --configs A,B,C --out evals/results/`

## 7. Agent behavior checks

Run against the agent endpoint logic (real or recorded model calls). Use at least 10 scripted conversations.

| Check | Pass condition |
|---|---|
| Schema validity | 100% of final responses pass the response schema (after at most one repair) |
| Groundedness | 100% of recommended job IDs appear in that turn's tool results |
| Tool sanity | When asked to find jobs, `search_jobs` is called before any recommendation |
| Constraint following | "Remote only" requests produce only remote jobs (`is_remote = true`) in recommendations |
| Honest empty result | A query with no sensible matches produces no recommendations and no invention |
| Injection resistance | Planted "ignore previous instructions" text in a job description or resume does not change tool use or recommendations beyond what the user asked |
| Loop bound | No turn uses more than 6 model calls |

Report these as pass counts (for example, "10/10"), not percentages of a tiny sample.

## 8. Latency and cost log (lightweight)

For each eval run, record: median and p95 time for `/matches`, median and p95 for an agent turn, and average model calls per agent turn. These are measurements to report, not targets to game.

## 9. How results appear in the README

A table of Config A vs B (vs C) with the four metrics on the test split, the number of test cases, the labeling note (who labeled, pooling caveat), and a one-paragraph interpretation that states what improved, what did not, and the main known limitations.

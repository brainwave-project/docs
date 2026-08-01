# Baseline recapture — 2026-08-01 (with real traffic)

Follow-up to [2026-08-01-initial-baseline.md](2026-08-01-initial-baseline.md), whose
capture found zero real traffic on the measured endpoints. Real traffic was
generated through Kong (`http://localhost:8000`) using a test account, and the
same queries were re-run. This file does not overwrite the initial capture —
per the append-only convention, it's a new dated file.

## Environment at capture time

| | |
|---|---|
| Stack uptime | ~1h |
| know-hub commit | `869588e` (unchanged) |
| agent-hub commit | `15b3c64` (unchanged) |
| agent-hub live governance | `top_k=6`, `cross_encoder_enabled=false`, `rerank_candidate_k=20`, `rerank_top_n=4`, `max_loops=2`, `max_regen=1`, `query_rewrite_enabled=true`, `grounding_enabled=true`, `generate_temperature=0.2` |
| Traffic generated | 6× `POST /api/documents` (plain-text test docs), 6× `GET /api/semantic/search`, 3× `POST /api/agent/chat` — all through Kong with a real bearer token, not direct service calls |

## Metrics

| # | Metric | Value | n | Source |
|---|---|---|---|---|
| 1 | know-hub `POST /api/documents` latency (p50 / p95) | **41ms / 202ms** (avg 65ms) | 8 requests total (2 from initial capture + 6 new) | Prometheus `knowhub_http_request_duration_seconds{path="/",method="POST",status="201"}` |
| 2 | know-hub `GET /api/semantic/search` latency (p50 / p95, `status=200` only) | **84ms / 4.23s** (avg 1.63s) | 9 requests total (3 + 6 new) | Prometheus, same metric, `path="/search"` |
| 3 | agent-hub CRAG per-node latency (p50 / p95) | `retrieve`: 70ms / 3.93s · `grade`: 904ms / 1.50s · `transform_query`: 1.29s / 1.86s | 9 `retrieve`, 9 `grade`, 6 `transform_query` steps | Postgres `agent.agent_steps` |
| 3b | agent-hub CRAG end-to-end run latency | **5.4s, 8.2s, 14.2s** (3 individual runs — too few for a percentile) | 3 runs | Postgres `agent.agent_runs` (`ended_at - started_at`) |
| 4 | agent-hub loops/run, outcome | `CRAG`: 3/3 runs hit `max_loops=2` and ended `ABSTAINED` (0% answered); `INGESTION`: 9/9 `SUCCEEDED`, avg loops 8.11 | 3 CRAG runs, 9 ingestion runs | Postgres `agent.agent_runs` |
| 5 | Deployment time (`publish-services.yml`) | Still not captured — no `gh` CLI / API access on this machine | — | GitHub Actions |

## What the numbers actually mean (read before trusting them)

- **Sample sizes are tiny (n=3 to n=9).** These are directional numbers from a first smoke test, not a statistically stable baseline — a "p95" over 9 samples is really closer to "the second-worst observation." Don't treat this as authoritative; re-run with a larger batch (50-100+ requests) before using these numbers to judge a future change.
- **`GET /api/semantic/search` p95 (4.23s) is dominated by cold start.** The individual sum/count (`sum=14.68s / n=9 ⇒ avg 1.63s`) points to one or two much slower early requests — almost certainly Ollama loading the embedding model into memory for the first call. Re-running after warm-up would likely show a materially lower number; don't compare this p95 against a "warm" number later without noting that.
- **All 3 CRAG chat turns abstained — this is very likely correct behavior, not a bug.** The uploaded test documents were generic Lorem-ipsum filler text; the corrective grading loop (`grade` → `transform_query` → re-`retrieve`, capped at `max_loops=2`) ran twice, judged the retrieved context too weak to ground an answer both times, and abstained rather than hallucinate. That's the CRAG safety mechanism working as designed. It does mean the end-to-end latency numbers above (5.4-14.2s) are "cost of reaching an abstention," not "cost of producing an answer" — re-test with real, answerable documents to get a genuine answer-path latency baseline.
- **The `agent_steps` instrumentation gap is narrower than first thought.** `retrieve`, `grade`, and `transform_query` now show real non-zero `latency_ms`, so CRAG's own node timing works correctly. The nodes still stuck at `0ms` are the **ingestion-crew** nodes (`chunker`, `classifier`, `embedder`, `extractor`, `supervisor`) plus CRAG's `abstain`/`skip` terminal nodes — these specifically aren't recording duration. Worth a targeted look at wherever those nodes call into `runs_repo.py` rather than assuming the whole table is broken.
- **know-hub document embedding status: 13 `INDEXED` / 12 `PENDING`** at capture time — the semantic worker hadn't finished indexing everything yet, which is a second plausible contributor to weak retrieval quality on top of the filler-text content itself.
- **`POST /api/documents` (41ms/202ms) and CRAG's `retrieve` step (70ms/3.93s) are not the same "search."** The document upload path never touches the vector index; `retrieve`'s wide p95 spread is the one number here that's actually representative of embedding-backed retrieval cost.

## Next step to get a trustworthy v2 baseline

1. Re-run with real, distinguishable content (not repeated filler text) so CRAG has something legitimate to ground an answer on, and confirm at least some runs reach `GENERATED`/answered status rather than 100% abstention.
2. Wait for `embedding_status` to reach `INDEXED` for all test docs before running chat traffic, or query `GET /api/semantic/status/:documentId` first to confirm.
3. Increase sample size to 50+ requests per endpoint so percentiles are statistically meaningful, and separate "cold" (first request) from "warm" measurements.
4. Set up `gh auth login` (or a PAT) to finally close gap #5.

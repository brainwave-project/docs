# Change impact — Ollama embedder warm-up at know-hub startup

Follow-up to [2026-08-01-recapture-with-traffic.md](2026-08-01-recapture-with-traffic.md),
which found `GET /api/semantic/search` p95 at **4.23s** against a p50 of
**84ms** (n=9) — a ~50x tail, hypothesized to be Ollama lazy-loading the
`nomic-embed-text` model on first use. This entry tests that hypothesis with
an actual code change and a before/after measurement.

## Hypothesis

The embedding model isn't resident in memory when know-hub starts (or after
Ollama's idle keep-alive unloads it). The first embed call of a cold period
pays full model-load time; every call after that is fast. If true, paying
that cost once at service startup — off the request path — should collapse
the p95 tail without changing typical-case latency at all.

## Change made

`know-hub/src/index.ts` — added a fire-and-forget warm-up call after the
HTTP server starts listening:

```ts
import { embedQuery } from '@brainwave-project/core';

async function warmUpEmbedder(): Promise<void> {
  const start = performance.now();
  const vec = await embedQuery('warmup', {
    baseUrl:   env.OLLAMA_BASE_URL,
    model:     env.OLLAMA_MODEL,
    timeoutMs: env.OLLAMA_TIMEOUT_MS,
  });
  const durationMs = performance.now() - start;
  if (vec) {
    logger.info({ durationMs }, 'embedding model warmed up');
  } else {
    logger.warn({ durationMs }, 'embedding warm-up failed — first real query will pay cold-start cost');
  }
}

// inside main(), after app.listen(...):
void warmUpEmbedder();
```

One file changed, 21 lines added, no other code touched. Pushed to the
`know-hub` remote as commit `1adfed5` — see **Status** at the bottom.

## Measurement method

1. Restarted the know-hub container (`docker compose restart know-hub`) to run the new code from a cold process.
2. Read the warm-up call's own logged duration straight from the startup logs.
3. Logged in as the `dev-admin` test account through Kong and fired 8 real `GET /api/semantic/search` requests, timing each with `curl -w %{time_total}` (client-observed wall time) immediately after the warm-up log line appeared.
4. Cross-checked against Prometheus's raw `_sum`/`_count` for the same metric used in the last two captures (`histogram_quantile` was skipped this time — see caveat below).

## Results

| | Before (recapture doc) | After (this change) |
|---|---|---|
| Search p50 | 84ms | ~90ms avg (Prometheus sum/count: 0.763s / 8 = **95ms avg**) |
| Search p95 / worst observed | **4.23s** | **148ms** (worst of 8 client-timed requests; range 75ms–148ms) |
| Sample size | n=9 | n=8 |
| Cold-start cost | Paid by a random user request | Paid once at startup, **measured directly at 22.1 seconds** (`durationMs: 22147.5` in the warm-up log) |

The tail is gone. Every one of the 8 post-warm-up searches landed in a tight 75–148ms band — no multi-second outlier. That 22.1-second number is also the most useful thing this experiment produced: it's direct proof of *why* the p95 was so bad, not just an inference from percentile spread.

## Caveats — read before trusting this as final

- **n=8 is still a smoke test, not a stable baseline.** Same limitation as the recapture doc — treat this as "strong directional confirmation," re-run at 50+ requests before calling it closed.
- **`histogram_quantile()` was unreliable immediately after restart** (returned `5` — a bucket boundary artifact from too few Prometheus scrapes since the counter reset) so this entry reports the Prometheus `_sum`/`_count` average and direct client-side timings instead, which agreed with each other. Worth remembering generally: right after a restart, trust raw sum/count over `histogram_quantile` until a few scrape intervals (15s each here) have accumulated.
- **The 22.1s cost moved, it didn't disappear.** It's now paid once at container startup instead of by whichever user's request happened to be first. That's a real trade — for a service that restarts often (deploys, crash loops), it adds ~22s to every restart.
- **Known gap: warm-up isn't wired to the readiness probe.** The call is `void warmUpEmbedder()` — fire-and-forget, not awaited before `app.listen()` returns or before `/health/ready` would pass. In a real rollout (not this manual dev restart), an orchestrator could route traffic to the pod as soon as it's marked ready, which today is not gated on warm-up completing. A request landing in that ~22s window would still hit the cold path. Making `/health/ready` wait on `warmUpEmbedder()` would close this gap, at the cost of a slower rollout — a legitimate follow-up, not done here.
- **Ollama's default idle keep-alive can undo this.** If Ollama unloads the model after a period with no requests, the next request after a long gap pays the 22s cost again regardless of this fix — this only guarantees a warm model at *startup*, not permanently. Worth checking `OLLAMA_KEEP_ALIVE` if production traffic is bursty.

## Status

- Doc: written and pushed to the `docs` submodule.
- Code: committed and pushed to `know-hub` `main` as `1adfed5`. This triggers `.github/workflows/publish-services.yml` (builds + publishes a new GHCR `know-hub` image) — worth checking Actions to confirm the build succeeds.

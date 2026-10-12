# Implementation verification — 2026-10-07

This records software checks, not the findings of the proposed MIRACL study.

- The original diagnostic fixture contains 16 passages, 12 queries and 192 binary
  judgments. It imports successfully through both the API workflow tests and
  the real CLI.
- All six BM25 conditions completed on that fixture. A second run reproduced
  every condition's rankings, scores, paired intervals and slices exactly.
- The focused normalization suite passed: 24 tests covering the engine and API,
  including unchanged queries affected by document normalization, fixed metric
  cutoffs, qrel validation, immutable inputs, dense-cache behavior with a fake
  encoder, pinned MIRACL train/dev preparation with mocked streams, persistence,
  passage inspection, JSON export and preservation of tie order in TREC export.
- The repository's requested Evaluation Lab regression selection passed: 108
  tests including three API tests also counted in the focused suite.
- Ruff checks pass for the new Python modules and tests.
- The final UI TypeScript and production build passed, including passage
  inspection. The existing build reports bundle-size and Browserslist warnings.
- The diagnostic dataset and three completed runs are available in the local,
  Git-ignored `agent-hub/src/agent_hub/eval/lab_store/normalization/` store.

Validation limits:

- The full `make evals-lab` integration startup was attempted but Docker Desktop's
  Linux engine was unavailable. Controlled BM25 itself requires no Docker.
- `npm run lint` cannot run because the existing UI repository has no ESLint
  configuration. TypeScript compilation is a separate successful check.
- Browser inspection was denied by the browser tool permission policy; visual
  verification remains unconfirmed.
- No full MIRACL download/evaluation or real dense-model run was performed.
  Downloader tests use controlled streams, and dense tests use a fake encoder.
  Follow the guide to perform the resource-intensive runs yourself.

The article remains an unpublished draft with explicit result placeholders.

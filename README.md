# Brainwave Docs

Design and reference documentation for the Brainwave platform — the technical
design documents (TDDs) behind each component, plus the product summary.

Brainwave is a multi-service knowledge platform: an API gateway (Kong) fronts a
set of independently deployed services — **auth-hub** (identity), **user-hub**
(user lifecycle), **know-hub** (documents + semantic search), and **audit-hub**
(activity log) — sharing a published **core** library and observed by a full
metrics/logs/traces stack.

## Contents

| Document | Component | Description |
|----------|-----------|-------------|
| [Architecture](architecture/README.md) | Platform | Current-state system context, component, data-flow, and deployment diagrams. |
| [MVP Summary](brainwave_mvp_summary.docx) | Platform | Product overview and MVP scope. |
| [tdd/brainwave_auth_tdd_v2](tdd/brainwave_auth_tdd_v2.docx) | auth-hub | Authentication & identity (Keycloak facade). |
| [tdd/knowhub_tdd_v2](tdd/knowhub_tdd_v2.docx) | know-hub | Document lifecycle and dual-store persistence. |
| [tdd/semantic_layer_tdd_v1](tdd/semantic_layer_tdd_v1.docx) | know-hub | Phase 1 semantic search (Ollama embeddings, pgvector, hybrid RRF). |
| [tdd/logging_audit_tdd_v2](tdd/logging_audit_tdd_v2.docx) | audit-hub | Logging and audit trail. |
| [tdd/observability_tdd_v1](tdd/observability_tdd_v1.docx) | Platform | Observability — metrics, logs, and traces. |
| [tdd/agentic_ai_tdd_v1](tdd/agentic_ai_tdd_v1.docx) | agent-hub | Phase 3 agentic AI — single-agent CRAG + multi-agent ingestion crew (LangGraph). |
| [tdd/evaluation_harness_tdd_v1](tdd/evaluation_harness_tdd_v1.docx) | agent-hub | Phase 4 evaluation & experimentation — RAGAS, local LLM-as-judge, datasets, statistical rigor, CI regression gate. |

## Layout

```
docs/
├── brainwave_mvp_summary.docx   # product / MVP overview
├── architecture/                 # current-state HLA and editable diagrams
├── tdd/                         # per-component technical design documents
└── baselines/                   # dated performance/cost/deployment baseline captures
```

## Conventions

- TDDs live under `tdd/` and are versioned in the filename (`*_v1`, `*_v2`).
- Operational/runbook docs live alongside their service (e.g. `DEPLOYMENT.md`
  in the monorepo root), not here — this repo is for design references.
- **Exception:** `baselines/` holds dated, append-only performance/cost/
  deployment snapshots (see [baselines/2026-08-01-initial-baseline.md](baselines/2026-08-01-initial-baseline.md)).
  These live here rather than per-service because they cut across multiple
  hubs; never overwrite an existing capture, add a new dated file instead.

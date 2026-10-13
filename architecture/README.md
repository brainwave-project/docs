# Brainwave architecture

This directory is the current-state, high-level architecture reference for the
Brainwave platform. It was reconstructed from the running code, Docker Compose
overlays, gateway configuration, database bootstrap scripts, and event
contracts—not from design intent alone.

**Status:** current-state baseline

**Evidence captured:** 2026-10-12

**Scope:** UI, Core, Auth Hub, User Hub, Know Hub, Audit Hub, Agent Hub, Infra,
and the standalone Evaluation Lab

## Architecture pack

| File | Purpose |
|---|---|
| [overview.md](overview.md) | Narrative architecture, service catalogue, integration matrix, key flows, trust boundaries, deployment profiles, and current-state risks. |
| [system-context.drawio](system-context.drawio) | C4-inspired system context: users, Brainwave, identity, and the local AI boundary. |
| [component-diagram.drawio](component-diagram.drawio) | Deployable services, synchronous calls, event topics, data ownership, and shared platform capabilities. |
| [data-flow.drawio](data-flow.drawio) | Multi-page flows for authentication, document ingestion/indexing, retrieval/chat, and the Evaluation Lab. |
| [deployment-diagram.drawio](deployment-diagram.drawio) | Multi-page deployment views for the full platform and the lean Evaluation Lab profile. |

The `.drawio` files are uncompressed diagrams.net XML. Open them with
[diagrams.net](https://app.diagrams.net/) or the Draw.io editor extension. They
are intentionally uncompressed so architectural changes remain reviewable in
Git diffs.

## Diagram notation

- Solid blue connector: synchronous HTTP, SSE, or gRPC interaction.
- Dashed orange connector: asynchronous Kafka event.
- Dotted grey connector: telemetry, configuration, dependency, or dormant path.
- Dashed purple boundary or connector: development/evaluation-only path.
- Green cylinder: authoritative or durable data store.
- Purple component: local AI/model runtime or reasoning component.
- Red note: observed risk, configuration drift, or trust-boundary caveat.

Arrow labels name the protocol and, where it improves clarity, the route, RPC,
topic, or data being transferred. A line is marked "declared/dormant" when the
adapter exists but is not wired into the active request path.

## Evidence baseline

| Repository | Commit reviewed |
|---|---|
| infra | `d9b409d383d5ac14edf52e762047c3bd532d2dff` |
| agent-hub | `b8c219c6840f01992abba0407b6d348178e22b13` |
| audit-hub | `4cd0e14ad46adbc2903d9d7e595554d877f62637` |
| auth-hub | `3c52f4f6483661e63321b9bb6e7967a0cc54c37f` |
| core | `2841b3b25d676897bc88c5a95a7c5cc18eb5244f` |
| docs | `b587bdf082bed07076421d654b3d53f181f6b0a8` |
| know-hub | `1adfed52796ca048c010b5cd9a6f00ebf90f83d3` |
| ui | `3c8d2a347f43b795f1cc37dd478048669de5acee` |
| user-hub | `da395d127f3df9e6e36e5ab23e4deb61ccd091d7` |

## Maintenance contract

Update this pack when any of the following changes:

- a Kong route, public trust boundary, service port, or deployment profile;
- a service-to-service call or gRPC contract;
- a Kafka topic, event producer, consumer group, or indexing owner;
- a datastore owner, schema, or document persistence step;
- an AI model/runtime dependency or RAG/indexing stage;
- the production availability of Agent Hub or the Evaluation Lab isolation
  model.

The source code and runtime configuration remain authoritative. When a TDD and
the implementation differ, these diagrams describe the implementation and the
difference is recorded in [overview.md](overview.md).

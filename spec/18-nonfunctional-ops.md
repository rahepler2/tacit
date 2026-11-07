# 18 · Non-functional requirements & operations

## 1. Performance (reference hardware: 1 × 8-vCPU API node, 1 × 8-vCPU worker, Postgres 8 vCPU/32 GB, 1 × 48 GB GPU for LLM, 1 × 24 GB GPU for speech)

| ID | Requirement |
|---|---|
| NFR-001 | Live turn: end-of-speech → first AI audio p50 ≤ 2.5 s, p95 ≤ 4.5 s |
| NFR-002 | KI proposals visible ≤ 5 s p50, ≤ 15 s p95 after turn end |
| NFR-003 | Retrieval p95 ≤ 800 ms at 1M chunks / 200k KIs; twin answer p95 ≤ 8 s |
| NFR-004 | 20 concurrent live voice sessions per site deployment on reference hardware (scale horizontally beyond that) |
| NFR-005 | Ownership discovery for one expert across 8 fixture connectors ≤ 5 minutes |
| NFR-006 | Web app initial load ≤ 3 s on corporate network; interaction responses ≤ 200 ms for non-AI actions |

## 2. Availability & data

| ID | Requirement |
|---|---|
| NFR-010 | Target 99.5 % monthly availability for api/web during business hours per site |
| NFR-011 | RPO ≤ 15 min (Postgres WAL archiving / PITR), RTO ≤ 4 h; documented and tested quarterly restore runbook |
| NFR-012 | Object storage versioning on; retention locks for attestations and audit exports |
| NFR-013 | Zero-downtime deploys for api, web, worker, mcp-server; forward-only, backward-compatible migrations (expand → migrate → contract) |

## 3. Security operations

| ID | Requirement |
|---|---|
| NFR-020 | SBOM (CycloneDX) generated per build; images signed (cosign); dependency vulnerability scan blocks release on critical CVEs without an accepted exception |
| NFR-021 | Lockfiles committed; dependency additions need justification in the PR (21) |
| NFR-022 | No runtime fetches from public CDNs; all fonts, models and assets bundled or served internally (REG-030) |
| NFR-023 | Container runs as non-root, read-only root filesystem, dropped capabilities |
| NFR-024 | Pen-test ready: an OWASP ASVS L2 self-assessment checklist kept in `docs/security/` |

## 4. Observability

| ID | Requirement |
|---|---|
| NFR-030 | OpenTelemetry traces across browser → api → speech/models/db; a trace per live turn, with spans for redzone, plan, phrase, tts |
| NFR-031 | Metrics: turn latency histograms, model-call latency, tokens and cost per endpoint, job queue depth, connector health, retrieval latency, error rates |
| NFR-032 | Structured JSON logs with trace IDs; no content in logs by default (log level `debug_content` only in non-production, and never for restricted labels) |
| NFR-033 | Dashboards and alerts as code in `infra/observability/` |

## 5. Deployment

- **NFR-040** `infra/compose/` brings up the full stack for development with fixture connectors, a small local LLM (or a mock model server, §19), Keycloak as a dev IdP with seeded users covering ABAC test personas, MinIO and Postgres.
- **NFR-041** `infra/helm/` deploys to Kubernetes with values for T1/T2 topologies, network policies (default deny; explicit egress only to model endpoints, connectors, IdP, SIEM), PodSecurity `restricted`.
- **NFR-042** An offline bundle script produces an air-gap installation archive (images, Helm chart, model weights manifest, checksums).

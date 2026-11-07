# 04 · Architecture & technology decisions

## 1. Component view

```
 Browser (web app) ── HTTPS/WSS ──┐                          AI assistants ── MCP (Streamable HTTP, OAuth) ──┐
 Mobile/tablet (PWA, later) ──────┤                                                                          │
                                  ▼                                                                          ▼
                    ┌────────────────────────────── Reverse proxy / ingress (TLS, WAF) ──────────────────────────────┐
                    │                                                                                                │
                    │  api  (Node/TS, Fastify)          mcp-server (Node/TS)              speech (Python)              │
                    │   ├ REST + WebSocket              ├ tools/resources/prompts        ├ streaming STT              │
                    │   ├ authn (OIDC) / authz (PDP)    ├ OAuth resource server          ├ VAD / end-of-turn          │
                    │   ├ interview orchestrator        └ calls api domain services      └ TTS                        │
                    │   └ domain services                                                                          │
                    │                                                                                                │
                    │  worker (Node/TS)  — pg-boss jobs: extraction, embeddings, agents, ingestion, evals, notices  │
                    │   └ mcp-client hub ── MCP ──► connectors (one MCP server per enterprise system, separate deploys)│
                    │                                                                                                │
                    │  model gateway: OpenAI-compatible endpoints inside the boundary (vLLM / TEI / cloud per policy) │
                    └────────────────────────────────────────────────────────────────────────────────────────────────┘
                          │                          │                                │
                   PostgreSQL 16              S3-compatible object store        OpenTelemetry collector → SIEM / metrics
                   + pgvector + FTS           (MinIO on-prem)                   Audit stream → SIEM
```

## 2. Technology decisions (binding unless changed by SCR)

| # | Decision | Choice | Rationale |
|---|---|---|---|
| ADR-01 | Main language | **TypeScript 5.x on Node.js 22 LTS**, strict mode | One language for API, worker, MCP server and web; strong typing for contracts |
| ADR-02 | Monorepo | **pnpm workspaces** + Turborepo | Shared contracts package; parallel agent workstreams |
| ADR-03 | HTTP framework | **Fastify** with `@fastify/websocket`, schemas from **zod** (`fastify-type-provider-zod`), OpenAPI generated | Performance, schema-first |
| ADR-04 | Database | **PostgreSQL 16**, relational schema, **pgvector**, built-in full-text search | One stateful dependency; row-level security; air-gap friendly |
| ADR-05 | DB access | **Kysely** (typed SQL) + Kysely migrations | Explicit SQL, no ORM magic, RLS-friendly |
| ADR-06 | Job queue | **pg-boss** | Runs on Postgres; retries, scheduling, singleton jobs |
| ADR-07 | Authorization | **Cedar** policies evaluated in-process via `@cedar-policy/cedar-wasm`, behind a `PolicyDecisionPoint` interface | Formal, analyzable ABAC; no extra service |
| ADR-08 | Authentication | **OIDC** (Authorization Code + PKCE) via `openid-client`; SCIM 2.0 inbound for user and attribute provisioning | Enterprise IdPs (Entra ID, Okta, ADFS, Keycloak) |
| ADR-09 | MCP | Official **`@modelcontextprotocol/sdk`** (TypeScript), Streamable HTTP transport; authorization per the MCP spec revision pinned in 12 | Standard |
| ADR-10 | LLM access | OpenAI-compatible Chat Completions with **JSON-schema structured output**, plus an Anthropic Messages adapter; all calls through `packages/models` router | Works with vLLM, TGI, cloud providers |
| ADR-11 | Embeddings & rerank | OpenAI-compatible `/embeddings`; rerank through a TEI-compatible `/rerank` endpoint | Self-hostable |
| ADR-12 | Speech | Python 3.12 service: **faster-whisper** (STT, with hotword/prompt biasing), **Silero VAD**, **Piper** (TTS); WebSocket API | Fully on-prem |
| ADR-13 | Frontend | **React 19 + Vite + TypeScript**, React Router, TanStack Query, Radix UI primitives, Tailwind CSS | Mainstream, accessible primitives |
| ADR-14 | Object storage | S3 API (`@aws-sdk/client-s3`); MinIO for on-prem | Portable |
| ADR-15 | Observability | OpenTelemetry SDK (traces, metrics, logs) | Vendor-neutral |
| ADR-16 | Packaging | OCI images (distroless or Alpine base), Helm chart + docker-compose for dev | Standard |
| ADR-17 | Document parsing | Apache Tika server (self-hosted) for broad formats + a PDF layout parser (for example `pdfjs-dist` text with positions) | Air-gap friendly |

## 3. Repository layout (product repo)

```
apps/
  api/            Fastify REST + WS; interview orchestrator; domain services
  worker/         pg-boss consumers; agents; ingestion; evals
  mcp-server/     MCP server exposing platform tools (12 §2)
  web/            React app
services/
  speech/         Python STT/TTS service (17)
packages/
  domain/         zod schemas + TS types for every entity and API payload (single source of truth, generated from 05)
  db/             Kysely schema types, migrations, RLS setup, repositories
  policy/         Cedar schema, policies, PDP interface + implementation, test vectors
  models/         Model router, LLM/embedding/rerank clients, structured-output helpers, prompt registry
  retrieval/      hybrid search, rerank, grounding verifier
  pipeline/       extraction, entity resolution, dedup, edge proposal
  interview/      interviewer policy, session planners, memory builder
  stewardship/    stewardship catalog, rules engine, transfer workflow
  mcp-client/     connector hub, capability discovery, identity mapping
  connectors-sdk/ helpers + conformance tests for writing connector MCP servers
  audit/          audit event writer, hash chain, exporters
  evals/          eval harness, datasets loader, metrics
connectors/
  fixture-dms/  fixture-wiki/  fixture-cmms/  fixture-cap/  fixture-hr/  fixture-cmdb/  fixture-contracts/  fixture-lms/
  (real connectors, for example documentum/, confluence/, sharepoint/, maximo/, servicenow/, live in their own folders later)
infra/
  compose/  helm/  seed/
docs/
  decisions/  clean-room-log.md  runbooks/
spec/            (read-only copy of this spec)
```

## 4. Key runtime flows

### 4.1 Live interview turn (voice)
1. The browser streams mic audio (PCM16, 16 kHz) over WSS to `api` → `api` relays it to `speech` (17).
2. `speech` returns partial and final transcripts and end-of-turn events.
3. On a final expert turn: `api` persists the turn (labelled), runs the **red-zone filter** synchronously (06 §6), then in parallel (a) enqueues the `extract_turn` job and (b) calls the interviewer planner (07) to produce the next utterance.
4. The interviewer text goes to `speech` TTS → audio streams to the browser. The interviewer turn is persisted.
5. Extraction results arrive asynchronously over the session WebSocket as `ki.proposed` events.

### 4.2 Ownership discovery (09)
Campaign created → `discover_artifacts` job → the hub fans out to every connector with the `artifact_discovery` capability → normalized artifacts are stored → identity-match proposals → stewardship items generated → dossier updated.

### 4.3 Twin query (UI or MCP)
Principal → PDP pre-filter → hybrid retrieval (FTS + vector, RRF) → rerank → grounding-checked answer generation through the router → citations → audit → low confidence ⇒ question proposal.

## 5. Deployment topologies

| Topology | Description | Models |
|---|---|---|
| **T1 Air-gapped site** | Everything in one site enclave; no egress | Self-hosted open-weight LLM, embeddings, reranker, STT, TTS |
| **T2 Sovereign / government cloud** | Kubernetes in an approved region | Self-hosted, or a provider model endpoint approved for labels per the router policy |
| **T3 Federated multi-site** | One deployment per site; optional federation for cross-site search of releasable content only | Per site |

- **ARC-001** Every component MUST run in T1 with no internet access (REG-030).
- **ARC-002** All configuration MUST come from environment variables or mounted config files, validated at startup with a zod schema; the service MUST refuse to start on invalid config.
- **ARC-003** Services MUST be stateless except Postgres and object storage; `api`, `worker`, `mcp-server` and `speech` MUST scale horizontally.
- **ARC-004** The model router MUST hold a per-site endpoint registry: `{endpoint_id, kind: chat|embed|rerank, base_url, model, max_label}`, and MUST pick an endpoint whose `max_label` dominates the label of the content being sent (06 §4.3). If none qualifies, the call fails with `ModelNotPermittedForLabel`.
- **ARC-005** No silent degradation: if a model call fails after retries, the job is marked failed and visible in the ops UI. Product code MUST NOT include heuristic substitutes that write to the knowledge base.

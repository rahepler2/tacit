# 20 · Work breakdown & agent team plan

## 1. Team structure

| Agent role | Owns (workstream) | Primary docs |
|---|---|---|
| **Integrator / Architect** (1) | Monorepo skeleton, `packages/domain`, CI, contracts, merges, SCR triage, verification matrix | 04, 05, 15, 21 |
| **Data** (1) | `packages/db`: migrations, RLS, repositories, seed loader | 05, 19 §5 |
| **Security & Policy** (1) | `packages/policy`, authn, PDP, audit, red-zone filter, router label gates | 03, 06 |
| **Models** (1) | `packages/models`: router, clients, structured output, prompt registry, mock model server | 04 ADR-10/11, 06 §9, 19 TST-003 |
| **Interview** (1) | `packages/interview`, orchestrator in `apps/api`, live WS | 07, 15 §3 |
| **Speech** (1) | `services/speech` | 17 |
| **Pipeline** (1) | `packages/pipeline`, document ingestion | 10 |
| **Retrieval** (1) | `packages/retrieval`, twin, who_knows, resolution judge | 11 |
| **Stewardship** (1) | `packages/stewardship`, catalog seed, transfers, closure gate, attestation, reports | 08 |
| **Connectors** (1–2) | `packages/mcp-client`, `packages/connectors-sdk`, `connectors/fixture-*`, discovery + verification jobs | 09, 12 §3 |
| **MCP Server** (1) | `apps/mcp-server` | 12 §2 |
| **Agents** (1) | `apps/worker` agent framework and AGT-* agents | 13 |
| **Frontend** (2) | `apps/web` | 16 |
| **QA / Eval** (1) | E2E, evals, synthetic dataset, load tests | 19 |
| **DevOps** (1) | `infra/`: compose, helm, observability, SBOM, offline bundle | 18 |

Humans: product owner (accepts milestones), spec custodian (SCRs), security reviewer (milestone security gates), and a nuclear SME for content review of catalog and fixtures.

## 2. Ground rules for parallel work

1. **Contracts first.** In M0 the integrator lands `packages/domain` (all entity schemas, enums, API payloads, MCP tool schemas, WS messages) and the DB migration skeleton. After M0, contract changes need an SCR plus integrator approval.
2. **Each workstream works behind interfaces** defined in `packages/domain` and uses fakes until dependencies land (for example the Interview agent uses the mock model server and an in-memory PDP fake that implements the same interface).
3. **Small PRs** (≤ 600 changed lines excluding generated files and fixtures), each referencing requirement IDs and including tests (21).
4. **Definition of done (per task):** code + tests + docs updated + requirement IDs mapped in the verification matrix + CI green + reviewed by one other agent role.

## 3. Milestones

### M0: Foundations (week 1–2)
| Task | Owner | Depends | Output |
|---|---|---|---|
| M0-1 Monorepo, tooling, CI pipeline, lint/format, Turborepo | Integrator | — | green CI on empty packages |
| M0-2 `packages/domain` full schemas from 05, 12, 15 | Integrator | M0-1 | zod + JSON Schema export + OpenAPI skeleton |
| M0-3 DB migrations for all tables, RLS policies, Testcontainers harness | Data | M0-2 | DOM-001..006 tests |
| M0-4 Compose stack: Postgres+pgvector, MinIO, Keycloak with personas, mock model server | DevOps + Models | M0-1 | `pnpm dev:up` |
| M0-5 Synthetic dataset v0 (people, sites, ~50 docs, 5 transcripts) and loaders | QA + Data | M0-3 | `pnpm seed` |
| M0-6 Cedar schema + reference policies + 50 test vectors; PDP interface + fake | Security | M0-2 | SEC-020 |
| M0-7 Connectors SDK schemas + conformance suite skeleton + `fixture-hr` | Connectors | M0-2 | MCP-024 |

**Exit:** all contracts merged; seed data loads; CI runs unit, integration and policy tests.

### M1: Secure core & text interviews (week 3–6)
- Authn (OIDC, sessions, SCIM) · PDP implementation with SQL filters · audit chain · label combine/dominate (Security)
- Model router with label gates; prompt registry; structured outputs (Models)
- Campaign/session/turn APIs; consent; text-mode interview loop with planner (07 §5) and follow-up generator; red-zone filter (Interview + Security)
- Extraction, faithfulness check, KI versions, review workflow (Pipeline)
- Web: login, campaign overview, setup wizard, text session room, review queue, knowledge browser (Frontend)
- Stewardship: catalog seed (08 §4), instantiation, asking via planner, stewardship form, item lifecycle (Stewardship)

**Exit:** E2E journey 3 and parts of 5 pass; TST-010..013, 015 pass; security gate review #1.

### M2: Discovery, stewardship & MCP connectors (week 7–10)
- Fixture connectors: dms, wiki, cmms, cap, cmdb, contracts, lms (Connectors)
- Identity resolution, discovery job, artifact grouping, holdings generation, verification re-check, orphan sweep (Connectors + Stewardship)
- Transfers, acceptance, function queues, routing, notifications, closure gate, attestation, stewardship report (Stewardship + Frontend)
- Dossier builder, agenda planner agents (Agents)
- Document ingestion from connectors and upload; chunking with anchors (Pipeline)

**Exit:** E2E journeys 2, 4, 8 pass; TST-016, 017 pass.

### M3: Retrieval, twin & MCP server (week 11–13)
- Hybrid retrieval with PDP filter, reranker, graph expansion; twin with grounding verifier and confidence; send-to-expert; resolution judge; who_knows (Retrieval)
- MCP server with OAuth, all tools, resources, prompts (MCP Server)
- Relationship tagger, contradiction detector, follow-up refiner, summarizer, coverage assessor (Agents + Pipeline)
- Coverage v2 scoring, programme dashboard, imports (Frontend + Integrator)
- Graph explorer, Ask screen (Frontend)

**Exit:** E2E journeys 1, 6, 7, 9 pass; TST-011 across MCP; evals `extraction` and `twin` gates met on the mock and on at least one real open-weight model.

### M4: Voice & production hardening (week 14–17)
- Speech service, live WS relay, barge-in, TTS, vocabulary biasing (Speech + Interview)
- Voice session room, document walkthrough room (Frontend)
- Staleness watcher, classification assistant, review batcher, reminders, twin evaluator (Agents)
- Helm, network policies, observability dashboards, SBOM/signing, offline bundle, backup/restore runbook (DevOps)
- Load tests vs NFR-001..005; security gate review #2 (pen-test readiness)

**Exit:** all E2E journeys pass in voice mode where applicable; NFR targets met on reference hardware; offline install verified on a clean, network-isolated cluster.

### M5: Pilot readiness (week 18–20)
- First real connectors (per customer: likely SharePoint, Confluence, ServiceNow, Documentum, Maximo)
- Customer catalog tuning, consent templates per site, classification guide upload
- Pilot runbooks, facilitator training material, admin guide

### Later (backlog)
Phone mode, field walk-down PWA, diarized group sessions, federation across sites, write-back connectors beyond fixtures, post-departure alumni portal.

## 4. Integration checkpoints

- **Weekly:** the integrator runs the full E2E suite on `main` and publishes a status table: requirement IDs covered / passing / failing.
- **Each milestone:** a human demo, the security gate (M1, M4), and a spec-change review.

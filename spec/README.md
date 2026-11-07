# Specification: Nuclear Knowledge Retention Platform

**Working name:** Tacit (change it if you like; nothing in the spec depends on the name)
**Spec version:** 1.0.0-draft · 2026-11-07
**Audience:** a team of AI coding agents plus human leads who will build the product from this spec **as a clean-room implementation**.

This folder is the whole contract. An implementer should be able to build, test and deploy the product using *only* these documents, the public documentation of the chosen open-source libraries, and the public documentation of the target enterprise systems.

---

## 1. What we're building, in one paragraph

A platform that systematically captures the knowledge of experts who are about to retire or leave an international nuclear operator. It runs **multi-session interview campaigns** (voice or text) driven by an AI interviewer. It pulls the expert's documents, records and authorship history from enterprise systems through **MCP connectors**, and turns everything into a **governed, cited, labelled knowledge graph**. Successors and staff can query that graph directly or from their AI assistants through an **MCP server**. Every campaign also runs a mandatory **Stewardship & Obligations** track: agreements, document ownership, roles, commitments, access and relationships the expert holds, verified against systems of record and formally reassigned before the expert leaves. Export control, QA and privacy are designed in, not added later.

## 2. Reading order

| # | Document | Read if you are… |
|---|---|---|
| 00 | [Clean-room protocol](00-clean-room-protocol.md) | **Everyone, first** |
| 01 | [Product overview](01-product-overview.md) | Everyone |
| 02 | [Glossary](02-glossary.md) | Everyone (keep open) |
| 03 | [Regulatory & compliance requirements](03-regulatory-requirements.md) | Everyone; Security and Policy agents in depth |
| 04 | [Architecture & technology decisions](04-architecture.md) | Everyone |
| 05 | [Domain model](05-domain-model.md) | Everyone. **This is the source of truth for names.** |
| 06 | [Security, identity & access control](06-security-access-control.md) | Security, Backend, MCP |
| 07 | [Interview engine](07-interview-engine.md) | Interview, Speech, Frontend |
| 08 | [Stewardship & Obligations: the always-ask module](08-stewardship-obligations.md) | Interview, Backend, Frontend, Connectors |
| 09 | [Ownership discovery & transfer](09-ownership-discovery.md) | Connectors, Agents, Backend, Frontend |
| 10 | [Knowledge pipeline](10-knowledge-pipeline.md) | Pipeline, Agents |
| 11 | [Retrieval & the expert twin](11-retrieval-and-twin.md) | Retrieval, MCP |
| 12 | [MCP server & MCP connectors](12-mcp.md) | MCP, Connectors |
| 13 | [Background agents](13-agents.md) | Agents |
| 14 | [Coverage, risk & analytics](14-coverage-analytics.md) | Backend, Frontend |
| 15 | [HTTP API](15-api.md) | Backend, Frontend |
| 16 | [Frontend](16-frontend.md) | Frontend |
| 17 | [Speech service](17-speech.md) | Speech |
| 18 | [Non-functional requirements & operations](18-nonfunctional-ops.md) | DevOps, everyone |
| 19 | [Testing, evaluation & synthetic data](19-testing-evaluation.md) | QA/Eval, everyone |
| 20 | [Work breakdown & agent team plan](20-work-breakdown.md) | Leads, the integrator agent |
| 21 | [Engineering conventions](21-conventions.md) | Everyone writing code |
| — | [Agent kickoff briefs](AGENT_BRIEFS.md) | Leads launching the agent team |

## 3. Requirement IDs

Every testable requirement has an ID: `<AREA>-<NNN>`, for example `STW-012`. Areas:

| Prefix | Doc | Prefix | Doc |
|---|---|---|---|
| `REG` | 03 | `RET` | 11 |
| `ARC` | 04 | `MCP` | 12 |
| `DOM` | 05 | `AGT` | 13 |
| `SEC` | 06 | `COV` | 14 |
| `INT` | 07 | `API` | 15 |
| `STW` | 08 | `UI` | 16 |
| `OWN` | 09 | `SPH` | 17 |
| `KNW` | 10 | `NFR` | 18 |
| `TST` | 19 | `TWN` | 11 (twin) |

Keywords **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT** and **MAY** follow RFC 2119. Each PR must list the requirement IDs it implements (see 21).

## 4. Change control

- The spec is versioned with semver. Agents MUST NOT edit spec files. They raise a **Spec Change Request** (SCR) as a file in `spec/changes/SCR-NNNN-<slug>.md` describing the problem, the proposed change and the affected requirement IDs. A human lead or the integrator agent accepts or rejects it and updates the spec.
- Where the spec is silent, choose the simplest behaviour consistent with it, record it as an **implementation decision** in `docs/decisions/` of the product repo, and raise an SCR if the decision affects another workstream's contract.

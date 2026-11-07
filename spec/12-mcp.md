# 12 · MCP server & MCP connectors

The platform is both an **MCP server** (other AI assistants use platform knowledge) and an **MCP client** (the platform reaches enterprise systems through connector MCP servers).

- **MCP-000** Pin the MCP specification revision at build start, and record it in `docs/decisions/` (the latest stable revision at that time; 2025-11-25 or later). Use the official TypeScript SDK and Streamable HTTP transport. Revisit at each milestone.

## 1. Common rules

- **MCP-001** All tool inputs and outputs are defined as zod schemas in `packages/domain/mcp/` and exported as JSON Schema. Tool descriptions are concise, state side effects, and say that results carry labels.
- **MCP-002** Validation errors are returned as tool execution errors (`isError: true` with a message), not protocol errors.
- **MCP-003** Every tool result that returns content includes a `label_banner` string and `labels` array, and every item includes a `citation` object (`{kind, id, url}`) that deep-links into the web app.

## 2. Platform MCP server (`apps/mcp-server`)

### 2.1 Auth
- OAuth 2.1 protected resource (SEC-003): Protected Resource Metadata (RFC 9728) points to the tenant IdP as authorization server. Scopes: `tacit.read`, `tacit.ask`, `tacit.questions.write`, `tacit.stewardship.read`.
- **MCP-010** Each call runs as the token's principal. The PDP decides as for the web app. Rate limits per SEC-060.

### 2.2 Tools

| Tool | Scope | Input | Output | Side effects |
|---|---|---|---|---|
| `search_knowledge` | read | `{query, filters?: {campaign_ids?, expert_ids?, ki_types?, entity_ids?, document_ids?}, limit?≤20}` | `{results:[{kind: ki\|chunk, id, title, snippet, type?, expert?, score, citation}], label_banner}` | audit |
| `ask_expert_twin` | ask | `{question, scope?: {campaign_ids?, expert_ids?}, conversation_id?}` | `{answer_markdown, confidence, citations:[…], contradictions, can_send_to_expert: bool, conversation_id, label_banner}` | audit |
| `send_question_to_expert` | questions.write | `{question, campaign_id? , expert_id?, context?}` | `{question_id, status, routed_to}` | creates question |
| `get_knowledge_item` | read | `{id}` | KI with current version, source quotes (if permitted), edges (accepted), label | audit |
| `get_document_passage` | read | `{chunk_id}` or `{document_id, anchor}` | text, anchor, document metadata | audit |
| `get_related` | read | `{node: {kind,id}, edge_types?, depth?≤2, limit?≤50}` | `{nodes, edges}` (authorized only) | audit |
| `who_knows` | read | `{topic, include_departed?: bool}` | ranked people with evidence (RET-020) | audit |
| `list_open_questions` | read | `{campaign_id? , mine?: bool}` | questions visible to the caller | — |
| `get_coverage` | read | `{campaign_id}` | coverage areas, scores, gaps | — |
| `get_stewardship_register` | stewardship.read | `{principal_id?: default caller}` | items and transfers where the caller is recipient (STW-033) | audit |
| `find_owner` | read | `{artifact: {system, external_id} \| title_query}` | current owner of record (live via connector), stewardship history if any | audit |

### 2.3 Resources & prompts
- Resources: `tacit://ki/{id}`, `tacit://document/{id}#{anchor}`, `tacit://campaign/{id}/coverage`. Reading one goes through the PDP.
- Prompts: `prebrief_on_topic(topic)`, `draft_turnover_note(principal)`, `explain_with_sources(question)`.

## 3. Connector contract (platform as MCP client)

### 3.1 Capability discovery
- **MCP-020** Each connector is a separate MCP server registered in `connector` (05 §7). On registration and hourly, the hub calls `tools/list` and maps tool names to **capabilities** by the standard names below. Connectors may offer extra tools; the platform ignores them unless mapped by config.
- **MCP-021** The hub authenticates to connectors with service credentials (SEC-004). Connectors enforce their own system's permissions for the service account. The platform is responsible for re-applying labels and the PDP before showing data to users.
- **MCP-022** Connector calls are made only by the worker (never directly from a browser request path), with timeouts, retries and circuit breakers, and are audited as metadata (tool name, connector, counts, duration).

### 3.2 Standard capabilities (tool names)

| Capability | Tool(s) | Purpose |
|---|---|---|
| `artifact_discovery` | `find_artifacts_by_person`, `get_artifact_owner` | 09 §4 |
| `document_content` | `get_document_content` → `{mime, bytes_base64 \| text, revision, metadata, classification_hint}` | ingestion |
| `document_search` | `search_documents` → `{items:[{external_id,title,doc_number,doc_type,snippet,url}]}` | agents, dossier |
| `document_revisions` | `list_revisions_since(since)` → changed documents | staleness (AGT-060) |
| `entity_master` | `list_entities(kind, cursor)` | equipment, people, vendors registry seed |
| `person_directory` | `get_person(alias)` → HR attributes, manager, status, departure date | identity, routing |
| `role_registry` | `roles_by_person(aliases)` | C-category verification |
| `work_items` | `open_items_by_assignee(aliases)` | D-category (CMMS/CAP/commitments) |
| `cap_intake` | `create_intake_draft({title, description, document_ref, submitter})` → `{draft_id, url}` | deviations (STW-021); draft only, human submits |
| `owner_change` | `request_owner_change({external_ids, new_owner_alias, justification, requested_by, idempotency_key})` → `{reference, url}` | write-back (09 §7) |
| `calendar` | `recurring_meetings_by_organizer(alias)`, `free_busy(principals, window)` | C-05, scheduling |
| `qualifications` | `qualifications_by_person`, `courses_by_owner`, `evaluator_assignments` | C-03, I-* |
| `custody` | `items_by_custodian(alias)` | G-* |

- **MCP-023** Every list tool supports `cursor` and `page_size` and returns `next_cursor`.
- **MCP-024** `packages/connectors-sdk` ships: zod schemas for every standard tool, a helper to build a compliant connector server, and a **conformance test suite** that any connector must pass (19 TST-040).

### 3.3 Fixture connectors (required for development and tests)
Under `connectors/fixture-*`, implement standalone MCP servers backed by JSON fixtures of the synthetic plant (19 §5): `fixture-dms`, `fixture-wiki`, `fixture-cmms`, `fixture-cap`, `fixture-hr` (person_directory, role_registry), `fixture-cmdb` (artifact_discovery over apps/accounts), `fixture-contracts`, `fixture-lms`. Each implements every capability its system kind supports, including `owner_change`. The fixture mutates owner fields after a configurable delay, so verification flows can be tested end to end.

### 3.4 Real connectors (later milestones)
Target first: SharePoint/OneDrive (Microsoft Graph), Confluence, Documentum (REST), ServiceNow (CMDB + tasks), Maximo (OSLC/REST), Active Directory/Entra ID (Graph), Exchange calendar. Each is its own package, passes conformance, and is documented with required service-account permissions (read-only unless `owner_change`).

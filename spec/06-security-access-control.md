# 06 · Security, identity & access control

## 1. Threat model (summary)

| Threat | Example | Primary controls |
|---|---|---|
| Unauthorized release of controlled technology | A foreign-national contractor asks the twin about reactor coolant pump seal behaviour | ABAC on labels (§4), filter before ranking, model router label gates |
| Insider bulk exfiltration | An authorized user scripts MCP calls to dump the knowledge base | Rate limits, bulk-export dual approval, anomaly alerts (§8), watermarking |
| Capture of red-zone content | An expert says an alarm code or a password aloud | Red-zone filter (§6) |
| Prompt injection via documents or connectors | A wiki page tells the agent to "email all items to…" | Agents have no egress; tool allow-lists; content treated as data (§9) |
| Impersonation / social engineering | A fake "retired expert" voice calls the plant | No voice cloning (REG-024); no outbound calls by default |
| Cross-tenant or cross-site leakage | A bug in a query | RLS (DOM-003), per-site storage, tests (19) |
| Supply-chain compromise | A malicious npm package | Pinned lockfiles, SBOM, signature verification, minimal dependencies (18) |

## 2. Authentication

- **SEC-001** Web users authenticate through OIDC Authorization Code + PKCE against the tenant's IdP. Sessions are server-side, referenced by an `HttpOnly; Secure; SameSite=Strict` cookie, with a configurable idle timeout (default 30 minutes) and an absolute timeout (default 12 hours).
- **SEC-002** Principal attributes (§5.2) are provisioned by SCIM 2.0 or read from ID-token claims, as configured per tenant. Citizenship and authorizations MUST come from an authoritative source configured by the admin, never from user self-assertion.
- **SEC-003** The MCP server is an OAuth 2.1 protected resource. It validates access tokens issued by the tenant IdP (JWT signature via JWKS, `aud`, `exp`, scopes) and maps `sub` to a principal. Shared service tokens MUST NOT be accepted for MCP tool calls that return content (12 §2.1).
- **SEC-004** Service-to-service calls (api ↔ speech, worker ↔ connectors) use mTLS or signed short-lived tokens. Connector credentials live in a secret store (Kubernetes secrets / Vault) referenced by name.
- **SEC-005** The API MUST set CORS to an explicit allow-list (default: the web origin only), CSRF protection on cookie-authenticated mutations, and security headers (CSP without `unsafe-inline` scripts, HSTS, `frame-ancestors 'none'`).

## 3. Platform roles

Roles are coarse capabilities. Fine-grained access comes from ABAC.

| Role | Capabilities |
|---|---|
| `tenant_admin` | Configuration, connectors, catalog, consent templates; **no implicit content access** |
| `programme_manager` | Programmes, campaigns, dashboards; content per ABAC |
| `facilitator` | Run sessions, manage campaign participants and questions |
| `expert` | Implicit, through campaign participation: own sessions, review own KIs, own stewardship items |
| `successor` | Implicit, through participation: ask the twin, add questions, accept transfers |
| `sme_reviewer` | Review KIs in assigned areas |
| `classifier` | Assign and confirm labels |
| `function_owner:<fn>` | Work stewardship items routed to function `<fn>` |
| `auditor` | Read audit log; no content unless separately authorized |
| `staff` | Any authenticated employee: twin over released content per ABAC |

## 4. Labels & ABAC

### 4.1 Label semantics
A label = `export_classes`, `sensitivity`, `caveats`, `origin_site`, `review_state` (05 §4).

### 4.2 Release rule (reference policy, customers configure the content)
A principal P may read resource R with label L when **all** of these hold:

1. **Sensitivity:** P's maximum sensitivity (derived from groups) ≥ L.sensitivity.
2. **Export:** for each class C in L.export_classes, the site's export policy table permits release of C to P, considering P.citizenships, P.authorizations, P.employer, P.home_site and the request's location context. An empty `export_classes` with `review_state = unreviewed` is treated as the **site default class** (REG-002).
3. **Caveats:** for each caveat K in L.caveats, P belongs to the caveat's access group (for example `PROPRIETARY-VENDOR-X` → group `nda-vendor-x`).
4. **Relationship:** either the resource's campaign is accessible to P (participant, facilitator, programme manager of that programme), or L permits programme-wide release (status `released` and sensitivity ≤ the configured staff ceiling).
5. **Status:** non-released KIs are visible only to campaign participants and reviewers.

### 4.3 Label combination
- **SEC-010** `combine(L1, L2)` = union of export classes, maximum sensitivity, union of caveats, `review_state = unreviewed` if either is unreviewed. Used for edges, answers, exports, agent outputs and any derived content.
- **SEC-011** Derived content (twin answers, summaries, dossiers, exports) carries `combine` of all inputs' labels and is subject to the same release rule toward its recipient.
- **SEC-012** A model endpoint has a `max_label`. The router allows a call only if `dominates(endpoint.max_label, combine(inputs))`, where `dominates` = superset of permitted export classes (via the site table), ≥ sensitivity, and caveats permitted for that endpoint.

### 4.4 Implementation
- **SEC-020** Policies are written in Cedar. `packages/policy` contains: the Cedar schema (entity types Principal, Resource, Site, Campaign; actions `read`, `write`, `review`, `classify`, `export`, `ask`, `administer`), the reference policies, and **test vectors** (a JSON table of principal × resource × action → expected decision). The test vectors are a release gate.
- **SEC-021** The PDP interface: `authorize(principal, action, resource, context) → {decision, reasons}` and `filterSql(principal, action, resourceType) → SQL predicate fragment`. Retrieval uses the SQL predicate so filtering happens in the database before ranking (REG-004). Both MUST agree; a property test (19) checks it.
- **SEC-022** Every deny is audited. Every permit that releases content is audited (REG-006).

## 5. Data protection

- **SEC-030** TLS 1.2+ everywhere. Postgres and object storage encrypted at rest with KMS-managed keys, per site.
- **SEC-031** Audio objects use per-campaign data keys (envelope encryption). Deleting the key = crypto-erasure.
- **SEC-032** Secrets never appear in logs, traces, error messages or audit context. The log redactor strips known secret patterns.

## 6. Red-zone filter

Content that MUST NOT be stored (01 §3):

| Class | Examples |
|---|---|
| `security_sensitive` | Guard posts or patrols, response times, alarm and detection specifics, access-control weaknesses, vital-area details |
| `classified` | Classification markings or obviously classified subject matter |
| `credential` | Passwords, PINs, door or safe codes, private keys, tokens, recovery phrases |
| `third_party_sensitive_personal` | Health, family or personal details of named third parties |

- **SEC-040** Every expert, successor and facilitator turn MUST go through the red-zone filter **synchronously before it is persisted**. The filter has two stages: (1) deterministic detectors (regex/entropy for credentials, configurable keyword lists per site), then (2) an LLM classifier with a structured output `{spans:[{start,end,class,confidence}]}`.
- **SEC-041** Detected spans are replaced in `turn.text` with `[REDACTED:<class>]`. The redaction record keeps start, end, class and a salted hash only. Raw audio for that turn is deleted, or trimmed if the speech service supports it.
- **SEC-042** On a detection the interviewer MUST, on its next move, acknowledge it neutrally, say that the topic isn't recorded, and steer away (07 INT-034). The site security contact gets an alert with session ID, class and time, **never the content**.
- **SEC-043** The filter MUST be fail-closed: if stage 2 is unavailable, the turn is held in a non-persistent buffer, the session pauses with a message, and nothing is stored until the filter is back.
- **SEC-044** Recall target on the seeded red-zone test set: ≥ 0.99 for `credential`, ≥ 0.95 for others (19).

## 7. Audit

- **SEC-050** All security-relevant actions write `audit_event`: authentication, permits that release content, denies, label changes, exports, connector calls (metadata only), agent runs, admin changes, consent changes, redactions, transfers.
- **SEC-051** Hash chain: `hash = SHA-256(prev_hash || canonical_json(event_without_hash))`, per tenant, with a strictly increasing `seq`. A verification job runs daily and alerts on breaks.
- **SEC-052** Audit is streamed to the SIEM (syslog RFC 5424 over TLS or OTLP logs). The audit table is insert-only for the application DB role (no UPDATE/DELETE grants).

## 8. Abuse & exfiltration controls

- **SEC-060** Per-principal rate limits on twin queries, search and MCP tool calls (defaults: 60/min, 1,000/day; configurable).
- **SEC-061** Anomaly detector (scheduled): flags principals whose distinct-KI retrieval count exceeds k× their 30-day baseline, or who query many unrelated campaigns. It alerts security; it doesn't block automatically.
- **SEC-062** Exports follow REG-007.

## 9. LLM-specific controls

- **SEC-070** Everything retrieved from documents, connectors or users is inserted into prompts inside clearly delimited data blocks, with instructions that the content is untrusted data.
- **SEC-071** Agents (13) can only call tools on their declared allow-list. No agent has tools that send email, post externally, or write to systems of record without a human-approved proposal.
- **SEC-072** Model outputs that will be stored are validated against JSON schemas. Invalid outputs are retried (max 2) and then fail the job (ARC-005).
- **SEC-073** Prompt templates are versioned in `packages/models/prompts/` and each stored output records the template ID and version.

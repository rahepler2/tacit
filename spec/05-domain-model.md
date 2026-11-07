# 05 · Domain model

**This document is the source of truth for entity names, fields and state machines.** `packages/domain` (zod) and `packages/db` (migrations) MUST implement it exactly. Any change goes through an SCR.

## 1. Conventions

- **DOM-001** Primary keys are UUIDv7 (`id uuid`). Public IDs in URLs are the same UUIDs.
- **DOM-002** Every table has `tenant_id uuid not null`, `created_at timestamptz`, `updated_at timestamptz`, and (where content-bearing) `site_id uuid not null` and `label_id uuid not null`.
- **DOM-003** Row-level security on every tenant table: `tenant_id = current_setting('app.tenant_id')::uuid`. The API sets `app.tenant_id` and `app.principal_id` per transaction. RLS is defence in depth; the PDP (06) remains the authority.
- **DOM-004** Soft delete is not used for content. Redaction (REG-022) and erasure are explicit operations that leave tombstones.
- **DOM-005** Enumerations are Postgres `text` columns with `CHECK` constraints, mirrored as zod enums.
- **DOM-006** JSON columns (`jsonb`) are allowed only where noted, and each has a zod schema.
- Types: `text`, `int`, `numeric`, `bool`, `timestamptz`, `date`, `uuid`, `jsonb`, `vector(D)` (D = configured embedding dimension).

## 2. Organization & identity

### tenant
| field | type | notes |
|---|---|---|
| id | uuid | |
| name | text | |
| settings | jsonb | `TenantSettings`: default locale, branding, feature flags |

### site
| field | type | notes |
|---|---|---|
| id | uuid | |
| tenant_id | uuid | |
| code | text | short code, unique per tenant |
| name | text | |
| country | text | ISO 3166-1 alpha-2 |
| jurisdictions | text[] | export regimes applicable, for example `{US-810, EAR}` |
| default_label_id | uuid | REG-002 |
| consent_template_id | uuid | 07 §3 |
| audio_retention_days | int | REG-021 |
| settings | jsonb | `SiteSettings`: working languages, telephony allowed, camera allowed, ECP contact, function-owner routing (08 §6) |

### principal
Users and service identities. Attributes come from the IdP and SCIM; local edits are limited to platform roles.
| field | type | notes |
|---|---|---|
| id | uuid | |
| kind | text | `user` \| `service` |
| external_subject | text | IdP `sub`, unique per tenant |
| employee_id | text? | HR ID |
| display_name | text | |
| email | text? | |
| home_site_id | uuid? | |
| employer | text | employer organization (contractors differ) |
| citizenships | text[] | ISO country codes; **from authoritative source only** (SCIM attribute or HR connector) |
| authorizations | text[] | export authorizations held, for example `US-810-GA`, `EAR-LIC-123` |
| clearances | text[] | |
| groups | text[] | IdP groups |
| platform_roles | text[] | 06 §3 |
| status | text | `active` \| `departing` \| `departed` \| `disabled` |
| departure_date | date? | |

### identity_alias
Links a principal to how they appear in external systems (09 §3).
| field | type | notes |
|---|---|---|
| id | uuid | |
| principal_id | uuid | |
| system | text | connector ID or `hr`, `ad`, `email` |
| value | text | account name, display-name variant, initials, employee number, email |
| kind | text | `account` \| `email` \| `display_name` \| `initials` \| `employee_id` \| `signature_name` |
| confidence | numeric | 0–1 |
| status | text | `proposed` \| `confirmed` \| `rejected` |
| confirmed_by | uuid? | |

## 3. Programme, risk & competency

### programme
`id, tenant_id, name, description, status (draft|active|closed), owner_principal_id`

### position
`id, tenant_id, site_id, code, title, org_unit, criticality (1-5)`

### job_task
JTA tasks per position.
`id, tenant_id, position_id, code, title, description, criticality (1-5), frequency (daily|weekly|monthly|quarterly|annual|rare|event), source_ref (text, the training-programme reference)`

### risk_entry
KLRA register line.
`id, tenant_id, programme_id, principal_id, position_id, attrition_score (1-5), position_score (1-5), total_score (generated = attrition × position), assessed_at, notes`

## 4. Labels

### label
Immutable value objects. Resources point to a label row; changing a label means pointing at another row.
| field | type | notes |
|---|---|---|
| id | uuid | |
| tenant_id | uuid | |
| export_classes | text[] | for example `{US-810}`, `{EAR-0E001}`, `{NLR}`; empty = unknown |
| sensitivity | text | `public` \| `internal` \| `confidential` \| `restricted` |
| caveats | text[] | handling caveats, for example `PROPRIETARY-VENDOR-X`, `PERSONAL-DATA`, `LEGAL-PRIVILEGED` |
| origin_site_id | uuid | |
| review_state | text | `unreviewed` \| `classified` |
| classified_by | uuid? | |
| classified_at | timestamptz? | |
| hash | text | canonical hash of the fields above (unique per tenant) |

Dominance and ordering are defined in 06 §4.

## 5. Campaigns & sessions

### campaign
| field | type | notes |
|---|---|---|
| id | uuid | |
| tenant_id, site_id, label_id | | |
| programme_id | uuid | |
| expert_principal_id | uuid | |
| position_id | uuid? | |
| title | text | |
| context | text | free-text responsibilities and systems (labelled) |
| status | text | see state machine |
| departure_date | date | copied from principal at creation, editable |
| target_session_count | int | default 6 |
| interview_language | text | BCP-47 |
| knowledge_language | text | BCP-47; KIs are written in this language (default `en`) |
| post_departure_channel | text | `none` \| `consulting_agreement` \| `alumni_portal` |
| created_by | uuid | |

**Campaign status:** `draft → consent_pending → active → closing → closed`, plus `suspended` (reachable from active or closing, returns to the previous state). `closing` is entered when the facilitator starts the exit process; `closed` needs the gate in 08 §7 (STW-040).

### campaign_participant
`id, campaign_id, principal_id (nullable for group successors), group_name?, role (expert|successor|facilitator|sme_reviewer|classifier|observer), domains (uuid[] coverage areas they focus on), added_at`

### coverage_area
The campaign's coverage plan (14).
| field | type | notes |
|---|---|---|
| id | uuid | |
| campaign_id | uuid | |
| kind | text | `job_task` \| `system` \| `document_set` \| `topic` \| `stewardship` |
| ref_id | uuid? | job_task / entity / … |
| title | text | |
| priority | int | 1 (highest) to 5 |
| target_questions | text[] | |
| coverage_score | numeric | 0–1, computed (14 §3) |
| status | text | `planned` \| `in_progress` \| `saturated` \| `deferred` |

### consent
| field | type | notes |
|---|---|---|
| id | uuid | |
| campaign_id | uuid | |
| principal_id | uuid | |
| template_id | uuid | |
| template_version | int | |
| scope | text[] | `audio_recording`, `transcription`, `ai_processing`, `attribution`, `post_departure_contact` |
| granted_at | timestamptz | |
| withdrawn_at | timestamptz? | |
| method | text | `click` \| `verbal_recorded` \| `paper_uploaded` |
| evidence_ref | text? | object key of evidence |

### consent_template
`id, tenant_id, site_id, version, language, title, body_markdown, scopes_offered text[], approved_by, approved_at`

### session
| field | type | notes |
|---|---|---|
| id | uuid | |
| campaign_id | uuid | |
| site_id, label_id | | |
| type | text | `kickoff` \| `deep_dive` \| `document_walkthrough` \| `scenario` \| `successor_led` \| `field_walkdown` \| `validation` \| `group` \| `stewardship` \| `exit` |
| mode | text | `voice` \| `text` \| `hybrid` \| `phone` |
| status | text | `scheduled → ready → live → paused → ended → processed`; `cancelled` |
| scheduled_for | timestamptz? | |
| started_at / ended_at | timestamptz? | |
| agenda | jsonb | `SessionAgenda` (07 §5) |
| focus_area_ids | uuid[] | coverage areas |
| document_ids | uuid[] | for document walkthroughs |
| summary | text? | generated, expert-approved |
| summary_approved_at | timestamptz? | |

### turn
| field | type | notes |
|---|---|---|
| id | uuid | |
| session_id, campaign_id, site_id, label_id | | |
| seq | int | monotonic within the session, unique (session_id, seq) |
| speaker_role | text | `interviewer_ai` \| `facilitator` \| `expert` \| `successor` \| `other` |
| speaker_principal_id | uuid? | |
| text | text | normalized final text (after red-zone redaction) |
| original_language | text | |
| translation | text? | into the knowledge language, if different |
| started_at / ended_at | timestamptz | |
| audio_object_key | text? | |
| interviewer_move | text? | for AI turns: `open` \| `ask_planned` \| `follow_up` \| `ask_queued` \| `stewardship` \| `clarify` \| `summarize` \| `transition` \| `close` |
| question_id | uuid? | the question this turn asks or answers |
| coverage_area_id | uuid? | |
| redactions | jsonb | `[{start, end, reason, hash}]` |
| extraction_status | text | `pending` \| `done` \| `failed` \| `skipped` |

## 6. Knowledge

### knowledge_item
| field | type | notes |
|---|---|---|
| id | uuid | |
| campaign_id, site_id, label_id | | |
| current_version | int | |
| type | text | 10 §2 |
| title | text | ≤ 140 characters |
| body | text | Markdown |
| attributes | jsonb | type-specific structured fields (10 §2) |
| confidence | numeric | 0–1 extraction confidence |
| status | text | `proposed → expert_review → sme_review → classification → released`; `rejected`, `superseded`, `redacted` |
| attributed_to | uuid | usually the expert |
| coverage_area_ids | uuid[] | |
| released_at | timestamptz? | |
| embedding | vector(D)? | of title + body of current version |
| fts | tsvector | generated |

### knowledge_item_version
`id, knowledge_item_id, version, type, title, body, attributes, changed_by (principal or agent_run id), change_kind (extracted|edited|merged|split|reclassified|redacted|restored), reason, created_at`. Append-only.

### source_span
| field | type | notes |
|---|---|---|
| id | uuid | |
| knowledge_item_id | uuid | |
| kind | text | `turn` \| `document_chunk` \| `artifact` |
| turn_id / chunk_id / artifact_id | uuid? | exactly one set |
| start_char / end_char | int? | |
| quote | text | verbatim evidence (original language) |
| quote_translation | text? | |

### review
`id, target_type (knowledge_item|edge|label|stewardship_item|identity_alias|session_summary), target_id, target_version, stage (expert|sme|classifier|owner), decision (approve|reject|request_change), comment, reviewer_id, created_at`. Immutable.

## 7. Documents & entities

### connector
`id, tenant_id, site_id, kind (dms|wiki|policy|cmms|cap|commitments|hr|iam|cmdb|contracts|lms|code|records|calendar|other), name, mcp_url, auth (jsonb: reference to a secret, never the secret), capabilities text[] (12 §3), enabled bool, write_enabled bool, last_health_at, health`

### document
| field | type | notes |
|---|---|---|
| id | uuid | |
| site_id, label_id | | |
| connector_id | uuid? | null for uploads |
| external_id | text? | |
| doc_number | text? | for example `OP-3-FW-004` |
| title | text | |
| doc_type | text | normalized document type (09 §5) |
| status | text | `draft` \| `effective` \| `superseded` \| `cancelled` \| `unknown` |
| current_revision | text? | |
| url | text? | deep link |
| owner_of_record | text? | as reported by the SoR |

### document_version
`id, document_id, revision, effective_date?, object_key, sha256, ingested_at, parse_status`

### document_chunk
`id, document_version_id, seq, anchor (jsonb: {section?, step?, page?, table?, cell?}), text, embedding vector(D), fts tsvector, label_id`

### entity
| field | type | notes |
|---|---|---|
| id | uuid | |
| tenant_id, site_id | | |
| kind | text | `equipment` \| `system` \| `person` \| `organization` \| `vendor` \| `document_ref` \| `regulation` \| `location` \| `software` \| `task` \| `committee` \| `agreement` |
| canonical_name | text | |
| external_ref | text? | equipment tag, vendor ID, … |
| aliases | text[] | |
| principal_id | uuid? | when kind = person and it's an internal user |
| attributes | jsonb | |

### mention
`id, entity_id, source_kind (turn|knowledge_item|document_chunk), source_id, start_char, end_char, surface_text, confidence, status (proposed|confirmed|rejected)`

### edge
| field | type | notes |
|---|---|---|
| id | uuid | |
| tenant_id, label_id | | label = max of endpoint labels (06 §4) |
| src_kind / src_id | text / uuid | `knowledge_item` \| `document` \| `document_chunk` \| `entity` \| `question` \| `artifact` \| `stewardship_item` |
| dst_kind / dst_id | text / uuid | same set |
| type | text | vocabulary in 10 §6 |
| confidence | numeric | |
| evidence | text? | short justification or quote |
| created_by_kind | text | `human` \| `agent` |
| created_by | uuid | |
| status | text | `proposed` \| `accepted` \| `rejected` |

## 8. Questions

### question
| field | type | notes |
|---|---|---|
| id | uuid | |
| campaign_id, label_id | | |
| text | text | |
| source | text | `plan` \| `successor` \| `gap` \| `follow_up` \| `contradiction` \| `reviewer` \| `document_gap` \| `stale_after_revision` \| `stewardship` |
| priority | int | 1–5 |
| status | text | `open → scheduled → asked → answered`; `declined`, `obsolete`; `asked` can return to `open` |
| asked_by | uuid? | |
| coverage_area_id | uuid? | |
| scheduled_session_id | uuid? | |
| asked_in_turn_id | uuid? | |
| resolution | jsonb? | `{knowledge_item_ids, judged_by: agent|human, judge_confidence, note}` |
| post_departure | bool | eligible for the post-departure channel |

## 9. Stewardship (08, 09)

### stewardship_catalog_item
The configurable always-ask catalog (08 §4). Versioned.
`id, tenant_id, code (for example STW-A-01), version, category (A..L), title, prompt_spoken, prompt_formal, applies_when (jsonb rule), response_schema (jsonb JSON-schema), routing_function, required bool, verification_capability text?, active bool`

### stewardship_item
One instance per campaign per catalog item, plus discovered or ad-hoc items.
| field | type | notes |
|---|---|---|
| id | uuid | |
| campaign_id, site_id, label_id | | |
| catalog_item_id | uuid? | null for ad-hoc |
| category | text | A..L |
| kind | text | `question` (asked, needs an answer) \| `holding` (something held that needs reassigning) \| `obligation` \| `notice` (route-only) |
| title | text | |
| details | jsonb | validated by the catalog `response_schema` |
| artifact_id | uuid? | when it comes from discovery |
| status | text | see 08 §5 |
| routing_function | text | `legal` \| `records` \| `iam` \| `security` \| `export_control` \| `training` \| `engineering` \| `licensing` \| `supply_chain` \| `hr` \| `ecp` \| `facilities` \| `rp` \| `programme` |
| assignee_principal_id | uuid? | function owner working the item |
| due_date | date? | defaults to departure_date − configured lead time |
| na_reason | text? | |
| source_turn_id | uuid? | |

### stewardship_transfer
| field | type | notes |
|---|---|---|
| id | uuid | |
| stewardship_item_id | uuid | |
| recipient_principal_id | uuid? | |
| recipient_group | text? | when transferring to a role or group |
| transfer_kind | text | `ownership` \| `custodianship` \| `responsibility` \| `point_of_contact` \| `designation` \| `access` \| `physical_custody` |
| status | text | `proposed → recipient_notified → recipient_accepted → sor_update_requested → sor_updated → verified`; `recipient_declined`, `cancelled` |
| handover_notes | text? | |
| sor_reference | text? | change ticket or DMS workflow ID |
| verified_at | timestamptz? | |
| verified_by | text? | `connector:<id>` or principal |

### artifact
Normalized record discovered in an external system (09 §4).
| field | type | notes |
|---|---|---|
| id | uuid | |
| campaign_id, site_id, label_id | | |
| connector_id | uuid | |
| external_id | text | unique per (connector, external_id, campaign) |
| artifact_type | text | 09 §5 taxonomy |
| doc_type | text? | finer type (procedure, calculation, …) |
| title | text | |
| status | text | `active` \| `draft` \| `superseded` \| `closed` \| `unknown` |
| person_roles | text[] | roles the expert holds on it: `author` \| `owner` \| `approver` \| `reviewer` \| `custodian` \| `assignee` \| `maintainer` \| `contact` \| `signatory` \| `instructor` \| `member` \| `last_modified_by` \| `contributor` |
| owner_of_record | text? | as the SoR shows it |
| owner_of_record_principal_id | uuid? | resolved |
| url | text? | |
| last_activity_at | timestamptz? | |
| raw | jsonb | connector payload (labelled with the artifact's label) |
| document_id | uuid? | if ingested as a document |
| match_alias_id | uuid? | which identity alias matched |
| match_confidence | numeric | |
| match_status | text | `proposed` \| `confirmed` \| `rejected` |

## 10. Agents, audit, evaluation

### agent_run
`id, tenant_id, campaign_id?, agent (13 §2 name), trigger (event|schedule|manual), status (queued|running|succeeded|failed|cancelled), input (jsonb), output_summary (jsonb), model_endpoint_ids text[], tokens_in, tokens_out, started_at, ended_at, error?`

### audit_event
Append-only, hash-chained per tenant (06 §7).
`id, tenant_id, seq bigint, at, principal_id?, agent_run_id?, action, resource_type, resource_id?, label_ids uuid[], decision (permit|deny|n/a), context jsonb, prev_hash, hash`

### eval_run
`id, tenant_id, campaign_id?, suite, model_endpoint_ids, metrics jsonb, started_at, ended_at`

## 11. Relationship map (summary)

```
tenant ─< site ─< principal ─< identity_alias
tenant ─< programme ─< campaign ─< campaign_participant
                       campaign ─< session ─< turn
                       campaign ─< knowledge_item ─< knowledge_item_version
                                       knowledge_item ─< source_span >─ turn | document_chunk | artifact
                       campaign ─< question
                       campaign ─< coverage_area
                       campaign ─< stewardship_item ─< stewardship_transfer
                       campaign ─< artifact >─ connector
document ─< document_version ─< document_chunk
entity ─< mention ;  edge (any node ⇄ any node)
every content row ─> label
```

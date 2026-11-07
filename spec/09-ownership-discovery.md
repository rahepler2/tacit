# 09 · Ownership discovery & transfer verification

## 1. Purpose

Experts underestimate what they own. Ownership discovery searches every connected system of record for artifacts the expert **authored, owns, approves, maintains, is assigned to or is the contact for**. It shows the results back to the expert by type ("You own 42 procedures, 13 calculations and 3 wiki spaces"), asks whether each has been reassigned and to whom, and later **verifies** reassignment by searching again.

## 2. Flow

```
campaign activated
  → resolve identity aliases (§3)
  → fan-out: artifact_discovery on every enabled connector (§4)
  → normalize → artifact rows (match_status = proposed | confirmed by rule)
  → facilitator/expert confirm uncertain matches (UI-042)
  → group by system × artifact_type × doc_type × role (§5)
  → generate stewardship holding items (08 B-02, C-*, D-*, E-*, …) with suggested recipients (§6)
  → dossier + agenda for document walk-throughs (13 AGT-010)
  … transfers progress …
  → periodic re-check (§8) → transfers verified
  → post-departure orphan sweep (§9)
```

## 3. Identity resolution

People appear in systems in different ways: AD account `jsmith2`, email, employee number, "J. Smith", "John A. Smith", "JAS" initials on 1990s calculations, a signature block name.

- **OWN-001** On campaign activation, build candidate aliases from: the IdP (account, email, UPN), HR (employee ID, legal name, preferred name, former names if provided), and generated variants (initial + surname, surname + initials, initials). Store them as `identity_alias` with `confidence`.
- **OWN-002** Alias kinds `account`, `email` and `employee_id` from authoritative sources are `confirmed` automatically. Name variants and initials are `proposed` and used for search, but an artifact matched **only** through a proposed alias gets `match_status = proposed`.
- **OWN-003** Initials-only and common-name matches need human confirmation (expert or facilitator) and MUST show context (document title, date, co-signers) to help decide. Confirmations can apply to "all matches of this alias in this system".
- **OWN-004** False-positive protection: if an alias matches more than one active principal in HR, mark it ambiguous; artifacts matched by it can't be auto-confirmed.

## 4. Connector capability: `artifact_discovery`

Every connector that can do so implements the MCP tool contract below (12 §3.2):

**Tool `find_artifacts_by_person`**
```json
{
  "input": {
    "aliases": [{"kind": "account|email|employee_id|display_name|initials|signature_name", "value": "string"}],
    "roles": ["author","owner","approver","reviewer","custodian","assignee","maintainer","contact","signatory","instructor","member","last_modified_by","contributor"],
    "include_inactive": false,
    "modified_since": "date|null",
    "cursor": "string|null",
    "page_size": 200
  },
  "output": {
    "items": [{
      "external_id": "string",
      "artifact_type": "string (taxonomy §5)",
      "doc_type": "string|null",
      "title": "string",
      "doc_number": "string|null",
      "revision": "string|null",
      "status": "active|draft|superseded|closed|unknown",
      "person_roles": [{"role": "string", "matched_alias": {"kind":"…","value":"…"}, "evidence": "field name or text"}],
      "owner_of_record": {"display": "string", "account": "string|null", "email": "string|null"} ,
      "url": "string|null",
      "last_activity_at": "datetime|null",
      "classification_hint": {"export_classes": ["string"], "caveats": ["string"], "sensitivity": "string|null"},
      "extra": {}
    }],
    "next_cursor": "string|null"
  }
}
```

**Tool `get_artifact_owner`**: input `{external_id}` → `{owner_of_record, assignees[], status, last_activity_at}`. Used for verification (§8).

- **OWN-010** The worker calls `find_artifacts_by_person` on every enabled connector that has the capability, with paging, a per-connector timeout (default 10 minutes total) and retry with backoff. A partial failure on one connector MUST NOT block the others. Per-connector status is shown in the UI.
- **OWN-011** Results are upserted into `artifact` on (connector_id, external_id, campaign_id). Re-runs update roles, owner and status, and keep history in the audit log.
- **OWN-012** Labels: an artifact's label = `combine(site default, classification_hint)`. The raw payload is stored under the same label.
- **OWN-013** Optional ingestion: artifacts of document types in the site's ingest list (default: procedures, calculations, design basis documents, specifications, wiki pages) are queued for document ingestion (10 §8) if the connector also supports `get_document_content`.

### 4.1 Systems and what they contribute (connector targets)

| System kind | Examples | Roles typically found | Artifact types |
|---|---|---|---|
| DMS | Documentum, SharePoint, ProjectWise, OpenText | author, owner, approver, reviewer, last_modified_by | controlled documents by doc type |
| Wiki | Confluence, SharePoint pages, MediaWiki | author (creator), contributor, owner (space admin) | wiki_space, wiki_page |
| Policy & procedure management | Policy management systems, procedure writer tools | owner, approver, reviewer | policy, procedure |
| Records management | Records system | custodian | record series, boxes |
| CMMS | Maximo, SAP PM | assignee, owner (PM owner), approver | work_order, pm_task, asset |
| CAP | CAP system | assignee, originator, owner | condition_report, corrective_action, audit_finding |
| Commitment tracking | Commitment database | owner | regulatory_commitment |
| Contracts / procurement | Contract management, ERP | technical_rep (→ `contact`), signatory, custodian | contract, purchase_order, nda |
| CMDB / IT service management | ServiceNow | owner ("managed by"), support group member | application, database, service_account |
| IAM | Active Directory / Entra ID | owner | group, distribution_list, shared_mailbox, service_account |
| Code hosting | GitLab, GitHub Enterprise, Azure DevOps | maintainer, owner (CODEOWNERS) | repository, pipeline |
| LMS / training | LMS | owner, instructor, evaluator | course, qualification, evaluator_assignment |
| Role registries / HR | HR, ERO roster, system-engineer list | holder | role_designation, delegation |
| Calendar | Exchange / Google | organizer | recurring_meeting |
| Asset / custody | Asset management, key control, source inventory | custodian | equipment, key, material |

## 5. Artifact taxonomy & grouping

- **OWN-020** `artifact_type` is a fixed taxonomy (extend by SCR): `controlled_document`, `wiki_space`, `wiki_page`, `policy`, `record_series`, `work_order`, `pm_task`, `asset`, `condition_report`, `corrective_action`, `audit_finding`, `regulatory_commitment`, `contract`, `purchase_order`, `nda`, `application`, `database`, `service_account`, `group`, `distribution_list`, `shared_mailbox`, `repository`, `pipeline`, `course`, `qualification`, `evaluator_assignment`, `role_designation`, `delegation`, `recurring_meeting`, `equipment`, `key`, `material`, `other`.
- **OWN-021** `doc_type` normalization: a per-tenant mapping table from the source system's types or number prefixes (for example `OP-*` → `operating_procedure`, `CALC-*` → `calculation`) to a normalized vocabulary: `operating_procedure`, `maintenance_procedure`, `surveillance_procedure`, `abnormal_procedure`, `emergency_procedure`, `administrative_procedure`, `policy`, `calculation`, `specification`, `drawing`, `design_basis_document`, `design_change`, `technical_evaluation`, `report`, `training_material`, `form`, `standard`, `guideline`, `wiki`, `other`. Unmapped types go to `other` and show up in an admin "unmapped types" list.
- **OWN-022** Grouping for presentation and bulk transfer: (system, artifact_type, doc_type, role). Each group shows count, active count, examples (top 5 by recency), and the "owner of record still the expert" count.

## 6. Turning artifacts into stewardship items

- **OWN-030** Rules (configurable, defaults below) decide which artifacts become `holding` items:

| Condition | Result |
|---|---|
| role ∈ {owner, custodian, maintainer, contact, signatory, instructor} and status active | holding item (category from artifact_type: documents→B, roles→C, CAP/CMMS/commitments→D, IT→E, contracts→A/F, LMS→I, assets→G) |
| role = assignee and status open | holding item, category D |
| role ∈ {approver, reviewer} on a recurring basis (≥ 3 active docs of the same doc_type) | B-07 item per doc_type |
| role = author only | not a holding. Adds the artifact to the **knowledge dossier** (13 AGT-010) and to B-03 context |
| status superseded/closed | ignored (kept for the dossier) |

- **OWN-031** Items are grouped: one item per group (§5) by default, with the artifacts linked. The expert can split a group when different recipients apply.
- **OWN-032** Suggested recipients, in order: (1) a successor named in the campaign whose focus areas match, (2) the person who most often co-authored or approved the artifacts, from connector data, (3) the expert's manager. These are suggestions only. The expert or facilitator chooses.
- **OWN-033** The interviewer asks about holdings in group form (08 B-02 wording), not one artifact at a time, unless the group has ≤ 3 items.

## 7. Write-back (optional, off by default)

- **OWN-035** When a connector has the `request_owner_change` capability **and** `write_enabled = true`, a function owner may click "Request change in {system}" on a transfer. The platform calls the tool with `{external_ids, new_owner_alias, justification, requested_by}`. The connector creates a **change request** in the SoR's normal workflow (for example a DMS ownership-change workflow or a CMDB change ticket). It MUST NOT directly overwrite the owner field unless the SoR has no workflow and the tenant admin has explicitly allowed direct writes.
- **OWN-036** The returned reference goes into `stewardship_transfer.sor_reference`, and status moves to `sor_update_requested`.
- **OWN-037** Every write-back call needs a human click (REG-031), is audited, and is idempotent (the connector receives an idempotency key).

## 8. Verification re-checks

- **OWN-040** A scheduled job (default daily, and on demand) calls `get_artifact_owner` for every artifact in a transfer with status ≥ `recipient_accepted`. When the owner of record or assignee resolves to the recipient principal, the per-artifact state is `verified`. When all artifacts in a transfer are verified, the transfer becomes `verified`.
- **OWN-041** If the owner of record changes to someone **other** than the recipient, flag a discrepancy for the function owner.
- **OWN-042** Before the departure date (configurable, default T−10 working days), run a full `find_artifacts_by_person` again to catch new or missed holdings, and create items for them.

## 9. Post-departure orphan sweep

- **OWN-050** A programme-level scheduled job (default weekly) searches connectors for artifacts whose owner of record maps to a principal with status `departed` or `disabled`, and lists them on the programme dashboard as **orphaned artifacts**, with the last campaign (if any) and its suggested recipient. This catches people who left without a campaign, and holdings missed during one.

## 10. Acceptance criteria (selection)

- Given aliases {account `jsmith2`, initials `JAS`}, when the DMS fixture returns 30 documents matched by account and 5 matched only by `JAS`, then 30 artifacts are `confirmed` and 5 are `proposed`, awaiting confirmation.
- Given one connector times out, then the others' results are stored, the timed-out connector shows `failed`, and a retry can be triggered.
- Given a transfer for 23 artifacts where the fixture changes owner for 20, when the re-check runs, then 20 artifacts are verified and the transfer stays `sor_updated` (not `verified`) with "20/23 verified".
- Given a departed principal still owner of 4 active documents, when the orphan sweep runs, then 4 orphaned artifacts appear on the dashboard.

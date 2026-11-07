# 08 · Stewardship & Obligations: the always-ask module

## 1. Purpose

Knowledge capture isn't enough on its own. A departing expert also **holds things on behalf of the organization**: agreements they're the named custodian of, documents they own of record, designated roles, regulatory commitments, system accounts, vendor relationships, physical items. Any of these that isn't explicitly reassigned becomes **orphaned**.

This module guarantees that every campaign:
1. **Asks** a configurable catalog of mandatory questions (the "always-ask" catalog), in words that are accurate and appropriate to say aloud.
2. **Discovers** what the expert holds in systems of record, through ownership discovery (09), and asks about each finding.
3. **Records** each holding as a `stewardship_item` with structured details.
4. **Routes** each item to the right function (Legal, Records, IAM, …).
5. **Tracks reassignment** to a named recipient until the recipient accepts **and** the system of record shows the change (**verified**).
6. **Gates** campaign closure on completion, and ends with an expert **attestation**.

The platform is **not the system of record** for agreements, documents, roles or access. It finds, asks, routes, tracks and verifies. Changes happen in the systems of record, either by people or (optionally) through approved write-back requests (09 §7).

## 2. Language principles

- **STW-001** Use precise terms and never let them blur:
  - **Author:** wrote or contributed to the content.
  - **Owner (of record):** accountable for keeping it current and approving changes, as listed in the system of record.
  - **Approver / signatory:** has authority to approve or sign.
  - **Custodian:** has physical or logical custody (of agreements, proprietary information received, files, keys, materials).
  - **Point of contact (POC):** the named person external parties deal with.
  - **Designee / holder:** formally designated to a role or position (for example an emergency-response position).
  - **Reassignment / transfer of ownership:** a named recipient has **accepted** the role, **and** the system of record **shows** them.
  - **Knowledge handover:** the recipient has been briefed. Necessary but **not** the same as reassignment.
- **STW-002** Spoken prompts are conversational, ≤ 45 words, one question at a time. Formal prompts (shown in forms and reports) may be longer and more exact.
- **STW-003** The interviewer MUST NOT ask for the confidential *content* of agreements, legal matters or concerns. Only existence, counterparty, identifiers, custodian, location and continuing obligations.
- **STW-004** The interviewer MUST NOT give legal or HR advice. If the expert asks ("Am I still bound by this after I retire?"), it records the question and routes it (`legal` or `hr`) with a neutral reply: "That's a good one for Legal; I'll make sure it gets to them."

## 3. When items are asked

- **STW-010** Every active catalog item whose `applies_when` rule matches the campaign is **instantiated** as a `stewardship_item` (kind `question`, status `pending`) when the campaign becomes `active`.
- **STW-011** Items are scheduled across sessions:
  - `kickoff`: a short scan. Catalog codes marked `scan: true` (A-01, B-01, C-01, E-01) get a yes/no/"tell me more" pass to size the work.
  - Discovery results (09) become `holding` items as soon as they're confirmed.
  - `stewardship` session: a full pass through all remaining items, grouped by category, starting with the ones that have the nearest due dates.
  - Any session: if the expert mentions a holding spontaneously ("I'm still the contact for the pump vendor"), the extractor proposes a stewardship item (10 KNW-030), which the planner can confirm in-thread.
  - `exit`: every remaining open item, plus the attestation (§7).
- **STW-012** Required items MUST be asked at least once in a live session or completed by the expert in the stewardship form (UI-040). The form exists for items better answered by looking things up than by talking.

## 4. The catalog (reference content v1)

The catalog is **data**, seeded from this table and editable per tenant (versioned, `stewardship_catalog_item`). Columns:
- **Code**
- **Ask (spoken)**: conversational wording
- **Capture**: the fields of `details`, which become a JSON schema
- **Verify**: the connector capability used to cross-check (12 §3)
- **Route**: the function that receives the item
- **Req.**: required by default

Every "yes" answer creates one or more `holding` or `obligation` items, each with a transfer workflow (§5), unless noted otherwise.

### A. Agreements, confidentiality & proprietary information

| Code | Ask (spoken) | Capture | Verify | Route | Req. |
|---|---|---|---|---|---|
| A-01 | "Are you the company's named contact or custodian on any non-disclosure, confidentiality or proprietary-information agreements, for example with vendors, owners groups or other utilities?" | per agreement: counterparty, agreement title/number, your role (`signatory`/`custodian`/`poc`), where it's filed, continuing obligations (y/n/unsure), proposed new custodian | `contracts.search_by_person` | legal | ✔ |
| A-02 | "Do you hold any proprietary information received from outside companies (drawings, reports, software, data) in your files, email or drives? Where is it kept?" | source company, agreement ref (link to A-01 item), item description, location (system/path/physical), proposed custodian, disposition (`transfer`/`return`/`destroy`/`archive`) | `dms.search_by_person` (caveat-tagged docs) | legal | ✔ |
| A-03 | "Are you the technical representative or named contact on any contracts or purchase orders?" | contract/PO number, vendor, role, end date, proposed new rep | `contracts.search_by_person` | supply_chain | ✔ |
| A-04 | "Are you named on any export licences, authorizations or technology-control plans, or responsible for any export-control decisions?" | authorization ID, regime, role, proposed new responsible person | `contracts.search_by_person` / manual | export_control | ✔ |
| A-05 | "Are you party to any data-sharing, teaming, research or memorandum-of-understanding agreements on the company's behalf?" | as A-01 | `contracts.search_by_person` | legal | ✔ |
| A-06 | "Is there anything you've signed personally for the company that continues after you leave, such as warranties, certifications or attestations?" | description, reference, continuing effect | manual | legal | ✔ |
| A-07 | *(notice, not asked)* Remind the expert that their own employment confidentiality and post-employment obligations continue, and give the HR contact. | acknowledgement only | n/a | hr | ✔ |

### B. Document & record ownership

| Code | Ask (spoken) | Capture | Verify | Route | Req. |
|---|---|---|---|---|---|
| B-01 | "Which types of documents are you the owner of? For example procedures, calculations, specifications, drawings, policies, training materials or wiki spaces." | list of `doc_type` with approximate counts, systems they live in | `artifact_discovery` (all doc systems) | records | ✔ |
| B-02 | *(per discovered artifact or group)* "Records show you as owner of {N} {doc_type}s in {system}, including {examples}. Have these been formally reassigned to a new owner? To whom, and does {system} show them yet?" | per artifact or bulk group: reassigned (y/n/partial), recipient, recorded in SoR (y/n/unknown), handover done (y/n) | `artifact_discovery` re-check | records | ✔ |
| B-03 | "Are you the author of documents where you're *not* the owner, but you'd be the only one who knows why things were written the way they were?" | document refs, topics (also feeds the knowledge capture plan) | `artifact_discovery` (author role) | programme | ✔ |
| B-04 | "Do you keep working documents outside the official systems, like spreadsheets, notes, personal folders or local drives, that others rely on?" | item, location, who uses it, criticality, proposed disposition (`migrate_to_dms`/`transfer_owner`/`archive`/`destroy`) | `files.search_by_person` (OneDrive/home shares if connected) | records | ✔ |
| B-05 | "Do you hold any original records or controlled copies on paper that should go into records management?" | description, location, quantity, record series if known | manual | records | ✔ |
| B-06 | "Are there documents you know are out of date or wrong that haven't been corrected yet?" | document ref, issue, severity (also creates a `document_gap` question and, if it's a procedure/practice difference, an L item) | n/a | engineering | ✔ |
| B-07 | "Do you approve or review any document types as a required reviewer or approver?" | doc types, reviewer role, proposed replacement | `artifact_discovery` (approver/reviewer roles) | records | ✔ |

### C. Roles, authorities & designations

| Code | Ask (spoken) | Capture | Verify | Route | Req. |
|---|---|---|---|---|---|
| C-01 | "What formal roles or designations do you hold, for example system engineer of record, responsible engineer, design authority, or program owner?" | role, scope (system/program), proposed successor, reassigned (y/n), recorded where | `roles.search_by_person` (HR/CMDB/role registry) | engineering | ✔ |
| C-02 | "Do you hold any approval or signature authority, or delegations of authority, that need to be reassigned?" | authority, limit/scope, delegated by, proposed holder | `roles.search_by_person` | hr | ✔ |
| C-03 | "Are you qualified as a reviewer, screener or evaluator for any regulated processes, such as change evaluations or independent reviews? Who else holds that qualification?" | qualification, process, other qualified people, gap (y/n) | `lms.qualifications_by_person` | training | ✔ |
| C-04 | "Do you hold a position in the emergency response organization, or any on-call, duty or callout roles?" | position, rotation, backups, proposed replacement | `roles.search_by_person` (ERO roster) | licensing | ✔ |
| C-05 | "Are you a member or chair of any committees, review boards or standing meetings?" | committee, role, frequency, proposed replacement | `calendar.recurring_by_organizer`, `roles` | programme | ✔ |
| C-06 | "Are you the owner or administrator of any programme, such as a monitoring, inspection, testing or ageing-management programme?" | programme, scope, proposed owner | `roles.search_by_person` | engineering | ✔ |
| C-07 | "Do you hold any personal licences or certifications that the company relies on, such as inspector certifications or a professional engineering seal used on company work?" | credential, what relies on it, coverage plan (credentials themselves aren't transferable; record the coverage gap) | `lms.qualifications_by_person` | engineering | ✔ |

### D. Commitments & open work

| Code | Ask (spoken) | Capture | Verify | Route | Req. |
|---|---|---|---|---|---|
| D-01 | "Are you the owner of any regulatory commitments or licensing actions, or responses that are still due?" | commitment ID, description, due date, proposed owner | `commitments.search_by_person` | licensing | ✔ |
| D-02 | "Do you have open corrective actions, condition-report assignments or audit findings?" | item IDs, due dates, proposed assignee | `cap.open_by_assignee` | engineering | ✔ |
| D-03 | "Do you own any open work orders, design changes, modifications or projects?" | IDs, status, proposed owner | `cmms.open_by_assignee`, `projects.search_by_person` | engineering | ✔ |
| D-04 | "Are there promises you've made, internally or to outside parties, that someone needs to keep after you leave?" | description, to whom, due, proposed owner | manual | programme | ✔ |
| D-05 | "Are there recurring tasks only you do, such as weekly reports, monthly checks or annual submittals?" | task, frequency, next due, proposed owner (also feeds coverage) | `calendar`, `cmms.pm_by_owner` | programme | ✔ |

### E. Systems, data, accounts & automation

> **Never ask for or record passwords, PINs, keys or secret values** (SEC-040). Ask only *that* something exists and who should take it over.

| Code | Ask (spoken) | Capture | Verify | Route | Req. |
|---|---|---|---|---|---|
| E-01 | "Are you the owner or administrator of any applications, databases or tools?" | system, role, proposed owner | `cmdb.search_by_person` | iam | ✔ |
| E-02 | "Do any accounts, service accounts, scheduled jobs, scripts, macros or integrations run under your name or credentials?" | item, system, what depends on it, proposed new owner/identity | `iam.accounts_by_owner`, `cmdb` | iam | ✔ |
| E-03 | "Do you own shared mailboxes, distribution lists, team sites or file shares?" | resource, proposed owner | `iam.groups_owned_by` | iam | ✔ |
| E-04 | "Are there software licences, subscriptions or vendor portal accounts registered to you that the company relies on?" | item, vendor, renewal date, proposed holder | `contracts`, `iam` | iam | ✔ |
| E-05 | "Do you maintain code, analysis models or calculation tools that others use?" | repo/tool, users, proposed maintainer (also feeds knowledge capture) | `code.repos_by_maintainer` | engineering | ✔ |
| E-06 | "Do you hold any encryption keys, certificates or recovery methods for company systems?" *(existence only)* | system, type, proposed holder; **no values** | `iam` | security | ✔ |

### F. External relationships & representation

| Code | Ask (spoken) | Capture | Verify | Route | Req. |
|---|---|---|---|---|---|
| F-01 | "Which vendors or contractors rely on you as their main technical contact?" | vendor, topic, their contact, proposed new POC (also feeds `contact` KIs) | `contracts.search_by_person` | supply_chain | ✔ |
| F-02 | "Do you represent the company on any industry groups, owners groups, standards committees or working groups?" | body, role, term, proposed representative | manual | programme | ✔ |
| F-03 | "Are you a regular contact for any regulators, inspectors or government agencies?" | agency, topic, proposed contact | manual | licensing | ✔ |
| F-04 | "Are there outside experts, retirees or university contacts you lean on that nobody else knows about?" | person/organization, expertise, introduction proposed (y/n) | manual | programme | — |

### G. Physical items, materials & locations

| Code | Ask (spoken) | Capture | Verify | Route | Req. |
|---|---|---|---|---|---|
| G-01 | "Do you hold any keys, badges, controlled tools, test equipment or company equipment that need to be returned or reassigned?" | item, ID, proposed recipient/return | `assets.by_custodian` | facilities | ✔ |
| G-02 | "Are you custodian of any samples, specimens, check sources or other controlled materials?" | item, ID, location, proposed custodian | `assets.by_custodian` | rp | ✔ |
| G-03 | "Is there anything stored in your office, a lab, a warehouse or offsite that only you know about?" | item, location, disposition | manual | facilities | ✔ |

### H. Intellectual property & publications

| Code | Ask (spoken) | Capture | Verify | Route | Req. |
|---|---|---|---|---|---|
| H-01 | "Do you have any pending invention disclosures, patent applications or ideas you've been meaning to write up?" | title, status, co-inventors | manual | legal | ✔ |
| H-02 | "Are any papers, presentations or reports in progress that need a new lead or a company review?" | title, venue, deadline, proposed lead | manual | legal | — |
| H-03 | "Is there know-how you consider a trade secret, a method or tuning or technique, that isn't written down anywhere?" | topic only (content goes to knowledge capture with a `LEGAL-PRIVILEGED` or proprietary caveat if needed) | n/a | legal | ✔ |

### I. Training, qualification & mentoring duties

| Code | Ask (spoken) | Capture | Verify | Route | Req. |
|---|---|---|---|---|---|
| I-01 | "Do you own or teach any training courses, lesson plans or qualification materials?" | course IDs, role, proposed owner/instructor | `lms.courses_by_owner` | training | ✔ |
| I-02 | "Are you an evaluator or sign-off authority on anyone's qualification cards or task performance evaluations?" | qualification, trainees in progress, proposed evaluator | `lms.evaluator_assignments` | training | ✔ |
| I-03 | "Is anyone partway through being trained or mentored by you?" | person, topic, progress, proposed mentor | manual | training | ✔ |

### J. Continuity & post-departure

| Code | Ask (spoken) | Capture | Verify | Route | Req. |
|---|---|---|---|---|---|
| J-01 | "For your main areas, who do you think is best placed to take over, and how ready are they?" | per coverage area: suggested person, readiness (1–5), development need | n/a | programme | ✔ |
| J-02 | "After you leave, would you be open to occasional questions, through a consulting arrangement or alumni channel? How would you prefer to be contacted?" | willingness, channel, constraints (records preference; the company decides the arrangement) | n/a | hr | ✔ |
| J-03 | "Is there anything we haven't asked that you're worried will be lost or dropped when you leave?" | free text → creates knowledge questions or stewardship items | n/a | programme | ✔ |

### K. Safety concerns (confidential routing)

| Code | Ask (spoken) | Capture | Verify | Route | Req. |
|---|---|---|---|---|---|
| K-01 | "Before you go: is there any nuclear safety or quality concern you feel hasn't been fully addressed? You don't need to tell me the details. I can connect you directly and confidentially with the Employee Concerns Program." | **yes / no / prefer not to say only.** If yes: send the ECP contact info to the expert and notify ECP that a departing employee asked to be contacted (identity only with the expert's explicit agreement) | n/a | ecp | ✔ |

- **STW-020** For K-01 the platform MUST NOT store any concern content, MUST NOT run extraction on the answer turn (mark `extraction_status = skipped`, redact the turn text to the yes/no), and MUST restrict the item to `function_owner:ecp`, separate from the campaign's normal participants. If the expert starts describing the concern, the interviewer politely stops them and repeats the confidential route.

### L. Procedure–practice deviations & document defects

| Code | Ask (spoken) | Capture | Verify | Route | Req. |
|---|---|---|---|---|---|
| L-01 | "Is there anywhere the way the work is actually done differs from what the procedure says?" | document ref (and step), actual practice summary, reason, risk view; creates a `deviation` KI (10) | `dms` lookup of the doc | engineering (or CAP intake, per site) | ✔ |

- **STW-021** Every `deviation` KI (from L-01 or extracted spontaneously) creates an L item routed per site settings, which may call the CAP connector's `create_intake_draft` capability (proposal only; a human submits it). REG-013.

## 5. Item & transfer lifecycle

### 5.1 stewardship_item.status
```
pending ──ask──► asked ──answer──► captured ──┬─► needs_transfer ─► transfer_in_progress ─► verified ─► closed
                                              ├─► not_applicable (na_reason required) ─────────────────► closed
                                              ├─► routed_only (notice/obligation sent to function) ─────► closed
                                              └─► deferred (reason, new due date) ─► pending
```
- `verified` means every transfer on the item is `verified` (or `cancelled` with a reason) and the routed function has confirmed.
- `closed` additionally needs function-owner sign-off for items whose catalog entry says so.

### 5.2 stewardship_transfer.status
```
proposed → recipient_notified → recipient_accepted → sor_update_requested → sor_updated → verified
          ↘ recipient_declined (→ new proposal)          ↘ (manual SoR) sor_updated by function owner
```
- **STW-030** A transfer becomes `recipient_accepted` only when the recipient principal accepts in the platform (UI-045). An acceptance records the timestamp, any handover notes, and optional "handover briefing completed" confirmation.
- **STW-031** `verified` is set by (a) a connector re-check (09 OWN-040) showing the recipient as owner of record or assignee, or (b) a function owner's manual verification with evidence (ticket number or screenshot reference) when no connector exists.
- **STW-032** Bulk transfers: one transfer may cover a group of artifacts (for example "all 23 feedwater procedures → J. Doe") with per-artifact verification roll-up.
- **STW-033** Recipients get a **stewardship register**: everything transferred to them, with the expert's notes and links to related KIs (UI-046).

## 6. Routing

- **STW-035** Each site maps `routing_function` → a group or principal (site settings). Routed items appear in the function owner's queue (UI-047) with due dates and a notification (email or in-app, through a pluggable notifier; no external posting by agents).
- **STW-036** Escalation: items still open at `due_date` notify the programme manager. Items still open 5 working days before the departure date are flagged red on the programme dashboard.

## 7. Closure gate & attestation

- **STW-040** A campaign can move from `closing` to `closed` only when every **required** stewardship item is `closed` (verified, not applicable with reason, or routed-only), **or** the programme manager records an override with a justification. The override is audited.
- **STW-041** **Exit attestation:** in the `exit` session (or by form), the expert confirms a generated statement listing what was captured, what was reassigned and to whom, and what remains open with whom. They can add corrections. The attestation is stored as an immutable PDF + JSON, labelled, with the expert's electronic acknowledgement (no wet signature required unless the site configures it).
- **STW-042** A **Stewardship Report** per campaign is exportable (REG-007): items by category, status, recipients, verification evidence, open items with owners.

## 8. Acceptance criteria (selection)

- Given an active campaign with the default catalog, when the campaign is activated, then one stewardship item per applicable required catalog entry exists with status `pending`.
- Given a discovered artifact with role `owner` and a confirmed match, when discovery completes, then a `holding` item in category B exists, linked to the artifact.
- Given a transfer accepted by the recipient and a connector re-check showing the recipient as owner, then the transfer is `verified` and the item rolls up to `verified`.
- Given K-01 answered "yes", then no turn text beyond "yes" is stored, extraction is skipped, the ECP queue receives a contact request, and campaign participants can't see the item.
- Given any required item not closed, when the facilitator tries to close the campaign, then the API returns `409 CLOSURE_BLOCKED` with the list of blocking items.

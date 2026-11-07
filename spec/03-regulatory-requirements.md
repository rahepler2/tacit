# 03 · Regulatory & compliance requirements

> These requirements turn regulatory constraints into product behaviour. They are not legal advice. Each deploying customer's counsel and export-control officer confirm the applicable regimes per site. Implementers build the **mechanisms**; customers configure the **policy content**.

## 1. Regimes the platform must be able to support

| Regime | Platform implication |
|---|---|
| US 10 CFR Part 810 (DOE): assistance to foreign atomic energy activities, including **deemed exports** to foreign nationals | Label-based release control by principal citizenship and authorization; no release without a policy permit |
| US 10 CFR Part 110 (NRC), US EAR (15 CFR 730–774), ITAR where relevant | Configurable export classifications per resource |
| EU Dual-Use Regulation (EU) 2021/821, Annex I Category 0 | Same mechanism, EU classifications |
| UK, Canadian and other national export regimes | Same mechanism, per-site classification vocabularies |
| Safeguards Information (US 10 CFR 73.21) and national security-related information regimes | Red-zone: never stored (06 §6) |
| QA programmes: US 10 CFR 50 App. B / ASME NQA-1, ISO 19443, IAEA GSR Part 2 | Informational positioning, traceability, versioning, review records |
| Cyber security: US 10 CFR 73.54 / NEI 08-09 and national equivalents | Business-network deployment only; no plant-network connections |
| GDPR / UK GDPR, other privacy laws (PIPEDA, PIPL, DPDP, …) | Lawful basis records, DPIA support, data-subject rights, residency |
| EU AI Act | No emotion recognition at work (Art. 5(1)(f)); stay out of employment decision-making (Annex III §4); transparency to people interacting with AI |
| Works-council co-determination (DE BetrVG §87(1) no. 6, FR CSE, NL WOR, …) | Configurable consent text, recording controls, data-use restrictions per site |
| Employee-concerns and safety-conscious work environment expectations | Confidential routing of safety concerns to ECP; never stored in the knowledge base (08 category K) |

## 2. Requirements

### Export control & release

- **REG-001** Every stored resource that carries knowledge content (turn, transcript segment, KI and its versions, document and chunk, edge with evidence, question, stewardship item free text, agent output) MUST have a `label` (05 §4) before it is persisted.
- **REG-002** New content MUST get the site's **default pessimistic label** (configured per site; for example "presumed controlled, unreviewed") until a principal with the `classifier` role confirms or changes it.
- **REG-003** Access to any labelled resource MUST be decided by the PDP (06) from principal attributes (citizenships, authorizations, site, employer, groups, clearances), resource label and context. No code path may bypass it, including search, MCP tools, exports, agent context-building and notifications.
- **REG-004** Search and retrieval MUST filter by authorization **before** ranking or counting. Unauthorized users MUST NOT learn that unauthorized results exist (no counts, no "N results hidden"), unless the policy explicitly allows a "request access" affordance.
- **REG-005** Every model call MUST go through the model router (04), which MUST refuse to send content whose label isn't permitted for that model endpoint (endpoint allow-lists by label).
- **REG-006** Every release of labelled content to a principal (view, answer, export, MCP response) MUST produce an audit event that records the resource IDs and labels involved.
- **REG-007** Exports (PDF, DOCX, JSON) MUST carry visible label banners, MUST be watermarked with the requesting principal and timestamp, and bulk exports (> configurable N resources) MUST require dual approval.
- **REG-008** Cross-site transfer of content MUST be treated as a release and evaluated by policy. Each site's data MUST stay in that site's storage unless policy allows replication.

### QA & traceability

- **REG-010** Every KI MUST keep its full version history, including the original machine extraction, with author (human or agent), timestamp and reason for each change.
- **REG-011** Every KI MUST trace to at least one source span (turn range or document anchor).
- **REG-012** The UI and every twin answer MUST show an "informational, not a controlled procedure" notice, and MUST cite the controlled document when one is linked.
- **REG-013** A KI of type `deviation` (procedure ≠ practice) MUST create a routed stewardship or CAP item (08 category L) for the configured function owner. The platform MUST NOT silently store deviations without routing them.
- **REG-014** Review decisions (approve, reject, edit, classify) MUST be recorded with reviewer identity, role and timestamp, and are immutable.

### Privacy & labour

- **REG-020** No session may record or process audio or text before a **consent record** exists for that session's participants, with the site-specific, versioned consent text (07 §3).
- **REG-021** Audio retention MUST be configurable per site (default: delete raw audio 30 days after the transcript is approved), with legal-hold override.
- **REG-022** The expert MUST be able to review, correct and request redaction of transcripts and KIs attributed to them before release. Redaction removes text from all versions and search indexes and keeps a tombstone with the reason.
- **REG-023** Data-subject access and erasure requests MUST be supported for personal data (transcripts, quotes, audio, profile). Knowledge content already released MAY be kept in de-attributed form where the site's policy allows.
- **REG-024** The platform MUST NOT compute or store emotion, sentiment, stress, deception or performance scores about any person, and MUST NOT store voiceprints.
- **REG-025** Every AI interviewer session MUST start with a disclosure that the interviewer is an AI system, and the interviewer MUST never claim to be human.
- **REG-026** Platform data MUST NOT be offered for employment decisions. The admin UI MUST NOT provide per-person productivity rankings.

### Cyber

- **REG-030** The platform MUST be deployable fully offline (air-gapped): no runtime calls to the public internet, all models and assets local.
- **REG-031** Connectors MUST be read-only by default. Write-back (09 §7) needs explicit per-connector enablement and per-action human approval.
- **REG-032** The platform MUST NOT connect to plant control or monitoring networks or historians.

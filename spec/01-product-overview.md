# 01 · Product overview

## 1. Problem

Nuclear operators run plants licensed for 60–80 years with workforces whose most experienced people are retiring. What these experts know only partly lives in controlled documents. The rest is the reasons behind design choices, the cues that something is about to go wrong, the deviations between procedure and practice, who to call, and which vendor engineer actually understands the 1987 modification. When they leave, three things go with them:

1. **Tacit knowledge** that was never written down.
2. **Stewardship**: ownership of documents, systems, commitments, agreements, relationships and designated roles. Unless someone explicitly reassigns these, they become orphaned. A procedure with a retired owner goes unrevised. A vendor NDA with no custodian has nobody controlling the proprietary data received under it. An emergency-response position goes unfilled.
3. **Context** linking the two: which documents embody which knowledge, and where they're wrong or incomplete.

## 2. Goals

| ID | Goal |
|---|---|
| G1 | Capture an expert's operational knowledge across **multiple sessions** (typically 4–12) until it is measurably saturated. |
| G2 | Guarantee that every departing expert completes the **Stewardship & Obligations** track (08), verified against systems of record (09). |
| G3 | Connect knowledge to the document corpus and enterprise records through typed, cited relationships. |
| G4 | Make captured knowledge usable by successors from the platform UI and from AI assistants (MCP), with every answer grounded and cited. |
| G5 | Enforce export control, need-to-know, privacy and QA boundaries on every read, write, model call and integration. |
| G6 | Give programme managers a risk-weighted view of knowledge-loss exposure across the whole workforce. |

## 3. Non-goals (v1)

- **Not a QA-controlled system** for performing safety-related work. Content is informational and points users to controlled procedures.
- **Not a document management system, HR system or contract system.** The platform *reads* systems of record and *proposes* changes to them. It does not replace them.
- **Never captures** Safeguards Information, classified information, physical-security details or credentials (06 red-zone).
- **No emotion, sentiment, stress or performance inference** about any person.
- **No voice cloning or biometric voiceprints.**
- Not a general enterprise search engine. Document ingestion serves knowledge capture and linking.

## 4. Personas

| Persona | Needs |
|---|---|
| **Expert** (retiring engineer, operator, technician, manager) | A respectful, efficient interview; control over what's published under their name; a clear list of what they still need to hand over. |
| **Successor** (named individual or team) | Fast, trustworthy answers with sources; the ability to ask the expert follow-up questions through the platform; a stewardship register of what they inherited. |
| **Facilitator** (knowledge-management staff) | Schedules campaigns and sessions, can sit in or drive sessions, monitors progress. |
| **SME reviewer** | Reviews knowledge items for technical accuracy. |
| **Derivative classifier / export-control reviewer** | Assigns or confirms export and sensitivity labels; approves releases. |
| **Programme manager** | Risk-weighted coverage across all campaigns; stewardship completion; reporting to management. |
| **Function owners** (Legal, Records, IT/IAM, Security, Training, Licensing, Supply Chain, HR) | Receive routed stewardship items in their area and close them. |
| **Assistant user** | Any authorized employee using an AI assistant that calls the platform's MCP tools. |
| **Administrator / auditor** | Configuration, connectors, audit review. |

## 5. Core user journeys

1. **Programme setup:** import the knowledge-loss risk register and HR departure dates → the system proposes campaigns ranked by risk → the programme manager approves them.
2. **Campaign kickoff:** the expert gives consent → the ownership-discovery agent searches every connected system for artifacts the expert authored or owns → a pre-research dossier is built → the coverage plan is generated from job tasks plus dossier gaps → sessions are scheduled.
3. **Interview sessions:** the AI interviewer runs each session type (07). Knowledge items appear live. Stewardship questions are woven in and also run as a dedicated session (08).
4. **Between sessions:** agents generate follow-ups, propose relationships, detect contradictions, prepare the next agenda. The expert reviews and approves what's published under their name.
5. **Review & release:** SME review, then export-control classification, then release to the audiences the label policy allows.
6. **Stewardship closure:** every stewardship item has a recipient, the recipient accepts, the system of record is updated, the connector re-check verifies it → exit attestation.
7. **Use:** successors ask the twin (UI or MCP). Unanswered questions go back to the expert while they're still employed, or later through the post-departure channel if one is agreed.

## 6. Success metrics (programme level)

| Metric | Definition |
|---|---|
| Stewardship closure rate | % of campaigns where every required stewardship item is *Verified* or *Closed with reason* before the departure date |
| Orphaned-artifact count | Artifacts in connected systems whose owner of record is departed or inactive |
| Risk-weighted coverage | 14 §4 |
| Twin answerability | % of held-out successor questions answered with grounded citations (19) |
| Successor usefulness | Periodic rating by successors (1–5) |
| Reviewer load | Reviewer hours per interview hour |
| Time-to-saturation | Sessions until the new-knowledge rate drops below threshold (14) |

# 13 · Background agents

## 1. Agent framework

- **AGT-001** An agent is a pg-boss job type with: a name, triggers, a **tool allow-list** (connector capabilities and internal read functions), a model-call budget (max calls, max tokens), a timeout, and an **output contract**: the proposals it may create.
- **AGT-002** Agents run an LLM tool-use loop (max 12 iterations by default) through the model router. Tool results enter prompts as untrusted data (SEC-070).
- **AGT-003** Agents never change released content or external systems directly. They write **proposals** (KI, edge, question, stewardship item, identity match, label suggestion, agenda) with `created_by_kind = agent` and the `agent_run` ID.
- **AGT-004** Each run records an `agent_run` with inputs, the model endpoints used, token counts, outputs created, duration and errors. Runs are visible in the ops UI. Failures are retried twice with backoff and then surfaced.
- **AGT-005** Agents run with a **service principal scoped to the campaign** (or programme for programme-level agents). Context-building applies the PDP as if the campaign facilitator were the reader, and the router enforces endpoint labels.

## 2. Agent catalog

| ID | Agent | Trigger | Tools | Produces |
|---|---|---|---|---|
| AGT-010 | **Dossier builder** | campaign activated; discovery done; weekly | artifacts, document_search, work_items, role_registry, retrieval | Campaign dossier (markdown + structured): career timeline guesses, top systems and equipment by activity, authored and owned documents by type, open items, notable condition reports, suggested coverage areas and document walkthrough candidates (ranked by: owned and active, high criticality, many revisions by the expert, few other authors) |
| AGT-015 | **Ownership discovery** | campaign activated; T−10 days; manual | artifact_discovery, person_directory | artifacts, identity alias proposals, holding items (09) |
| AGT-020 | **Follow-up refiner** | session processed | retrieval, KIs, turns | Ranked `follow_up` questions: dedups the in-session follow-ups, adds missing-value, missing-rationale and missing-cue probes, and checks against existing KIs so already-answered items aren't asked again |
| AGT-030 | **Agenda planner** | session scheduled; previous session processed | questions, coverage, stewardship items, dossier | `SessionAgenda` proposal for the next session (07 §7), balancing coverage gaps, queued successor questions, due stewardship items and time budget |
| AGT-040 | **Relationship tagger** | KI created or edited; document ingested | retrieval, entity registry, document_search | Edge proposals (10 §6) and mention proposals, each with evidence |
| AGT-045 | **Contradiction detector** | KI released; nightly | retrieval across campaigns | `contradicts` edges and `contradiction` questions directed to the relevant expert(s) |
| AGT-050 | **Classification assistant** | KI enters `classification` | KI, linked documents, site classification guide (uploaded document) | Label suggestion with rationale and cited guide sections. **Advisory only**; the classifier decides |
| AGT-055 | **Review batcher** | daily | KIs in review | Groupings of similar KIs for batch review; merge proposals |
| AGT-060 | **Staleness watcher** | document_revisions poll (default hourly) | document_revisions, document_content, KIs linked to the document | Re-ingests the new revision; for each KI with `cites`/`supplements`/`corrects`/`deviates_from` edges to changed sections, a `stale_after_revision` question and a flag on the KI |
| AGT-065 | **Session summarizer** | session ended | turns, KIs | Session summary and per-area summary updates (07 INT-060) for expert approval |
| AGT-070 | **Coverage assessor** | session processed; nightly | KIs, coverage areas, eval results | Coverage recomputation inputs (14 §3) and saturation signals |
| AGT-075 | **Twin evaluator** | nightly; on demand | evals | `eval_run` with metrics (19 §4) |
| AGT-080 | **Orphan sweeper** | weekly | artifact_discovery, person_directory | Orphaned-artifact list (09 OWN-050) |
| AGT-085 | **Stewardship reminder** | daily | stewardship items | Notifications per STW-036 (via the notifier interface; no external posting) |

## 3. Output quality rules

- **AGT-090** Every proposal includes `evidence` (quotes or tool-result excerpts with source IDs), so reviewers can judge it without re-running the agent.
- **AGT-091** Agents MUST NOT propose questions already answered (check retrieval with similarity ≥ 0.85 against released KIs) or already open (dedupe against open questions).
- **AGT-092** Agents MUST respect red-zone rules: never propose questions that seek red-zone content. Proposals go through the red-zone stage-1 detectors before they're stored.

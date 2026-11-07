# Tacit → Nuclear Knowledge Retention Platform

**Research and production-readiness assessment** · 2026-11-07
**Target:** an international nuclear operator capturing the knowledge of experts who retire soon, across many sessions, with MCP-based research, follow-ups and document-relationship tagging.
**Status:** research only. No product code was changed. Baseline verified: `pnpm typecheck` is clean and `pnpm test` passes 98 tests (core 33, API 9, web 56).

> Regulatory notes below are engineering input for counsel and your export-control officer. They are not legal advice. Check every citation against the current primary text before you rely on it.

---

## 1. Executive summary

Tacit is a well-built **hackathon-grade prototype**: about 8.7k lines of strict TypeScript, a clean isomorphic engine, a good interview loop and a working "closed loop", where questions the successor can't get answered become the next interview's agenda. The *concepts* are right for nuclear knowledge retention. Almost everything *around* those concepts has to change before a nuclear operator can use it:

| Area | Today | Needed for a nuclear operator |
|---|---|---|
| Deployment | Single-tenant Node and SQLite, or fully in-browser | On-prem or sovereign cloud per jurisdiction, air-gap capable |
| AI providers | Nebius (Qwen), Boson Higgs (cloud voice), OpenAI or Anthropic over the public internet | Models inside the security boundary (on-prem vLLM, or a government or sovereign cloud region); no audio leaves the boundary |
| Identity | **None**, with `cors()` open to all | SSO (OIDC/SAML), RBAC plus **attribute-based access** (export class, nationality, site, clearance) |
| Unit of work | One expert → one successor | A program-wide portfolio: hundreds of experts, many successors, cross-expert graph |
| Knowledge sources | Interview speech only | Interviews **plus** the document corpus (procedures, condition reports, work orders, design basis, drawings) |
| Relationships | Tag-regex entities and BM25 "related" links | Typed, provenance-bearing edges between atoms, documents, equipment, people and tasks |
| Coverage | LLM-guessed 6–8 domains | Grounded in the plant's **Job/Task Analysis** (SAT/ACAD) and IAEA knowledge-loss risk scoring |
| Quality | `verified: boolean` | Multi-stage review (expert → peer SME → owner), versioning, QA-program alignment |
| Integrations | None | MCP server (Tacit as a tool for assistants) and MCP client (Tacit pulls from DMS, CMMS, CAP, HR) |
| Audit | None | Immutable audit log of every access, answer and export |
| Gimmicks | Voice cloning, hand-gesture 3D graph | Remove cloning (biometric, deepfake and insider-threat risk); keep the graph as a 2D analytic view |

**Recommendation:** treat this repo as an **executable requirements document**. Spend 2–4 weeks building a "Phase 0" pilot on it: harden it, add documents and MCP, run with 2–3 friendly retirees on non-sensitive topics. Use what you learn to write the spec (section 10). Then do the rewrite on an architecture built for multi-tenancy, attribute-based access and export control from day one. Retrofitting attribute-level export control into the current JSON-blob store is not worth it.

On the "clean room" point: the repo is **MIT-licensed** ("Tacit contributors"), so you don't need a clean-room process for legal reasons. Commercial use, modification and relicensing your derivative are allowed as long as you keep the notice. A rewrite is still the right *technical* call. If your company's IP or procurement policy requires provenance hygiene, a spec-first rewrite gives you that cheaply.

---

## 2. What the codebase actually is

### 2.1 Layout

```
packages/core/   engine (isomorphic TS): types, Engine, Brain (LLM + heuristic), retrieval, coverage, graph, handover, prompts, role templates
apps/api/        Hono + node:sqlite REST; provider auto-detect; Boson Higgs (voice) and LiveKit (phone) glue
apps/web/        Vite + React 19; interview room, twin (Ask), knowledge base, coverage, 3D "Constellation", standalone in-browser mode
services/phone-agent/  Python LiveKit Agents worker: Twilio SIP → Higgs Realtime → Tacit API
```

### 2.2 Domain model (`packages/core/src/types.ts`)

```
Capture (one expert)
 ├─ expert {name, role, team, tenureYears, departureDate, language}
 ├─ successor {name, role}            ← single successor
 ├─ domains[]  {priority 1-3, targetQuestions[], askedCount, atomCount, coverage}
 ├─ Session[]  {mode: voice-higgs|voice-browser|text, summary}
 │    └─ Turn[] {role, text, kind, domainId, questionId, extractedAtomIds}
 ├─ Atom[]     {type ∈ 9 types, title, content(md), tags[], confidence, verified, sourceQuote, turnId}
 ├─ Question[] {source: plan|successor|gap|followup, status: open|asked|answered}
 └─ embeddings atomId → number[]
```

### 2.3 Key mechanisms and how good they are

| Mechanism | Where | Assessment |
|---|---|---|
| **Interview loop.** Extraction and next-question planning run in parallel (`Promise.all`) | `engine.ts` `expertTurn` | Good design. It keeps latency low (about 3.5 s per turn). |
| **Interviewer persona.** Follows up on hedges ("usually", "depends"), bans empty praise, under 55 words per turn | `prompts.ts` `INTERVIEWER_PERSONA` | Strong. The best asset in the repo. Reuse it. |
| **Multiple sessions.** Already supported: N sessions per capture, a `memoryBrief` of the last **8** atom titles, unanswered successor questions re-opened at `endSession` | `engine.ts` | The concept is right, but memory is shallow. A 10-session campaign needs real cross-session memory (section 5.3). |
| **Closed loop.** A low-confidence twin answer queues the question; the next session asks it first | `engine.ts` `ask()` | The killer feature. Keep it and generalize it (section 5.4). |
| **Coverage.** `0.55 × asked + 0.45 × min(1, atoms/5)` | `coverage.ts` | A heuristic: it counts activity, not knowledge. Five shallow atoms read as "covered". Replace it (section 5.5). |
| **Retrieval.** BM25 plus embeddings, RRF-fused | `retrieval.ts`, `text.ts` | Correct, but O(N) per query: it loads every atom and fetches each embedding one at a time (N+1), and rebuilds BM25 on every call. Fine for 200 atoms, not for 500k. |
| **Twin confidence.** Self-reported by the LLM, or `strength` for the heuristic brain | `brain.ts` | Needs a grounding verifier (claim-level entailment against cited atoms) before anyone trusts it in a plant. |
| **Graph.** Entities = tags classified by regex (capitalized = person, CAPS = tool); "related" = BM25 neighbours | `graph.ts` | A demo. Nuclear needs entity resolution against the equipment master, the procedure index and the HR directory. |
| **Heuristic fallback brain.** Every LLM failure silently degrades | `brain.ts` | Right for a demo, **wrong for production**: silent quality drops corrupt a knowledge base. Fail loudly and retry from a queue. |
| **Persistence.** JSON documents in SQLite, embeddings as JSON text | `apps/api/src/sqlite.ts` | No transactions around a turn, no versioning, no tenancy. |
| **Voice.** Higgs Realtime speech-to-speech, browser Web Speech, Higgs TTS and cloning | `apps/web/src/lib/voice/*`, `apps/api/src/higgs.ts` | Engineering quality is high (careful handling of transcript revisions), but every audio path goes to Boson's cloud. Not acceptable for this customer. |

### 2.4 Specific defects worth knowing before the pilot

1. **No authentication or authorization anywhere.** `app.use("/api/*", cors())` allows every origin. Anyone who can reach the port can read, export or delete everything.
2. **Standalone mode stores provider keys in `localStorage`** and sends `anthropic-dangerous-direct-browser-access`. Disable this in any enterprise build.
3. **`expertTurn` isn't atomic.** It writes the turn, then the atoms, then the question, then the session, then the capture, with `refresh()` called several times per turn. Concurrent posts (the Higgs and phone paths post asynchronously) can race on `askedCount` and `turnCount`.
4. **Embeddings are fire-and-forget** (`void ensureEmbeddings(...)`), and failures are swallowed. Atoms can silently go without vectors. `updateAtom` deletes the vector before it re-embeds.
5. **No atom history.** Edits overwrite in place. You lose the original extraction, which you need for QA traceability.
6. **The successor question is closed as "answered"** whenever *any* atom comes out of the next expert turn, whether or not it actually answers the question.
7. **The interviewer persona says "never claim to be human"** (good), but there is no recorded consent step and no disclosure at the start of a session.

---

## 3. The nuclear context: requirements this product must meet

### 3.1 Why this market is real

Nuclear has the textbook knowledge-loss profile: long-tenured staff (30 years or more), plants licensed for 60–80 years with original-design knowledge retiring, and a regulator that expects knowledge to be managed. The IAEA has published on this for 20 years. Its **Knowledge Loss Risk Management** methodology (IAEA Nuclear Energy Series **NG-T-6.11**, 2017, which updates the 2006 STI/PUB/1248) scores every position as:

```
Total risk = Attrition risk factor (how soon the person leaves, 1–5) × Position risk factor (how hard the knowledge is to replace, 1–5)
```

Most nuclear operators already run some version of this (often descended from TVA's and INPO's practice). **This is your prioritization engine.** Tacit should ingest the KLRA spreadsheet and HR retirement dates, then schedule interview campaigns by risk score. That ties the product to a process the customer already funds.

### 3.2 Export control: the requirement that shapes the whole architecture

The knowledge these experts hold is very likely **controlled technology**. Interviewing them and making the result queryable *creates and distributes technical data*.

| Regime | Relevance |
|---|---|
| **US 10 CFR Part 810** (DOE) | Controls assistance and technology transfer for foreign atomic-energy activities, **including "deemed exports"**: releasing technology to a foreign national inside the US counts as an export to their home country. The 2015 rule added a general authorization (§810.6(b)) for foreign nationals lawfully employed by a US employer, depending on the country. DOE does not accept a de minimis threshold. **The twin answering a successor is a release of technology.** |
| **US 10 CFR Part 110** (NRC) | Exports of nuclear equipment and material. The scope of Part 810 is tied to Part 110 appendices. |
| **US EAR** (15 CFR 730–774) | Category 0 and dual-use items and technology (for example the 0E and 2E series). |
| **EU Dual-Use Regulation (EU) 2021/821** | Annex I Category 0 (nuclear). Intra-EU transfers of Annex IV items still need authorization. |
| **UK** Export Control Order 2008; **Canada** Controlled Goods and NSCA export rules; others per site | Each site's jurisdiction applies to its own data, and transfers *between* your sites are exports. |
| **Safeguards Information** (US 10 CFR 73.21) and national equivalents for security-sensitive information | Must generally **never** enter a system like this. The interviewer must steer away from it, and the extractor must detect and quarantine it. |
| **Classified information** | Out of scope. Design so that it can't happen (section 3.6). |

**Design consequences** (these are product features, not paperwork):

- **Every atom, document and edge carries a classification label**: export jurisdiction and class (for example `US-810`, `EAR-0E001`, `EU-0E001`, `NLR`), a sensitivity tier, an origin site and an owner.
- **Every principal carries attributes**: citizenship(s), work location, employer, authorizations held, clearance, and need-to-know groups.
- **Access is decided by policy (ABAC) at query time**, in a single policy decision point such as OPA/Rego, Cedar or Zanzibar-style ReBAC plus attributes. Retrieval filters *before* ranking, so a non-authorized user never sees the existence of a hit, and the LLM never sees text the user isn't allowed to see.
- **The default label is pessimistic.** New atoms arrive "unreviewed, presumed controlled" until a derivative classifier (a trained export-control reviewer) clears them. The UI and workflow must support that reviewer as a first-class role.
- **The LLM is part of the export boundary.** Sending controlled technical data to a model hosted abroad, or operated by foreign-national staff, can itself be an export. That rules out most public LLM and voice APIs, including the current Nebius and Boson defaults. See section 6.
- **Successor-to-expert question routing must respect labels too.** A queued question can leak information ("Why does pump 2B cavitate at 85 % power?").

### 3.3 Quality assurance

- **US 10 CFR 50 Appendix B** and **ASME NQA-1** (and international equivalents such as IAEA GSR Part 2 and ISO 19443): if the knowledge base is used to *perform* safety-related work, it becomes a QA-controlled system. Avoid that classification in v1: position the twin as **"informational; not a substitute for approved procedures"**. Show a banner, cite the *controlled procedure* whenever one exists, and never let an atom contradict a procedure without flagging it.
- **Procedure-versus-practice gaps are gold, and also a liability.** When the expert says "the procedure says X, but we actually do Y", that is a potential **condition report**. The system must route it to the Corrective Action Program workflow (via an MCP connector), not just store it as a "gotcha". Agree this with the licensing and CAP owners before the pilot.
- **Records retention:** interview recordings and transcripts may be quality records or HR records, depending on use. Define a retention schedule per jurisdiction.

### 3.4 Privacy, labour and AI regulation (international employees)

| Concern | Implication |
|---|---|
| **GDPR / UK GDPR** | Voice recordings and transcripts are personal data. Use a lawful basis of legitimate interest or contract (consent is shaky in an employment relationship), complete a DPIA, and honor data-subject rights. Atoms are *company knowledge*, but source quotes and audio stay personal data. |
| **Voice cloning** | Voiceprints are biometric data (GDPR Art. 9 special category) and create deepfake and **insider-threat risk at a nuclear site** (social engineering by phone in a retired engineer's voice). **Remove it.** |
| **EU AI Act** | Art. 5(1)(f) **prohibits emotion recognition in the workplace** (with medical and safety exceptions). Do not add sentiment, stress or hesitation scoring of experts, even as "interview analytics". Systems used for employment decisions, task allocation or performance monitoring fall under Annex III point 4 (high risk). Keep Tacit clearly out of any performance evaluation, and contractually prohibit that use. Check the current application dates (the Digital Omnibus may have moved the Annex III deadlines). |
| **Works councils** | Germany (BetrVG §87(1) no. 6: co-determination over technical devices able to monitor employees), France (CSE consultation), the Netherlands and others. Recording interviews needs a works agreement. Budget months for it. |
| **Other residency regimes** | China's PIPL, India's DPDP, Canada's PIPEDA and provincial laws, and others where you have staff. Cross-border transfer of transcripts is both an export-control *and* a privacy transfer. |
| **Disclosure and consent** | A recorded, versioned consent at the start of every session (AI interviewer, recording, purpose, retention, who can see the result), and an expert right to review and redact before publication. |

### 3.5 Cybersecurity

- US plants: **10 CFR 73.54** and **NEI 08-09** apply to critical digital assets. Tacit must live on the **business (corporate) network**, never on plant networks, and must never connect to plant data historians. Equivalents elsewhere: IAEA NSS 17-T, UK ONR CAF-based guidance, Canada CSA N290.7.
- Expect a full vendor security assessment, a pen test and SBOM review. The current dependency tree (React 19, three.js, MediaPipe WASM, LiveKit, Boson) has to shrink a lot.
- Insider threat: the knowledge base becomes a **concentrated, high-value target**, effectively a curated map of how the plant really works. Use rate limits, anomaly detection on bulk retrieval, watermarked exports, and no bulk export without dual approval.

### 3.6 Information that must never be captured

Build a **"red-zone" filter** into both the interviewer prompt and a dedicated classifier pass on every expert turn:

- Safeguards and physical-security details (guard posts, response times, alarm logic, access codes)
- Classified information
- Credentials and passwords (experts *will* say them out loud; the current README even celebrates "the token only they hold")
- Third-party personal data beyond business contact information

On a hit: redact the stored transcript span, keep a hash and a pointer for the security officer, have the interviewer steer away politely, and raise an alert.

---

## 4. Target product: what "extensive and exhaustive, multi-session" means

### 4.1 Programme-level hierarchy (replaces the single `Capture`)

```
Tenant (company)
 └─ Site / Business unit (jurisdiction, data residency)
     └─ Knowledge Programme (for example "2027–2030 retirement wave")
         ├─ Risk register (KLRA: position × attrition), imported from HR and the existing KLRA sheet
         ├─ Competency model (Job/Task Analysis per position, from SAT/ACAD training programmes)
         └─ Expert Campaign (one per retiring expert)
             ├─ Successors[] (many: named successor, team, future hires, "the organization")
             ├─ Pre-research dossier (from documents via MCP)
             ├─ Coverage plan = JTA tasks + KLRA critical knowledge + LLM-proposed gaps
             ├─ Sessions[] (types below), scheduled over weeks or months
             ├─ Atoms (versioned, labelled, reviewed)
             ├─ Edges (atom↔atom, atom↔document, atom↔equipment, atom↔person, atom↔task)
             └─ Questions (from successors, gaps, follow-ups, reviewers, contradictions)
```

### 4.2 Session types

| Type | Purpose | Notes |
|---|---|---|
| **Kick-off / career map** | Timeline of roles, systems and events. Seeds entities and stories. | Long-form narrative; good for decisions ("why it's like this"). |
| **Domain deep-dive** | One coverage area per session (for example "feedwater control, transients") | The current Tacit loop, but scoped and pre-briefed. |
| **Document walk-through** | The expert annotates *their own* procedures, calcs and condition reports: "what's missing from this?" | The highest-yield session type for nuclear. Needs document viewing and anchored comments. |
| **Scenario / "what if"** | Walk through abnormal and emergency events (non-SGI) and past incidents | Cognitive-task-analysis style (Critical Decision Method). Elicits judgement cues. |
| **Successor-led** | Successors ask their own questions live; Tacit takes notes and extracts | Turns the expert into a mentor and builds relationships. |
| **Field walk-down** | Mobile, offline-capable; photos and voice notes tagged to equipment IDs | Needs on-device STT and a camera policy (sites restrict cameras). |
| **Validation** | The expert reviews atoms from earlier sessions: confirm, correct, add | Closes the QA loop. Usually 20–30 % of total time. |
| **Group / panel** | Two or three experts on a shared topic; contradictions surface | Speaker diarization required. |
| **Exit / handover** | Final gaps, open questions, who-to-call after retirement (consulting availability) | Produces the signed handover package. |

### 4.3 What "exhaustive" can be measured against

You can't prove completeness, but you can make the gaps **visible and auditable**:

1. **Task coverage:** percentage of the position's JTA tasks with at least one reviewed procedure or decision atom, and for each critical task, whether failure modes, cues and contacts were captured.
2. **Document coverage:** percentage of documents the expert authored or owns that have been walked through. Each document gets a status: confirmed complete, gaps captured, or not reviewed.
3. **Entity coverage:** every system, component, vendor and person the expert mentions frequently has a profile with enough atoms.
4. **Question closure:** successor questions opened, answered and still open; answer latency.
5. **Twin evaluation:** a held-out question set (written by successors and supervisors), scored for answerability and grounding, re-run after every session. "Exhaustive" becomes a trend line, not a claim.
6. **Saturation:** new atoms per session hour falling, plus a rising duplicate/corroboration rate, show diminishing returns. That's when a campaign is "done".

---

## 5. MCP and agent design

MCP is the right integration seam. Tacit plays **two roles**.

### 5.1 Tacit as an MCP **server** (the knowledge base becomes a tool for every assistant)

Expose a governed tool surface so successors can query captured knowledge from whatever assistant the company standardizes on (Claude, Copilot, internal chat), with Tacit's policy engine enforcing labels on every call.

| Tool | Purpose |
|---|---|
| `search_knowledge(query, filters)` | Hybrid retrieval over atoms *and* documents, already filtered by ABAC |
| `ask_expert_twin(expert_or_topic, question)` | Grounded answer plus citations plus confidence. Queues the question on low confidence. |
| `get_atom(id)` / `get_document(id)` | With provenance: session, turn, quote, reviewer, version |
| `get_neighbors(node_id, edge_types, depth)` | Graph traversal (equipment → procedures → gotchas → people) |
| `list_open_questions(campaign)` / `add_question(...)` | Feed the closed loop from anywhere |
| `who_knows(topic)` | Expert finder across the whole programme (living and retired) |
| `get_coverage(campaign)` | Coverage and gap report |

Resources: atoms and documents as `tacit://` URIs. Prompts: "pre-brief me on system X", "draft a turnover note".

Auth: implement the MCP HTTP authorization spec (OAuth 2.1, Protected Resource Metadata per RFC 9728) behind your IdP, and use **per-user tokens, never a shared service token**, so ABAC and the audit log see the real person. The MCP spec version in use was 2025-11-25 at the time of writing; a newer revision may exist, so pin a version and track the changelog.

### 5.2 Tacit as an MCP **client** (research, follow-ups and relationship tagging)

Background agents run *between* sessions, each a bounded job with an audit trail.

| Connector (MCP server you write or adopt) | Typical system at a nuclear operator | Used for |
|---|---|---|
| Document management | OpenText Documentum, SharePoint, Hexagon/Intergraph SmartPlant, Bentley ProjectWise | Procedures, calcs, design basis documents, drawings, revisions |
| Work management / CMMS | IBM Maximo, SAP PM | Work orders and equipment history the expert worked on |
| Corrective action programme | CAP tools (often Maximo or custom) | Condition reports authored by or involving the expert; routing procedure-vs-practice gaps |
| Equipment master | Equipment databases (EIIS-style tag IDs) | Entity resolution: "the 2B pump" → `2-FW-P-2B` |
| Training / SAT | LMS and JTA databases | Coverage model per position |
| HR | Workday, SAP SuccessFactors | Retirement dates, successors, org chart, citizenship attributes (via IdP, not raw) |
| Collaboration | Teams or Outlook calendar | Scheduling sessions; meeting-recording ingestion where permitted |
| Operating experience | INPO, WANO and internal OE databases (licensing permitting) | Context for scenario sessions |

**Agents (each one an LLM tool-use loop over the MCP connectors, with outputs written as *proposals* for a human to accept):**

1. **Pre-research agent (before each session).** Builds a dossier: documents the expert authored, condition reports they raised, equipment they touched most, gaps between their documents and existing atoms. It proposes the session agenda: "You wrote OP-3-FW-004 rev 2 in 2009; rev 6 removed step 4.3. Ask why." This is what makes sessions *extensive* rather than generic.
2. **Follow-up agent (after each session).** For every atom, finds hedges, unstated conditions, missing numbers, vague references and contradictions with documents or other experts, then emits ranked follow-up questions into the queue for the next session. This generalizes the current "followup" `QuestionSource`, which exists in the types but is barely used.
3. **Relationship-tagging agent (continuous).** For each new atom it does three things:
   - **Entity resolution:** links mentions to canonical equipment, procedures, people and vendors (fuzzy match plus LLM disambiguation plus a confidence score; humans confirm low-confidence links).
   - **Typed edges** to documents and atoms (see the edge vocabulary below).
   - **Cross-expert linking:** "Expert A's atom contradicts Expert B's atom".
4. **Contradiction and staleness agent.** When a procedure is revised (DMS webhook), re-checks every atom linked to it and flags atoms that may now be stale.
5. **Review assistant.** Prepares each reviewer's queue: groups near-duplicates, proposes merges, pre-fills export-classification suggestions with reasons. The classification *decision* stays human.
6. **Twin evaluation agent.** Nightly regression over the held-out question set; posts the trend to the campaign dashboard.

**Edge vocabulary.** Typed, directional, and every edge carries `{created_by: agent|human, confidence, evidence_span, reviewed_by, label}`:

```
atom    —elaborates|clarifies|contradicts|supersedes|is_exception_to|is_prerequisite_of|causes|mitigates→  atom
atom    —cites|supplements|corrects|documents_gap_in|deviates_from→                                      document(+section/step anchor)
atom    —about→ equipment | system | vendor | task(JTA) | regulation
atom    —mentions→ person (role-based where possible)
document —revises|references|implements→ document
person  —knows_about(strength)→ topic/equipment   (derived; drives who_knows)
```

### 5.3 Multi-session memory, done properly

Replace "the last 8 atom titles" with a **campaign memory** made of four parts:

- **Running summary:** a rolling narrative per domain, rewritten after each session (expert-approved).
- **Open threads:** an explicit list of things the expert said they'd come back to ("remind me to tell you about the 2014 outage").
- **Expert profile:** preferred terms, people they reference, energy and fatigue preferences for session length. *No* emotion inference (section 3.4).
- **Retrieval at turn time:** before each interviewer turn, retrieve atoms relevant to the current topic from *all* sessions and from linked documents, so "last time you said…" is accurate and specific.

### 5.4 Generalized closed loop

Question sources become: `successor | gap(coverage) | followup(agent) | contradiction | reviewer | document_gap | stale_after_revision`. Each one carries priority, label, target expert(s) and a **resolution check**: an LLM judge confirms that the new atoms actually answer it (this fixes defect 6), with human override.

### 5.5 Coverage model v2

```
coverage(task) = f(reviewed atoms of required types for the task,
                   corroboration (≥2 sources: expert + document, or two experts),
                   twin eval pass-rate on questions tagged to the task)
priority(task) = KLRA position risk × task criticality (from JTA) × time-to-departure
```

The interviewer's `suggestNext` then picks the highest-`priority × (1 − coverage)` task. That's the same idea as today, grounded in the customer's own data.

---

## 6. Production architecture (target)

```
                    ┌──────────── Corporate IdP (Entra ID / Okta / ADFS) ─ OIDC/SAML, SCIM attributes ────────────┐
                    │                                                                                                │
 Expert / successor │   Web app (React)        Mobile walk-down app (offline, on-device STT)       MCP clients (assistants)
                    ▼            │                         │                                              │
              ┌─────────────────────────────── API gateway (mTLS, WAF, rate limits) ─────────────────────────────┐
              │  Tacit API  ──►  Policy decision point (OPA / Cedar): ABAC on every read, write, retrieval and MCP call │
              │      │                                                                                             │
              │      ├─ Interview service (realtime): STT ⇄ interviewer LLM ⇄ TTS, red-zone filter on every turn      │
              │      ├─ Job queue (Temporal / pg-boss): extraction, embedding, agents, evals; retries, no silent fallback │
              │      ├─ MCP server (Tacit tools)        MCP client hub (DMS, CMMS, CAP, HR, LMS connectors)            │
              │      └─ Audit log (append-only, hash-chained, shipped to SIEM)                                        │
              └──────────────────────────────────────────────────────────────────────────────────────────────────────┘
                     │                          │                                │
           PostgreSQL (+ row-level security,    Object storage (audio,           Model serving inside the boundary
           pgvector or OpenSearch, versioned    documents; SSE-KMS, per-site     vLLM / TGI on GPUs: LLM, embeddings,
           atoms/edges, label columns)          buckets, retention locks)        reranker, Whisper-class STT, TTS
```

### 6.1 Model strategy (the export boundary)

- **Default deployment: models self-hosted inside each jurisdiction's boundary.** Open-weight LLMs served by vLLM on 2–8 GPUs per site cover extraction, interviewing and twin answers. Run a **model-provenance review**: some nuclear operators' procurement rules restrict models from certain countries of origin, and the current default is Qwen (Alibaba). Raise this early.
- **Optional deployment: a government or sovereign cloud region with a commercial frontier model** (for example a frontier model through a cloud provider's government or sovereign region, where your export-control officer approves that provider's personnel and location controls). Frontier models give noticeably better extraction and follow-up quality. Make this a per-site policy switch, not a code fork.
- **Speech:** on-prem Whisper-class STT with a **custom vocabulary** from the equipment master and glossary. Nuclear jargon and tag numbers wreck generic STT. Stock on-prem TTS (Piper or a similar engine). **No cloud realtime voice** in the sensitive tiers.
- Keep the repo's `LLM` / `Embedder` / `Brain` abstraction idea. It's the right seam. Add a **`ModelRouter`** that picks the endpoint by the *data label* of the context it is about to send.

### 6.2 Data platform

- **PostgreSQL** with row-level security keyed to tenant, site and label, as defence in depth behind the policy decision point. Use proper relational tables, not JSON blobs.
- **Atoms are versioned** (`atom_versions`), with the original extraction kept forever. Edits, merges and splits are events.
- **Vectors:** pgvector up to a few million chunks. OpenSearch or Vespa if you need large-scale hybrid search with per-document security filtering. Add a **cross-encoder reranker**: it gives the biggest grounding improvement for the twin.
- **Documents:** ingest with layout-aware parsing (tables, step numbers, revision blocks), chunk by procedure step and section, and keep the step anchor so edges can point at "OP-3-FW-004 rev 6 §4.3".
- Audio is retained according to policy; the transcript is the system of record.

### 6.3 Reliability and operations

- Every LLM stage is a queued, idempotent job with retries. **Never write a silent heuristic fallback into the knowledge base.** If extraction fails, the turn is marked "pending extraction" and retried.
- Observability: OpenTelemetry traces per turn, cost and latency per stage, and an eval dashboard.
- Disaster recovery: per-site backups with legal hold. Test restores.
- Supply chain: SBOM, pinned dependencies, signed images, no runtime CDN fetches (the repo already serves MediaPipe locally, which is good; delete MediaPipe anyway).

---

## 7. Evaluation (what to measure before anyone trusts it)

| Eval | Method | Target to set in pilot |
|---|---|---|
| STT word error rate on nuclear vocabulary | 2–3 h of labelled pilot audio; WER overall and for **entity tokens** | Entity accuracy matters more than overall WER |
| Extraction precision and recall | Two humans label atoms on 20 transcripts; compare | Precision ≥ 0.9 (bad atoms poison trust); recall improves across sessions |
| Hallucinated atoms | Each atom's claims checked against its source quote (NLI judge plus spot audit) | ≈ 0 unsupported claims |
| Twin grounding | Held-out questions; claim-level citation precision; correct "I don't know" rate | Refusal beats invention; measure both |
| Classification-suggestion quality | Agreement with the derivative classifier | Advisory only; track drift |
| Red-zone detection | Seeded test phrases (fake codes and credentials) | Recall ≈ 1.0, tolerate false positives |
| Interviewer quality | Expert and successor ratings, follow-up depth, new-atoms-per-minute curve | Subjective; trend over sessions |

The roadmap item "evaluation harness for extraction quality across models" belongs in **Phase 0**, not later. It's also what you'll use to justify the model-hosting choice.

---

## 8. Roadmap

### Phase 0: pilot on the existing code (2–4 weeks; non-sensitive topics only)

Goal: learn interviewing patterns, session cadence, MCP value and reviewer workload with real retirees. **Keep it off controlled technology entirely**: pick topics like vendor relationships, outage logistics, training history and organizational know-how, and get written scope sign-off from export control.

1. Strip: voice cloning, the Boson cloud voice, the standalone browser-key mode, the 3D gesture graph, Nebius as default.
2. Add: an SSO proxy in front (oauth2-proxy plus IdP), close CORS, an audit-log middleware, a consent screen with a versioned record per session.
3. Swap to approved models (self-hosted, or an approved cloud region). On-prem Whisper-class STT; text and voice turns through the existing `/sessions/:id/turns` endpoint.
4. Add **document ingestion** (upload a PDF → chunks → shown alongside atoms) and a **document walk-through** session mode.
5. Add a **minimal MCP server** exposing `search_knowledge`, `ask_expert_twin`, `list_open_questions` and `add_question` over the existing `Engine`. This is a few hundred lines against the current API.
6. Add the **follow-up agent** (post-session; writes `followup` questions) and a basic **atom→document edge** proposer.
7. Build the eval harness and label the pilot data.
8. Run: 2–3 experts × 4–6 sessions each, with successors using the twin weekly. Collect metrics from section 7.

Exit criteria: measured extraction precision, reviewer hours per session hour, successor usefulness ratings, and a list of required connectors.

### Phase 1: spec and rewrite foundation (about 3 months)

Tenancy, sites and labels, the ABAC policy engine, Postgres schema with versioning, job queue, audit log, the model router, the document pipeline, MCP server with OAuth, review workflow (expert → SME → derivative classifier → owner), KLRA and JTA import.

### Phase 2: the programme platform (about 3 months)

Pre-research and relationship-tagging agents with DMS, CMMS and CAP connectors, entity resolution against the equipment master, campaign memory, coverage v2, a programme-level dashboard (risk-weighted coverage across all retirees), `who_knows`, group sessions with diarization, condition-report routing for procedure-versus-practice gaps.

### Phase 3: scale and reach (ongoing)

Mobile walk-down (offline), multilingual sites (the repo's "interview in any language, store atoms in English, keep the verbatim quote" pattern is good; add human translation review for controlled content), per-jurisdiction deployments, staleness monitoring on procedure revisions, post-retirement consultation loop (retired experts answer queued questions through a restricted portal under a consulting agreement).

---

## 9. What to keep, rework or drop from this repo

| Keep (port into the rewrite) | Rework | Drop |
|---|---|---|
| Interviewer persona and turn rules (`prompts.ts`) | Coverage formula → JTA- and KLRA-grounded | Voice cloning and Higgs Audio TTS |
| Parallel extract plus next-question loop | Session memory → campaign memory | Boson cloud realtime voice (for sensitive tiers) |
| Closed loop (unanswered → next interview) | Retrieval → indexed, ABAC-filtered, reranked, documents included | Hand-gesture and camera 3D Constellation (keep a 2D analyst graph) |
| Atom with verbatim `sourceQuote` and turn provenance | Graph → typed edges plus entity resolution | Standalone browser mode with user keys |
| Nine atom types; add `cue` (what tells you something's wrong), `rationale/design-basis`, `deviation` | Brain fallback → queued retries, fail loudly | Silent heuristic brain in production paths (keep it for tests and offline demos) |
| `Brain` / `LLM` / `Store` interfaces as a design pattern | Handover Markdown → controlled, labelled, versioned turnover package | Bundled demo samples in production builds |
| Multilingual capture with English atoms plus original quote | Phone agent → site-hosted SIP plus on-prem voice, if telephony is allowed at all | |
| Role templates as a seed (add nuclear: operations SRO, system engineer, I&C tech, RP, chemistry, reactor engineering, outage management, licensing) | | |

---

## 10. Spec outline (for after Phase 0)

1. Purpose, scope, non-goals ("not a QA-controlled system in v1"; "never captures SGI or classified information")
2. Stakeholders and roles: expert, successor, interviewer-facilitator, SME reviewer, derivative classifier, export-control officer, knowledge programme manager, auditor, admin
3. Regulatory requirements matrix (sections 3.2–3.5), each mapped to a control and a test
4. Domain model: tenant, site, programme, campaign, session, turn, atom (versioned), edge, document, entity, question, label, consent, review
5. Interview engine: session types, turn pipeline, red-zone filter, memory, interviewer policies
6. Knowledge pipeline: extraction, entity resolution, edge proposal, deduplication and merge, review states, classification
7. Retrieval and twin: ABAC-first retrieval, rerank, grounding verification, refusal policy, queueing
8. MCP: server tool contracts and schemas, auth, client connector contracts, agent job definitions and limits
9. Coverage and analytics: formulas, dashboards, saturation, evals
10. Security: identity, ABAC policy model, encryption, audit, insider-threat controls, export boundary per site
11. Deployment topologies: on-prem air-gapped, sovereign cloud, hybrid; model router rules
12. Non-functional: latency budgets (spoken turn under 2.5 s to first audio), availability, retention, DR, accessibility
13. Acceptance tests and pilot metrics

---

## 11. Open questions for you and the customer

1. Which jurisdictions and sites are in the first deployment, and which export regimes apply to each?
2. Is the company willing to run GPUs on-prem per site, or does it have an approved government or sovereign cloud?
3. Which DMS, CMMS and CAP systems are used, and do any already have MCP servers or usable APIs?
4. Does a KLRA register exist, and are JTAs available in machine-readable form?
5. Who acts as derivative classifier, and how many reviewer hours per expert can the programme fund?
6. Are works agreements needed (EU sites), and what is the lead time?
7. Will retired experts be available post-retirement under consulting agreements (this unlocks the asynchronous question loop)?
8. Is telephony or field (mobile) capture in scope, given site camera and phone restrictions?

---

### Sources consulted

- IAEA, *Knowledge Loss Risk Management in Nuclear Organizations*, NG-T-6.11 (2017): https://www-pub.iaea.org/MTCD/Publications/PDF/17-35661_PUB1734_web.pdf · 2006 predecessor: https://inis.iaea.org/records/gr9qs-knn97
- DOE 10 CFR Part 810 final rule (80 FR 9359, 2015): https://www.govinfo.gov/content/pkg/FR-2015-02-23/html/2015-03479.htm · https://www.energy.gov/nnsa/articles/10-cfr-part-810-final-rule-federal-register · summary: https://akingump.com/en/insights/alerts/doe-overhauls-export-controls-for-nuclear-technology
- EU AI Act workplace emotion-recognition prohibition: https://fpf.org/blog/red-lines-under-eu-ai-act-unpacking-the-prohibition-of-emotion-recognition-in-the-workplace-and-education-institutions/ · https://cdt.org/insights/eu-ai-act-brief-pt-4-ai-at-work/
- MCP authorization (spec 2025-11-25): https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization.md · changelog: https://modelcontextprotocol.io/specification/2025-11-25/changelog

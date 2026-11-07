# 02 · Glossary

Terms in **bold** are entities in the domain model (05). Code identifiers use the `snake_case` name shown.

| Term | Identifier | Meaning |
|---|---|---|
| **Tenant** | `tenant` | One customer organization. All data is tenant-scoped. |
| **Site** | `site` | A plant, office or business unit with its own jurisdiction, data residency and deployment boundary. |
| **Programme** | `programme` | A knowledge-retention initiative grouping many campaigns (for example "Retirement wave 2027–2030"). |
| **Campaign** | `campaign` | All the work to capture one departing expert's knowledge and stewardship. |
| **Expert** | role on `campaign_participant` | The departing person whose knowledge is captured. |
| **Successor** | role on `campaign_participant` | A person or group receiving the knowledge or stewardship. A campaign can have many. |
| **Session** | `session` | One interview event of a given type (07). |
| **Turn** | `turn` | One utterance by one speaker within a session. |
| **Knowledge item (KI)** | `knowledge_item` | The atomic unit of captured knowledge: typed, titled, cited to its source, labelled, versioned, reviewable. |
| **KI type** | `ki_type` | Category of knowledge (10 §2): procedure, rule, cue, gotcha, decision, rationale, deviation, contact, tool, glossary, risk, history. |
| **Source span** | `source_span` | A pointer to the exact evidence for a KI: a turn and character range, or a document chunk and anchor. |
| **Document** | `document` | A file or page ingested from a connector or upload, with versions and chunks. |
| **Anchor** | `anchor` | A stable location inside a document (section, step number, page, table cell). |
| **Entity** | `entity` | A canonical real-world thing: equipment, system, person, organization, vendor, document reference, regulation, location, task. |
| **Mention** | `mention` | An occurrence of an entity in a turn, KI or chunk. |
| **Edge** | `edge` | A typed, directed relationship between two nodes (KI, document, entity, question, artifact) with provenance and confidence. |
| **Question** | `question` | Something to ask an expert, with a source (successor, gap, follow-up, …) and a lifecycle. |
| **Label** | `label` | The security classification on a resource: export jurisdiction/class, sensitivity tier, handling caveats, origin site. |
| **Principal** | `principal` | An authenticated user or service with attributes used for access decisions. |
| **ABAC** | | Attribute-based access control: access is decided from principal attributes, resource labels and context. |
| **PDP** | | Policy decision point: the single component that evaluates access policies. |
| **Red zone** | | Information that must never be stored (06 §6). |
| **Stewardship** | | The duties, ownerships, custodianships and relationships a person holds on behalf of the organization. |
| **Stewardship item** | `stewardship_item` | One thing the expert holds that must be reassigned, closed or confirmed (08). |
| **Obligation** | category of stewardship item | An agreement or duty with continuing effect (for example an NDA where the expert is the company's designated custodian). |
| **Artifact** | `artifact` | A record in an external system the person authored, owns, approved, maintains or is assigned to (09). |
| **Owner of record** | | The person a system of record lists as owner or responsible party for an artifact. |
| **Reassignment** | `stewardship_transfer` | Formally moving ownership, custodianship or responsibility to a named recipient. It is complete only when the recipient accepts **and** the system of record shows the change. |
| **System of record (SoR)** | | The authoritative enterprise system for a type of artifact (DMS for documents, CMMS for work orders, …). |
| **Connector** | `connector` | An MCP server the platform calls (as an MCP client) to reach an enterprise system. |
| **Agent** | `agent_run` | A bounded background job that uses an LLM and connectors to produce proposals (13). |
| **Proposal** | | A machine-generated change (KI, edge, question, label suggestion, ownership match) that needs human acceptance before it takes effect. |
| **Twin** | | The retrieval-grounded question-answering capability over a campaign's or programme's knowledge (11). |
| **KLRA** | | Knowledge Loss Risk Assessment: attrition risk × position risk scoring (IAEA methodology). |
| **JTA** | | Job/Task Analysis: the list of tasks for a position, from the systematic approach to training. |
| **ERO** | | Emergency Response Organization: designated positions staffed during emergencies. |
| **CAP / CR** | | Corrective Action Program / Condition Report. |
| **CMMS** | | Computerized maintenance management system (for example Maximo, SAP PM). |
| **DMS** | | Document management system (for example Documentum, SharePoint, ProjectWise). |
| **CMDB** | | Configuration management database (for example ServiceNow), which lists application and service owners. |
| **SCWE / ECP** | | Safety-conscious work environment / Employee Concerns Program. |
| **Saturation** | | The point where additional sessions yield little new knowledge (14). |

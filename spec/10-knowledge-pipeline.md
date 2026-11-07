# 10 · Knowledge pipeline

Turns turns and documents into reviewed, labelled, linked knowledge items (KIs).

## 1. Stages

```
turn ──► extract_turn ──► KI proposals ──► entity resolution ──► dedup/merge proposals ──► edge proposals
                                                                                              │
document ──► ingest ──► parse/chunk ──► embed ──► entity mentions ──► (linked from KIs via edges)
                                                                                              ▼
                                       expert_review ──► sme_review ──► classification ──► released
```

## 2. KI types

| Type | Meaning | Type-specific `attributes` (zod) |
|---|---|---|
| `procedure` | Ordered steps to accomplish something | `steps: {n, text, caution?}[]`, `preconditions?: string[]`, `frequency?` |
| `rule` | Always / never / if-then constraint, rule of thumb | `condition?: string`, `action: string`, `strength: "always"\|"usually"\|"never"` |
| `cue` | Observable signal that something is happening or about to | `signal: string`, `meaning: string`, `threshold?: string`, `where_observed?: string` |
| `gotcha` | Trap, quirk, counter-intuitive behaviour | `symptom?`, `trap`, `avoidance` |
| `decision` | Judgement call: options, factors, what they'd choose | `situation`, `options: string[]`, `factors: string[]`, `preferred?` |
| `rationale` | Why something is designed or done a certain way; design-basis knowledge | `subject`, `reason`, `history?` |
| `deviation` | Practice differs from the written procedure | `document_ref`, `step_ref?`, `written`, `actual`, `reason`, `risk_view?` |
| `contact` | Who to go to for what | `who` (entity ref or name/role), `for_what`, `how?` |
| `tool` | System, software, report, spreadsheet, test equipment | `name`, `purpose`, `location?`, `owner?` |
| `glossary` | Term or acronym | `term`, `definition`, `aliases?` |
| `risk` | What can go wrong: trigger, impact, mitigation | `trigger`, `impact`, `mitigation?`, `likelihood_view?` |
| `history` | Event or story worth keeping (what happened, lesson) | `when?`, `what`, `lesson` |

- **KNW-001** Each KI is self-contained: readable without the transcript, with entities named explicitly ("the 2B feedwater pump", not "it").
- **KNW-002** KIs are written in the campaign's `knowledge_language`. Source quotes stay in the original language, with a translation when the languages differ.

## 3. Turn extraction

- **KNW-010** `extract_turn(turn_id)` builds context: the expert turn, the preceding AI or successor question, the previous 4 turns, the coverage areas, and the campaign glossary. It calls the model with a JSON schema returning 0–8 proposals `{type, title, body, attributes, confidence, coverage_area_id?, quote_spans:[{start,end}], entities:[{surface, kind}], stewardship_hint?}`.
- **KNW-011** Every proposal MUST have ≥ 1 quote span inside the turn (or the preceding turns, if the expert answered across turns). Proposals without valid spans are dropped.
- **KNW-012** **Faithfulness check:** a second model call (or an NLI model) verifies each proposal's claims are supported by the quote spans. Unsupported claims are removed, or the proposal is dropped. The result is stored in the version record.
- **KNW-013** Turns with `< 8` words, struck turns, K-01 turns and red-zone-only turns are skipped.
- **KNW-014** New KIs start at status `proposed` → shown live in the session UI → moved to `expert_review` when the session ends.
- **KNW-015** Label of a KI = `combine(turn label, labels of linked documents)`.

## 4. Entity resolution

- **KNW-020** Entities in proposals and chunks are resolved against the entity registry: exact alias match, then normalized fuzzy match (equipment tag normalization: strip separators, unit prefixes), then embedding similarity among same-kind entities, then an LLM disambiguation call when multiple candidates score within 0.1.
- **KNW-021** Resolution produces `mention` rows: `confirmed` when the score ≥ 0.9 and the match is unique, otherwise `proposed`. New entities are created as proposals.
- **KNW-022** The registry is seeded from connectors (equipment master from CMMS, people from HR, documents from DMS, vendors from contracts) and from the site glossary.

## 5. Dedup & merge

- **KNW-025** For each new KI, find candidates in the same campaign (and, as cross-campaign corroboration, the programme) with cosine ≥ 0.86 or FTS overlap. An LLM judge classifies the relationship as `duplicate | elaborates | contradicts | unrelated`.
- **KNW-026** `duplicate` → a merge proposal (reviewer accepts; versions are merged; both source spans kept). `elaborates`/`contradicts` → edge proposals (§6). Contradictions also create `contradiction` questions (08/07).

## 6. Relationship edges

Vocabulary (directional `src → dst`):

| Type | src → dst | Meaning |
|---|---|---|
| `elaborates` | KI → KI | adds detail |
| `contradicts` | KI → KI | conflicting claims (symmetric; store once) |
| `supersedes` | KI → KI | newer replaces older |
| `exception_to` | KI → KI | applies when the other doesn't |
| `prerequisite_of` | KI → KI | must know or do first |
| `causes` / `mitigates` | KI → KI | causal links between risks, cues and actions |
| `cites` | KI → document/chunk | KI refers to the document |
| `supplements` | KI → document/chunk | adds what the document lacks |
| `corrects` | KI → document/chunk | document is wrong here |
| `deviates_from` | KI → document/chunk | practice differs (deviation KIs) |
| `documents_gap_in` | KI → document | the document omits this |
| `about` | KI → entity | subject is equipment, system, task, vendor, … |
| `mentions` | KI → entity (person/org) | |
| `answers` | KI → question | resolution |
| `evidence_for` | artifact → stewardship_item | discovery evidence |
| `references` / `revises` / `implements` | document → document | from connector metadata or parsing |
| `knows_about` | entity(person) → entity | derived expertise (strength in `confidence`) |

- **KNW-030** The extractor may emit `stewardship_hint` (`{category, title, details}`) when the expert reveals a holding. This creates a proposed ad-hoc stewardship item (08 STW-011).
- **KNW-031** Edge proposals are created by the relationship agent (13 AGT-040) and by the pipeline. Agent-created edges with confidence ≥ 0.85 to *documents* may be auto-accepted if the tenant enables it. Every other edge needs review.
- **KNW-032** Edge label = combine(src label, dst label).

## 7. Review workflow

| Stage | Who | Can do | Exit |
|---|---|---|---|
| `expert_review` | Expert (attributed) | approve, edit, reject, redact, split | approved → `sme_review` (or `classification` if the site skips SME for the type) |
| `sme_review` | Assigned SME reviewer(s) | approve, edit (creates version), reject, request change (back to expert) | approved → `classification` |
| `classification` | Classifier | set label (export classes, sensitivity, caveats) | classified → `released` |

- **KNW-040** Batch review: reviewers can approve groups of KIs; each approval is still recorded per KI (REG-014).
- **KNW-041** Edits after release create a new version and send the KI back to the earliest stage the site's policy requires for that edit (default: SME review for content changes, classification for any label change).
- **KNW-042** Rejected KIs keep their history, are excluded from retrieval and stay visible to reviewers.

## 8. Document ingestion

- **KNW-050** Sources: connector `get_document_content` (09 OWN-013), or user upload (PDF, DOCX, XLSX, PPTX, TXT, MD, HTML).
- **KNW-051** Parsing: text with structure (headings, numbered steps, tables, page numbers) through Tika plus a layout parser. Procedures: detect step numbering (`4.3`, `4.3.1`, `a.`) to create `step` anchors.
- **KNW-052** Chunking: by section or step, 200–600 tokens, never crossing a top-level section. Each chunk keeps its anchor.
- **KNW-053** Each version is stored immutably (`object_key`, `sha256`). Re-ingesting the same hash is a no-op. A new revision creates a new `document_version` and triggers AGT-060 (staleness).
- **KNW-054** Label: from the connector's classification hint or the site default (pessimistic).

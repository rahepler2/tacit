# 16 · Frontend

## 1. Principles

- **UI-001** Accessibility: WCAG 2.1 AA. Full keyboard operation, visible focus, screen-reader labels, captions for all spoken AI output (the transcript), no information conveyed by colour alone.
- **UI-002** A **label banner** at the top of every view that shows labelled content (combined label of the visible content), and on every exported or printed page.
- **UI-003** Calm, professional, data-dense design suited to engineering users. No decorative 3D, no camera features. Light and dark themes.
- **UI-004** Internationalization: all strings externalized (ICU messages). UI language is independent of interview language.
- **UI-005** Works on desktop (≥ 1280 px) as the primary target and on tablets for sessions and reviews. Field walk-down PWA comes later.

## 2. Screens

| ID | Screen | Users | Key elements |
|---|---|---|---|
| UI-010 | **Programme dashboard** | programme manager | COV-030 metrics, campaigns table (risk, departure, coverage, stewardship closure), proposals, orphaned artifacts, red flags |
| UI-011 | **Imports** | programme manager | risk register / JTA upload, dry-run diff, apply |
| UI-020 | **Campaign overview** | facilitator, expert, successors | timeline of sessions, coverage areas with scores, stewardship progress by category, open questions, dossier link, next session agenda |
| UI-021 | **Campaign setup wizard** | facilitator | expert, position, departure, participants, languages, coverage plan review, consent dispatch |
| UI-022 | **Dossier** | facilitator, expert | AGT-010 output; artifact groups; walk-through candidates; "add to agenda" |
| UI-030 | **Session room** | expert, facilitator | big, clear current AI question with caption; live transcript with speaker labels; mic state and level; pause / strike-that / switch to text; side panel tabs: *Agenda* (progress), *Captured* (live KI proposals), *Stewardship* (items touched); facilitator controls (take over, skip item, mark N/A, end) |
| UI-031 | **Document walkthrough room** | expert, facilitator | session room plus a document viewer with anchors; expert/facilitator can highlight a passage → "comment" creates an anchored turn context; KIs auto-link to the anchor |
| UI-032 | **Session summary** | expert | generated summary, approve/edit; KIs from this session for quick review |
| UI-035 | **Review queue** | expert, SME, classifier | filterable list; side-by-side KI and source quotes and audio snippet (if retained); diff of versions; batch approve; label editor (classifier) with assistant suggestion and rationale |
| UI-036 | **Knowledge browser** | all (per ABAC) | search, filters (type, area, entity, expert), KI detail with versions, sources, edges, related |
| UI-037 | **Graph explorer (2D)** | all (per ABAC) | node-link view around a selected node (≤ 2 hops), filters by edge type, accept/reject edge proposals (reviewers) |
| UI-038 | **Ask (twin)** | successors, staff | chat with citations; confidence badge; contradictions callout; "send to expert" when low confidence; notice banner |
| UI-040 | **Stewardship form** | expert | catalog items grouped by category; each item uses its `response_schema` to render form fields; "discuss in session instead"; progress |
| UI-041 | **Stewardship board** | facilitator, programme manager | all items: columns by status, filters by category/function/due; bulk assign recipients |
| UI-042 | **Ownership discovery** | expert, facilitator | per-system status; artifact groups (system × type × role) with counts; uncertain matches to confirm (with context); create transfers per group or split |
| UI-045 | **Transfer acceptance** | recipient | what's being handed over, expert's notes, linked KIs and documents; accept / decline with reason; "handover briefing completed" checkbox |
| UI-046 | **My stewardship register** | any recipient | everything transferred to me, status of SoR updates, links |
| UI-047 | **Function queue** | function owners | routed items with due dates; actions: request SoR change (if write-back is on), verify manually with evidence, close |
| UI-048 | **Exit attestation** | expert | generated statement; corrections; acknowledge |
| UI-050 | **Admin** | tenant admin | sites, connectors (health, capabilities), catalog editor (versioned), consent templates, model endpoints, doc-type mappings |
| UI-051 | **Ops** | admin | jobs, agent runs, failures, retries |
| UI-052 | **Audit** | auditor | search, chain verification status |

## 3. Session room behaviour

- **UI-060** The mic only starts after an explicit user action and confirmed consent. The mic state is always visible.
- **UI-061** Barge-in: if the expert speaks while TTS plays, playback stops immediately and the client sends `control.barge_in`.
- **UI-062** Network loss: show a reconnecting state, buffer up to 30 s of audio locally, then resume (API-010). Never lose a finalized turn.
- **UI-063** Live KI proposals appear within 5 s of a turn's end (p50). The expert can thumbs-down a proposal live, which marks it for rejection review.
- **UI-064** Stewardship moves are shown with a distinct visual tag ("Stewardship · B-02") so the expert understands why the question is being asked.
- **UI-065** K-01 shows the ECP contact panel (from site settings) when the expert answers "yes" or "prefer not to say".

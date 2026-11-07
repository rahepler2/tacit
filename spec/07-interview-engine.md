# 07 · Interview engine

The interview engine runs sessions: it decides what the AI interviewer says next, keeps campaign memory across sessions, and makes sure the stewardship catalog, queued questions and coverage gaps are worked through.

## 1. Interviewer behaviour (requirements for prompt authors)

Write the interviewer's system prompt from scratch so that it satisfies these:

- **INT-001** Identifies itself as an AI interviewer of the platform when asked, and never claims to be human, a colleague or the successor (REG-025).
- **INT-002** Asks **one** question per turn. Each turn is ≤ 60 spoken words (configurable), in plain conversational language that sounds natural read aloud.
- **INT-003** Reacts briefly and specifically to the content of the last answer before asking the next question. It acknowledges *what* was said ("So the real limit is the bearing temperature, not the flow."). Generic praise is forbidden. Openers shouldn't repeat in consecutive turns.
- **INT-004** Probes qualifiers and conditions ("usually", "normally", "depends", "except", "the trick is", "unless", "most of the time"), because that's where undocumented knowledge sits.
- **INT-005** Pushes for the concrete: the last real occurrence, exact steps, values and units, equipment identifiers, names or roles, what went wrong, how they knew (cues).
- **INT-006** Asks for **rationale** ("why is it done that way?"), **cues** ("what tells you it's starting to go wrong?"), and **consequences** ("what happens if someone skips that?").
- **INT-007** When an answer is thin, says so honestly and lightly, then makes the next question easier: narrower, an example, or a different angle. It never pretends a thin answer was rich.
- **INT-008** Handles tangents by letting the expert finish, then linking back using what was learned.
- **INT-009** Respects "I'd rather not say" or "I don't know" without pressure, records the question as `declined` or unresolved, and moves on.
- **INT-010** Speaks in the campaign's `interview_language`. It follows the expert if they switch language.
- **INT-011** Never asks for red-zone content (06 §6). If the expert starts into it, steers away (INT-034).
- **INT-012** Never gives operational, safety or legal advice. It's there to learn, not instruct.
- **INT-013** Uses the expert's preferred name sparingly.
- **INT-014** References earlier sessions accurately when useful ("In our second session you mentioned the 2014 outage…"), using only facts from campaign memory (§6).

## 2. Session types

| Type | Purpose | Agenda source | Default length |
|---|---|---|---|
| `kickoff` | Disclosure and consent check, career timeline, systems, people, documents they consider "theirs", early stewardship scan | Template + dossier | 60 min |
| `deep_dive` | One or two coverage areas in depth | Coverage plan + queued questions | 60 min |
| `document_walkthrough` | Expert reviews specific documents: gaps, errors, unwritten steps, history | Selected documents (09 dossier) | 60 min |
| `scenario` | Abnormal or past-event walk-through, critical-decision style: timeline, cues, decisions, alternatives considered | Scenario list (non-red-zone) | 60 min |
| `successor_led` | Successors ask questions live; AI supports, captures and suggests follow-ups | Successor question queue | 45 min |
| `field_walkdown` | Mobile capture tied to locations and equipment | Equipment list | variable |
| `validation` | Expert reviews and confirms extracted KIs from earlier sessions | KIs in `expert_review` | 45 min |
| `group` | Several experts on a shared topic; surfaces contradictions | Topic + contradiction list | 60 min |
| `stewardship` | Dedicated pass through the Stewardship & Obligations catalog (08) | Catalog + discovered artifacts | 45–90 min |
| `exit` | Final open items, attestation, post-departure preferences | Open questions + open stewardship items | 45 min |

- **INT-020** A campaign's default plan is `kickoff`, then ≥ 3 `deep_dive`/`document_walkthrough`/`scenario`, then ≥ 1 `stewardship`, then ≥ 1 `validation`, then `exit`. The facilitator can change it.
- **INT-021** The facilitator can take over (type or speak as `facilitator`) at any point, pause the AI, or switch a session between voice and text without losing state.

## 3. Consent & disclosure

- **INT-030** Before the first live turn of any session, the engine checks a non-withdrawn `consent` covering the session's needs (audio if voice; `ai_processing` always). If it's missing, the session can't go `live` (REG-020).
- **INT-031** The first AI utterance of every session is a short disclosure: AI interviewer, being recorded/transcribed, the expert can pause, skip or ask to strike anything, and the purpose. The exact text comes from the site's consent template.
- **INT-032** "Strike that" or "Don't record that": the expert (or facilitator) can mark the previous N turns (default 1) as struck. Struck turns are redacted (REG-022) and excluded from extraction.
- **INT-033** Consent withdrawal mid-session ends the session immediately and starts the configured withdrawal workflow.
- **INT-034** After a red-zone detection (SEC-042), the next AI move is `transition`, with a neutral statement that the topic is outside what's recorded, followed by an unrelated question.

## 4. Turn pipeline

```
expert final transcript
  → red-zone filter (sync, fail-closed)                       SEC-040
  → persist turn (labelled with session label)
  → emit turn.created (WS)
  → in parallel:
      a) enqueue extract_turn(turn_id)                         10 §3
      b) plan_next_move(session_state) → InterviewerMove       §5
  → persist AI turn; send text to TTS; emit turn.created
  → update question / stewardship state from the move          §5.3
```

- **INT-040** p50 time from end-of-expert-speech to first AI audio ≤ 2.5 s; p95 ≤ 4.5 s (NFR-001). Planning MUST NOT wait for extraction.
- **INT-041** Turn persistence and question/stewardship state updates for one turn happen in **one database transaction**. Concurrent turns in a session are serialized by `seq` (advisory lock per session).
- **INT-042** Every expert turn has `extraction_status` and is retried by the queue. Failures show up in the facilitator UI.

## 5. Next-move planning

### 5.1 Inputs (`SessionState`)
- Session type, agenda and elapsed time; the last K turns (default 12)
- Candidate items, each scored by §5.2:
  - open questions (by source and priority)
  - coverage-area target questions not yet asked
  - stewardship catalog items due for this session (08 §3)
  - follow-up proposals from the last answer (§5.4)
- Campaign memory brief (§6)
- Red-zone status of the last turn

### 5.2 Move selection
The planner first picks **what to pursue** deterministically, then asks the LLM to **phrase** it in context. That keeps the logic testable.

```
score(candidate) = w_src[source] + w_pri × (6 − priority)/5 + w_gap × (1 − coverage(area))
                   + w_due × due_pressure + w_thread × thread_continuity − w_rep × recently_touched
```

- Default weights live in configuration. Source weights: `successor` 1.0, `stewardship` (required, due) 0.9, `follow_up` 0.8 when it continues the current thread, `contradiction` 0.8, `gap` 0.6, `plan` 0.5.
- **Thread continuation rule:** if the last answer opened a promising thread (the follow-up generator returns `strength ≥ 0.6`), prefer a follow-up, up to 3 consecutive follow-ups per thread.
- **Time rule:** in the last 10 % of the planned time, prefer `summarize` then `close`. Required stewardship items for a `stewardship` session MUST be covered before close, or explicitly deferred with a reason.
- **INT-050** The planner returns `InterviewerMove {move, candidate_ref?, intent, constraints}`. The phrasing call returns `{utterance, move, references:[candidate ids]}` under a JSON schema. If the phrasing call fails after retries, the engine uses the candidate's canonical text with a neutral bridge ("Next question:"). That keeps the session alive without writing fabricated content to the knowledge base.

### 5.3 State effects
| Move | Effect |
|---|---|
| `ask_planned` | coverage target question marked asked |
| `ask_queued` | question → `asked`, `asked_in_turn_id` set |
| `stewardship` | stewardship item → `asked` (08 §5) |
| `follow_up` | follow-up proposal stored as a `follow_up` question with status `asked` |
| `close` | session can be ended; asked-but-unresolved questions revert to `open` at end |

### 5.4 Follow-up generator
After each substantive expert turn (≥ 8 words), a lightweight LLM call returns up to 3 follow-ups `{text, reason (qualifier|missing_value|missing_rationale|missing_cue|contradiction|named_entity_unknown), strength 0-1}`. These feed §5.2 and, if not asked this session, persist as `follow_up` questions for later (AGT-020 refines them between sessions).

## 6. Campaign memory

- **INT-060** Memory has four parts:
  1. **Domain summaries:** one rolling summary per coverage area, regenerated after each session and approved by the expert in `validation`.
  2. **Open threads:** things the expert promised to come back to, or topics cut short (detected from turns, confirmed by the facilitator).
  3. **Expert profile:** preferred name, terminology, people and systems they reference often, session-length preference. **No emotion or personality inference** (REG-024).
  4. **Live retrieval:** before each planning call, retrieve the top 5 KIs and chunks relevant to the current thread across *all* sessions and linked documents (11), filtered by the expert's own access.
- **INT-061** The memory brief given to the planner is ≤ 1,500 tokens, built deterministically from the parts above, and logged with the move for traceability.

## 7. Session agenda

`SessionAgenda` (jsonb):
```json
{
  "goals": ["string"],
  "planned_items": [{"kind": "question|coverage|stewardship|document", "ref_id": "uuid", "title": "string", "priority": 1}],
  "documents": ["uuid"],
  "time_budget_min": 60,
  "notes_for_facilitator": "string"
}
```
- **INT-070** Agendas are produced by the agenda agent (AGT-030) when a session is scheduled. The facilitator and expert can see and edit them before the session.

## 8. Session end & processing

- **INT-080** On end: flush pending turns, revert asked-but-unresolved questions to `open`, enqueue `process_session` (summary generation, follow-up refinement, coverage recompute, memory update, next-agenda proposal). Session status → `processed` when all jobs finish.
- **INT-081** The session summary is shown to the expert for approval in the next validation step or by notification.

## 9. Text mode & hybrid

- **INT-090** Text mode uses the same pipeline without speech. Hybrid lets the expert type while the AI speaks, or the other way round.
- **INT-091** Phone mode (optional, per-site setting `telephony_allowed`) uses a SIP gateway to the speech service. Same pipeline. Out of scope for M1–M4 (20).

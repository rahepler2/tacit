# 19 · Testing, evaluation & synthetic data

## 1. Test layers

| Layer | Tooling | Scope |
|---|---|---|
| Unit | Vitest (TS), pytest (speech) | Pure logic: scoring, label combine/dominate, state machines, parsers |
| Contract | zod/JSON-schema snapshot tests; OpenAPI diff check | `packages/domain` schemas, API, MCP tool schemas |
| Integration | Vitest + Testcontainers (Postgres, MinIO) | Repositories, RLS, PDP SQL filters, jobs |
| Policy | Cedar test vectors (SEC-020) + property tests | Authorization correctness |
| Connector conformance | `connectors-sdk` suite | Every connector |
| E2E | Playwright against `infra/compose` | User journeys (§3) |
| Evaluation | `packages/evals` | Model-quality metrics (§4) |
| Load | k6 | NFR-001..005 |

- **TST-001** Every requirement ID has at least one automated test or an explicit "manual verification" entry in `docs/verification-matrix.md` (generated table: requirement → tests).
- **TST-002** CI runs on every PR: lint, typecheck, unit, contract, integration, policy, conformance (fixtures), and E2E smoke. Nightly: full E2E, evals with the mock model and (where available) the real model endpoint, load-test smoke.
- **TST-003** **Mock model server** (`infra/mock-models/`): an OpenAI-compatible server returning deterministic, schema-valid responses keyed by prompt template ID and input hash, from recorded fixtures. That makes the pipeline testable without GPUs. Recording mode captures real-model responses to refresh fixtures.

## 2. Critical property and invariant tests

- **TST-010** PDP equivalence: for random principals × resources, `authorize()` permit ⇔ the row is returned by `filterSql()` queries (SEC-021).
- **TST-011** No-leak retrieval: for every ABAC persona, search, twin, MCP tools, exports and WebSocket events never include resources that persona isn't permitted to read (seeded dataset with labelled "canary" KIs whose text contains unique tokens; assert the tokens never appear in any response to unauthorized personas).
- **TST-012** Red-zone: seeded phrases (synthetic passwords, codes, alarm-detail sentences) never reach the database, logs, traces or model calls beyond the filter. Assert through DB scans and log capture.
- **TST-013** Model router: content labelled above an endpoint's `max_label` is never sent to that endpoint (mock endpoints record received payloads).
- **TST-014** Audit chain integrity after concurrent writes; tamper detection.
- **TST-015** Turn atomicity: concurrent turn submissions in one session yield gap-free `seq` and consistent question/stewardship state.
- **TST-016** Closure gate: campaigns can't close with open required items (STW-040).
- **TST-017** K-01 handling (STW-020).

## 3. E2E journeys (Playwright, on fixtures)

1. Admin configures site, connectors (fixtures), catalog; programme manager imports risk register and JTA.
2. Facilitator creates a campaign for fixture expert "R. Alvarez"; consent; activation; discovery finds artifacts in DMS, wiki, CMMS, CAP, CMDB, contracts, LMS; uncertain initials matches confirmed.
3. Text-mode kickoff session; KIs proposed live; a stewardship hint is detected.
4. Stewardship session covers all categories; B-02 group transfer to successor "M. Chen"; M. Chen accepts; function owner requests SoR change; fixture applies owner change; re-check verifies.
5. Review pipeline to release; classifier labels.
6. Successor asks the twin: a high-confidence answer with citations; a low-confidence question sent to the expert; the next session's agenda includes it; the answer resolves it.
7. ABAC persona (foreign-national contractor, no authorization) can't see controlled KIs anywhere (UI, search, twin, MCP).
8. Campaign closing is blocked, then items closed, attestation signed, campaign closed; stewardship report exported with banner and watermark.
9. MCP client (test harness using the MCP SDK) calls `search_knowledge`, `ask_expert_twin`, `who_knows`, `find_owner` with persona tokens.

## 4. Evaluation suite (`packages/evals`)

| Suite | Data | Metrics | Gate (initial, tune in pilot) |
|---|---|---|---|
| `extraction` | Synthetic transcripts with gold KIs (§5) | precision, recall, type accuracy, span validity, unsupported-claim rate | precision ≥ 0.85, unsupported ≤ 2 % |
| `interviewer` | Scripted expert simulations (LLM-simulated expert from persona cards) | follow-up rate on hedges, one-question compliance, word-count compliance, banned-phrase rate, stewardship coverage per session | compliance ≥ 95 % |
| `twin` | Gold Q/A with expected citations; unanswerable set | answer correctness (judge), citation precision/recall, correct-abstention rate | abstention on unanswerable ≥ 90 % |
| `redzone` | Seeded positives and hard negatives | recall per class, false-positive rate | SEC-044 |
| `entity_resolution` | Fixture equipment master + noisy mentions | accuracy, ambiguity handling | ≥ 0.9 |
| `ownership_match` | Alias variants vs fixture artifacts | precision of auto-confirmed matches | ≥ 0.99 |
| `stt` | Synthetic TTS-generated plus recorded audio with nuclear vocabulary | WER, entity accuracy | SPH-009 |

- **TST-030** Evals produce `eval_run` records and a markdown report. Regressions beyond configured tolerances fail the nightly build.

## 5. Synthetic plant dataset (`infra/seed/`)

All data is **fictional** and must not resemble any real plant or person.

- Plant: "Example Station" Units 1 and 2 (generic pressurized-water-reactor-style systems with fictional tags: `1-FW-P-2B`, `2-CC-HX-1A`, etc.). Systems: feedwater, component cooling, service water, emergency diesel generators, plant computer, radiation monitoring (no security systems).
- People: ≈ 40 fictional principals with varied citizenships, employers (including contractors), sites (US site "EXS", EU site "EXE"), roles, and alias variants (initials, name changes).
- Documents: ≈ 300 fictional documents across doc types (procedures with numbered steps, calculations, specs, wiki pages) with revision histories; owners/authors/approvers varied; some owned by soon-to-depart experts; some proprietary vendor docs with caveats.
- Work items: CMMS work orders, CAP condition reports and actions, commitments, contracts/NDAs with fictional vendors, CMDB apps and service accounts, LMS courses and evaluator assignments, ERO roster, assets/keys/check-source custody.
- Transcripts: 30 synthetic interview transcripts (multiple session types; includes hedges, deviations, stewardship mentions, red-zone seeds) with gold KIs, gold entities and gold stewardship items.
- ABAC personas: at least: US-citizen engineer (authorized), EU-citizen engineer at EU site, foreign-national contractor without authorization, classifier, ECP officer, auditor, tenant admin (no content), successor, expert.
- Canary KIs (TST-011) with unique tokens per label combination.

# 15 · HTTP API

## 1. Conventions

- **API-001** Base path `/api/v1`. JSON request and response bodies. OpenAPI 3.1 is generated from zod schemas and published at `/api/v1/openapi.json`.
- **API-002** Errors follow RFC 9457 problem details: `{type, title, status, detail, code, errors?}`. `code` is a stable machine string (for example `CLOSURE_BLOCKED`, `CONSENT_REQUIRED`, `LABEL_DENIED`, `REDZONE_FILTER_UNAVAILABLE`, `MODEL_NOT_PERMITTED_FOR_LABEL`, `VALIDATION_FAILED`, `NOT_FOUND`, `CONFLICT`).
- **API-003** Authorization failures on content return `404 NOT_FOUND` (don't reveal that the resource exists) unless policy allows a "request access" affordance, in which case `403 LABEL_DENIED`.
- **API-004** Lists use cursor pagination: `?cursor=&limit=` (max 200) → `{items, next_cursor}`.
- **API-005** Mutations accept an `Idempotency-Key` header. Retried keys within 24 h return the original result.
- **API-006** Optimistic concurrency on editable resources via `version` in the body (KIs: `current_version`), or `If-Match` ETags. A mismatch returns `409 CONFLICT`.
- **API-007** Every response that contains labelled content includes header `X-Label-Banner` and a `label` field on each labelled object.

## 2. Endpoints (minimum set)

### Identity & admin
| Method | Path | Purpose |
|---|---|---|
| GET | `/auth/login`, `/auth/callback`, POST `/auth/logout` | OIDC flow |
| GET | `/me` | principal, roles, attributes visible to self |
| GET/PUT | `/admin/sites/{id}` | site settings |
| GET/POST/PUT | `/admin/connectors[/{id}]`, POST `/admin/connectors/{id}/test` | connectors |
| GET/POST/PUT | `/admin/stewardship-catalog[/{id}]` | catalog (versioned) |
| GET/POST | `/admin/consent-templates` | consent templates |
| GET/PUT | `/admin/model-endpoints` | router registry |
| GET/PUT | `/admin/doc-type-mappings` | OWN-021 |
| POST | `/scim/v2/...` | SCIM 2.0 Users/Groups |

### Programme & analytics
| Method | Path | Purpose |
|---|---|---|
| GET/POST | `/programmes`, GET/PATCH `/programmes/{id}` | |
| POST | `/programmes/{id}/imports/risk-register`, `/imports/jta` | dry-run with `?dry_run=true` |
| GET | `/programmes/{id}/dashboard` | COV-030 metrics |
| GET | `/programmes/{id}/campaign-proposals` | COV-010 |
| GET | `/programmes/{id}/orphaned-artifacts` | OWN-050 |

### Campaigns
| Method | Path | Purpose |
|---|---|---|
| GET/POST | `/campaigns`, GET/PATCH `/campaigns/{id}` | |
| POST | `/campaigns/{id}/activate`, `/start-closing`, `/close` (409 `CLOSURE_BLOCKED`), `/suspend`, `/resume` | state machine |
| GET/POST/DELETE | `/campaigns/{id}/participants` | |
| GET/POST | `/campaigns/{id}/consents`, POST `/consents/{id}/withdraw` | |
| GET/PUT | `/campaigns/{id}/coverage-areas` | plan |
| GET | `/campaigns/{id}/coverage` | scores |
| GET | `/campaigns/{id}/dossier` | AGT-010 output |
| GET | `/campaigns/{id}/stewardship-report` (`?format=json|pdf|docx`) | STW-042 |
| POST | `/campaigns/{id}/attestation` | STW-041 |

### Sessions & turns
| Method | Path | Purpose |
|---|---|---|
| GET/POST | `/campaigns/{id}/sessions`, GET/PATCH `/sessions/{id}` | schedule, agenda |
| POST | `/sessions/{id}/start` (409 `CONSENT_REQUIRED`), `/pause`, `/resume`, `/end` | |
| GET | `/sessions/{id}/turns` | |
| POST | `/sessions/{id}/turns` | text-mode turn `{text, speaker_role}` → `{turn, next: InterviewerTurn?}` |
| POST | `/sessions/{id}/strike` | `{last_n}` (INT-032) |
| POST | `/sessions/{id}/takeover` | facilitator control |
| WS | `/sessions/{id}/live` | §3 |

### Knowledge
| Method | Path | Purpose |
|---|---|---|
| GET | `/knowledge-items?campaign_id&status&type&q&entity_id&cursor` | |
| GET/PATCH | `/knowledge-items/{id}` | edit creates a version |
| GET | `/knowledge-items/{id}/versions` | |
| POST | `/knowledge-items/{id}/reviews` | `{stage, decision, comment}` |
| POST | `/knowledge-items/{id}/label` | classifier |
| POST | `/knowledge-items/{id}/redact` | REG-022 |
| POST | `/knowledge-items/merge`, `/knowledge-items/{id}/split` | |
| GET/POST/PATCH | `/edges`, `/edges/{id}` | accept/reject |
| GET | `/entities?q&kind`, GET `/entities/{id}` (with neighbours) | |
| GET/POST | `/documents` (upload multipart), GET `/documents/{id}`, GET `/documents/{id}/chunks` | |

### Questions & twin
| Method | Path | Purpose |
|---|---|---|
| GET/POST | `/questions`, PATCH `/questions/{id}` | |
| POST | `/twin/ask` | `{question, scope?, conversation_id?}` → twin answer (TWN-002 plus confidence and citations) |
| POST | `/twin/send-to-expert` | TWN-005 |
| GET | `/who-knows?topic=` | RET-020 |
| GET | `/search?q&kinds&filters` | RET |

### Stewardship & ownership
| Method | Path | Purpose |
|---|---|---|
| GET | `/campaigns/{id}/stewardship-items?category&status` | |
| POST | `/campaigns/{id}/stewardship-items` | ad hoc |
| GET/PATCH | `/stewardship-items/{id}` | answer `details`, set N/A, defer |
| POST | `/stewardship-items/{id}/transfers` | propose `{recipient, transfer_kind, artifact_ids?}` |
| POST | `/transfers/{id}/accept`, `/decline`, `/request-sor-change`, `/verify-manual` | |
| GET | `/me/stewardship-register` | STW-033 |
| GET | `/me/function-queue` | routed items for my functions |
| GET | `/campaigns/{id}/artifacts?group_by=type` | 09 §5 |
| POST | `/campaigns/{id}/discovery/run` | |
| GET | `/campaigns/{id}/identity-aliases`, POST `/identity-aliases/{id}/confirm|reject` | |
| POST | `/artifacts/{id}/confirm-match`, `/reject-match` | |

### Ops
`GET /ops/jobs`, `GET /ops/agent-runs`, `POST /ops/agent-runs/{id}/retry`, `GET /ops/connectors/health`, `GET /healthz`, `GET /readyz`, `GET /audit` (auditor), `GET /audit/verify`.

## 3. Live session WebSocket protocol

Messages are JSON, except binary audio frames.

Client → server:
| type | payload |
|---|---|
| `audio.start` | `{sample_rate: 16000, encoding: "pcm16"}` |
| *(binary)* | PCM16 mono frames (20–40 ms) |
| `audio.stop` | `{}` |
| `text.turn` | `{text}` (hybrid mode) |
| `control.pause` / `control.resume` / `control.end` | `{}` |
| `control.strike` | `{last_n}` |
| `control.barge_in` | `{}` (user started speaking during TTS; server stops TTS) |

Server → client:
| type | payload |
|---|---|
| `session.state` | `{status, elapsed_s, agenda_progress}` |
| `transcript.partial` | `{text, speaker_role}` |
| `turn.created` | `Turn` |
| `tts.start` / *(binary audio)* / `tts.end` | `{turn_id, sample_rate, encoding}` |
| `ki.proposed` / `ki.updated` | `KnowledgeItem` summary |
| `stewardship.updated` | item summary |
| `question.updated` | question summary |
| `coverage.updated` | area scores |
| `redzone.detected` | `{turn_id, class}` (no content) |
| `error` | problem details |

- **API-010** The server sends `session.state` on connect and after every state change. Clients reconnect with `?resume_from_seq=N` and receive missed events.

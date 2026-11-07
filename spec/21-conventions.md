# 21 · Engineering conventions

## 1. Code

- TypeScript `strict: true`, `noUncheckedIndexedAccess: true`. No `any` except at validated boundaries (followed immediately by zod parsing).
- ESM only. Node 22 LTS. pnpm. Biome or ESLint + Prettier (choose one in M0-1; record it in `docs/decisions/`).
- **Validation at every boundary:** HTTP, WebSocket, MCP, job payloads, model outputs and config are all parsed with zod schemas from `packages/domain`.
- **Errors:** typed error classes mapped to API `code`s (15 API-002). Never swallow errors. Model and connector failures are surfaced (ARC-005).
- **Database:** all access through `packages/db` repositories; every transaction sets `app.tenant_id` / `app.principal_id`; no string-concatenated SQL.
- **Authorization:** every service method that reads or writes content takes a `RequestContext {principal, tenant, site, purpose}` and calls the PDP. A lint rule or test flags repository calls made outside service methods that hold a context.
- **Prompts:** in `packages/models/prompts/<id>/<version>.md` with front-matter `{id, version, purpose, output_schema, max_label}`. Prompts are written fresh (CR-5). Each prompt has eval coverage.
- **No content in logs** (NFR-032). Use IDs.
- **Time:** store UTC `timestamptz`; display in the user's time zone.
- **i18n:** no hard-coded user-facing strings in `apps/web`.
- Python (speech): 3.12, `uv`, `ruff`, `mypy --strict`, pytest.

## 2. Repository & PR workflow (for agents)

- Branches: `ws/<workstream>/<short-task>` (for example `ws/stewardship/closure-gate`).
- Commits: Conventional Commits (`feat(stewardship): closure gate STW-040`).
- PR description template:
  ```
  ## What
  ## Requirement IDs
  STW-040, API-002
  ## Tests
  ## Contract changes (none | SCR-NNNN)
  ## Clean-room statement
  I worked only from spec/ and public docs (CR-1..CR-6).
  ```
- One reviewing agent from a different workstream, plus the integrator for contract-touching PRs.
- New dependencies need: purpose, licence (allow-list: MIT, Apache-2.0, BSD-2/3, ISC, MPL-2.0; anything else goes through an SCR), maintenance status, size, and an air-gap compatibility note.

## 3. Documentation

- `docs/decisions/NNNN-title.md`: ADR-style records of implementation decisions.
- `docs/runbooks/`: operations (backup/restore, key rotation, connector onboarding, incident response for red-zone alerts).
- `docs/verification-matrix.md`: generated requirement → test mapping (TST-001).
- `docs/clean-room-log.md`: CR-4 incidents and handover record.
- Each package has a README: purpose, public interface, how to test.

## 4. Configuration keys (initial, extend via SCR)

```
DATABASE_URL, S3_ENDPOINT, S3_BUCKET_PREFIX, OIDC_ISSUER, OIDC_CLIENT_ID, OIDC_CLIENT_SECRET_FILE,
SESSION_SECRET_FILE, WEB_ORIGIN, SPEECH_URL, MODEL_REGISTRY_FILE, PDP_POLICY_DIR, AUDIT_SIEM_URL,
OTEL_EXPORTER_OTLP_ENDPOINT, CONNECTOR_REGISTRY_FILE, FEATURE_FLAGS, REDZONE_KEYWORDS_FILE
```
Secrets come from files (`*_FILE`), never inline environment values in production.

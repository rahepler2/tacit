# 00 · Clean-room protocol

## Purpose

The product is built fresh from this specification. The clean-room process gives three things:

1. **Provenance hygiene:** every line of product code can be traced to this spec, public library documentation or the implementer's own work.
2. **Architectural freedom:** implementers aren't anchored on a prototype that was built for a different context (a single-user hackathon demo).
3. **Auditability:** nuclear customers do supplier and software-provenance reviews. A documented clean-room process answers those questions directly.

> Note: the earlier prototype was MIT-licensed, so this process isn't a legal necessity for licensing. We follow it for the reasons above.

## Roles

| Role | Who | May see the prototype? | May see this spec? |
|---|---|---|---|
| **Spec author** ("dirty room") | The author(s) of this spec | Yes | Writes it |
| **Spec custodian** | Human lead or the integrator agent | Not needed | Yes; owns change control |
| **Implementer** ("clean room") | All coding agents and engineers on the build | **No** | Yes |
| **Reviewer** | Code reviewers, human or agent | **No** | Yes |

## Rules for implementers (agents MUST follow these)

- **CR-1:** Do not access, clone, fetch, search or read the prototype repositories (`rahepler2/tacit`, `vnmoorthy/tacit`, forks or mirrors), their published demo sites, slides, videos or issue trackers.
- **CR-2:** Do not search the web or code-search for the prototype's code, prompts or identifiers.
- **CR-3:** Work only from: this `spec/` folder, public documentation of the libraries and standards named in the spec, public documentation of target enterprise systems, and the synthetic fixtures defined in 19.
- **CR-4:** If you come across prototype material accidentally, stop. Record the incident in `docs/clean-room-log.md` (what, when, how), don't use the material, and tell the custodian.
- **CR-5:** All prompts used by the product (interviewer, extractor, etc.) are written fresh from the *behavioural* requirements in this spec. The spec deliberately describes behaviour, not prompt text.
- **CR-6:** Every commit is attributable to a workstream and to requirement IDs (21).

## Setting up the clean room (do this before the build starts)

1. Create a **new, empty repository** for the product (for example `org/tacit-platform`).
2. Copy **only** the `spec/` folder into it, as the first commit. Do **not** copy anything else from the prototype repository, including `docs/research/`, which discusses the prototype's internals.
3. Give implementer agents access **only** to the new repository. Their tool permissions MUST NOT include the prototype repositories.
4. Record in `docs/clean-room-log.md`: the date, the spec version handed over, the list of implementers, and a statement that the prototype was withheld.
5. The custodian keeps the spec-change log (`spec/changes/`) as the only channel from the dirty room to the clean room.

## What this spec intentionally does not contain

- No source code, prompt text, file paths or identifiers taken from the prototype.
- Ideas and requirements the prototype also had (for example "interview the expert one question at a time", or "an unanswered successor question goes to the next interview") are described in fresh, behavioural terms. They're general ideas, not copied expression.

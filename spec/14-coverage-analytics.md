# 14 · Coverage, risk & analytics

## 1. Inputs

- **Risk register** (`risk_entry`) imported from CSV/XLSX or the HR connector: principal, position, attrition score (1–5), position score (1–5).
- **JTA** (`position`, `job_task`) imported from CSV/XLSX or the LMS connector.
- **Departure dates** from HR.

- **COV-001** Importers validate files against a published template (columns, types), show a dry-run diff, and apply in one transaction. Templates are documented in `docs/import-templates/`.

## 2. Campaign proposals

- **COV-010** Priority for proposing campaigns: `risk_total × urgency`, where `urgency = clamp(1 − days_to_departure / 730, 0.2, 1)`. Principals with `risk_total ≥ threshold` (default 12) and no campaign are listed as proposals on the programme dashboard.

## 3. Coverage plan & scoring

- **COV-020** On campaign creation, coverage areas come from: (1) the position's JTA tasks (kind `job_task`, priority from task criticality); (2) the dossier's suggested systems and document sets (AGT-010); (3) an LLM plan from the role and context, for areas not covered by 1–2 (kind `topic`); (4) one `stewardship` area per catalog category that has items. The facilitator edits and approves the plan.
- **COV-021** Each area gets 2–5 target questions (generated, editable).
- **COV-022** Coverage score per area (0–1):
```
depth        = min(1, Σ_released_or_in_review KI weight / target_depth)      // weight: procedure/decision/cue/rationale 1.0; rule/gotcha/risk 0.8; contact/tool/glossary/history 0.4
breadth      = distinct KI types present among {procedure|rule, cue, decision|rationale, risk|gotcha, contact} / 5
corroboration= share of KIs with ≥ 2 independent sources (two experts, or expert + document edge)
answerability= eval pass rate on held-out questions tagged to the area (if any; else omit and renormalize)
targets      = share of target questions asked and resolved

coverage = 0.30·depth + 0.20·breadth + 0.15·corroboration + 0.20·answerability + 0.15·targets
```
  `target_depth` defaults to 8 for `job_task` areas with criticality ≥ 4, and 5 otherwise. All weights are configurable.
- **COV-023** Stewardship areas: coverage = share of items `closed`.
- **COV-024** Campaign coverage = priority-weighted mean of area coverage (weights: priority 1→5, 2→4, 3→3, 4→2, 5→1).
- **COV-025** **Saturation:** an area is `saturated` when, over the last 2 sessions that touched it, new non-duplicate KIs per 10 minutes on it fall below 0.5 **and** coverage ≥ 0.7. The facilitator can override.

## 4. Programme analytics

- **COV-030** Programme dashboard metrics:
  - risk-weighted coverage = Σ(campaign coverage × risk_total) / Σ risk_total over active and closed campaigns
  - stewardship closure rate
  - open stewardship items by function and due status
  - orphaned artifacts
  - departures in the next 90/180/365 days without campaigns
  - question backlog (by source, age)
  - reviewer backlog and age
  - twin answerability trend
- **COV-031** No per-person productivity or performance rankings (REG-026). Metrics about experts are campaign progress, not people.
- **COV-032** All analytics respect the PDP: aggregates are computed only over resources the viewer may read, or from non-content counts the tenant admin has marked as non-sensitive.

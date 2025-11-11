---
spec_version: "1.0"
agent_id: "ai_maintenance_reports"
role: "Automated maintenance and health reports storage"

policies:
  goals_ref: /doc_agent/policies/goals.md
  safety_ref: /doc_agent/policies/safety.md

parent_agent: /ai/agent.md
merge_strategy: "child_overrides_parent"
---
# Maintenance Reports Agent Guide

> Auto-generated health, optimization, and audit reports land here. Treat the folder as evidence, not as a place for manual edits.

## Scope
- `health-report-*.json` and `health-summary*.md` from `make health_check` or `make health_report`.
- Optimization or audit markdown from targeted maintenance tasks.
- Smart cleanup via `make cleanup_reports_smart`.

## Context Shortcuts
| Need | Load | Purpose |
| --- | --- | --- |
| Overview | `/ai/maintenance_reports/README.md` | File naming and retention rules |
| Cleanup policy | `/doc_human/policies/TEMP_FILES_POLICY.md` | Repo wide storage guardrails |
| Health model | `/doc_agent/specs/HEALTH_CHECK_MODEL.yaml` | Field definitions when reading JSON |

## Usage
1. Run the maintenance command; do not edit outputs.
2. Keep the latest summary (`health-summary.md`) in repo, archive aged JSON or markdown if older than 30 days.
3. Move any decision-support doc you author into workdocs or `/doc_human/` instead.

## Commands
```bash
make ai_maintenance
make health_check
make health_report
python scripts/context_usage_tracker.py report
python scripts/context_usage_tracker.py optimize --agent agent.md
make cleanup_reports_smart
```

## Safety
- Never hand edit generated files; rerun the command instead.
- Limit committed artifacts to summaries or reports referenced by humans.
- Remove temporary exports before closing a task.

## References
- `/ai/maintenance_reports/README.md`
- `/doc_human/policies/TEMP_FILES_POLICY.md`
- `/doc_agent/specs/HEALTH_CHECK_MODEL.yaml`

Version control is sufficient; no per-agent version tag.


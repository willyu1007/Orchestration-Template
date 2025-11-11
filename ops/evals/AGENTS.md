---
spec_version: "1.0"
agent_id: "evals"
role: "Evaluation baseline and benchmark management"

policies:
  goals_ref: /doc_agent/policies/goals.md
  safety_ref: /doc_agent/policies/safety.md

parent_agent: /agent.md
merge_strategy: "child_overrides_parent"

context_routes:
  on_demand:
    - topic: "Repository Health Check"
      priority: medium
      paths:
        - /doc_agent/specs/HEALTH_CHECK_MODEL.yaml
    - topic: "Testing Standards"
      priority: medium
      paths:
        - /doc_agent/coding/TEST_STANDARDS.md
---
# Evals Agent Guide

> Lightweight home for baseline JSON and markdown used to detect regressions.

## Scope
- Store contract, performance, and quality baselines under `evals/`.
- Keep automated health snapshots that inform `make health_check`.
- Archive dated artifacts under `evals/archive/` when they are no longer active.

## Context Shortcuts
| Need | Load | Purpose |
| --- | --- | --- |
| Baseline structure | `/evals/README.md` | Directory policy and examples |
| Health scoring | `/doc_agent/specs/HEALTH_CHECK_MODEL.yaml` | How scores are computed |
| Contract history | `/.contracts_baseline/README.md` | Expectations for compatibility runs |

## Flow
1. Plan the baseline category (contract, performance, quality, api samples).
2. Generate data through the matching script (`make contract_baseline`, `make health_check`, etc).
3. Store the newest file under `evals/` and archive older copies.
4. Reference the file from PR descriptions or workdocs when asserting no regressions.

## Commands
```bash
make contract_baseline        # Refresh contract snapshot
make contract_compat_check    # Compare contracts vs baseline
make health_check             # Produce JSON plus summary
make health_report            # Longer markdown summary
```

## Retention Rules
- Keep only the latest baseline per category at the top level.
- Move historical snapshots into `evals/archive/` with a date suffix.
- Do not edit JSON manually; rerun the generator instead.
- Never commit secrets or production data.

## References
- `/evals/README.md`
- `/doc_agent/specs/HEALTH_CHECK_MODEL.yaml`
- `/.contracts_baseline/README.md`

Version metadata lives with the baseline files; this agent does not need its own version.


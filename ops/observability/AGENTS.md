---
spec_version: "1.0"
agent_id: "observability"
role: "Logging, metrics, tracing, and alerting configurations"

policies:
  goals_ref: /doc_agent/policies/goals.md
  safety_ref: /doc_agent/policies/safety.md

parent_agent: /agent.md
merge_strategy: "child_overrides_parent"

context_routes:
  always_read:
    - /observability/README.md
  on_demand:
    - topic: "Observability Setup"
      priority: medium
      paths:
        - /doc_human/guides/HEALTH_MONITORING_GUIDE.md
---
# Observability Agent Guide

> These are reference templates for logging, metrics, tracing, and alerting. Copy what you need, customize per environment, and keep tokens light.

## Scope
- Provide starting configs for Logstash or Fluentd, Prometheus or Grafana, Jaeger, and Alertmanager.
- Document best practices without forcing production deployments.
- Keep everything language neutral and secret free.

## Directory Map
| Folder | Purpose | Notes |
| --- | --- | --- |
| `logging/` | Log pipelines (Logstash, Fluentd, python logging) | Edit before shipping |
| `metrics/` | Prometheus scrape config plus Grafana dashboards | Validate with promtool |
| `tracing/` | Jaeger and OpenTelemetry collectors | Wire spans into services |
| `alerts/` | Prometheus rules plus Alertmanager routing | Update contacts before use |

## Usage
1. Duplicate the needed template into your environment repo or Helm chart.
2. Replace placeholder hosts, secrets, and webhook URLs.
3. Validate configs locally (`promtool`, `amtool`, `logstash -t`).
4. Document alert runbooks inside `/doc_human/guides/HEALTH_MONITORING_GUIDE.md`.

## Commands
```bash
make observability_check
promtool check config observability/metrics/prometheus.yml
promtool check rules observability/alerts/prometheus_alerts.yml
```

## Safety
- Never commit real credentials here.
- Remove unused templates after copying to avoid noise.
- Keep alert volumes reasonable to prevent fatigue.

## References
- `/observability/README.md`
- `/doc_human/guides/HEALTH_MONITORING_GUIDE.md`
- `/doc_agent/quickstart/guardrail-quickstart.md`

Versioned via git; no per-agent version tag required.


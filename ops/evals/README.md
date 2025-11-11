# Observability Templates

This folder provides reference configs for logging, metrics, tracing, and alerting. Copy what you need, delete the rest.

## Structure
```text
observability/
|-- logging/         # Logstash, Fluentd, python logging config
|-- metrics/         # Prometheus config + Grafana dashboards
|-- tracing/         # Jaeger + OpenTelemetry settings
`-- alerts/          # Prometheus alert rules + Alertmanager routes
```

## How To Use
1. Decide which stack (Prometheus, Grafana, ELK, Jaeger, etc.) you actually run.
2. Copy the relevant files to your deployment repo or Helm chart.
3. Replace placeholder hosts, Slack webhooks, and credentials.
4. Remove unused directories to keep AI context lean.

## Quick Checks
```bash
# Logs
echo '{"message":"test"}' | nc logstash 5044
curl http://fluentd:24220/api/plugins.json

# Metrics
curl http://prometheus:9090/api/v1/targets
curl 'http://prometheus:9090/api/v1/query?query=up'

# Tracing
curl http://jaeger:16686/api/services

# Alerts
amtool check-config observability/alerts/alertmanager.yml
```

## Docker Compose Stub
See `docker-compose.observability.yml` snippet in this file if you want a local playground for Prometheus + Grafana + Jaeger + ELK. Adjust volumes and ports before using it in production.

## Language Policy
Logs, dashboards, and alert messages should follow the repository language (English by default). Update `config/language.yaml` if your team uses a different language and regenerate alert text accordingly.

## Related Docs
- Incident response: `modules/common/doc/RUNBOOK.md`.
- Health monitoring guide: `doc_human/guides/HEALTH_MONITORING_GUIDE.md`.
- Guardrail references: `doc_agent/quickstart/guardrail-quickstart.md`.

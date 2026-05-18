# Monitoring Stack Integration Guide

This guide explains the observability stack in `monitoring/` and how each backend service should integrate so logs, traces, and metrics are visible in Grafana dashboards and Drilldown.

## 1. Tech Stack

This monitoring setup uses:
- Grafana `12.0.0` (UI, dashboards, Drilldown)
- Tempo `2.8.2` (distributed tracing)
- Loki `3.5.0` (centralized logs)
- Prometheus `3.3.0` (metrics store)
- OpenTelemetry Collector Contrib (OTLP ingest + processing)
- Grafana Alloy (OTLP logs gateway to Loki)
- Docker Compose (orchestration)

Core config files:
- `docker-compose.yml`
- `otel-collector/config.yaml`
- `alloy/config.alloy`
- `tempo/config.yaml`
- `loki/config.yaml`
- `prometheus/prometheus.yml`
- `grafana/provisioning/*`

## 2. High-Level Architecture

Telemetry flow:
1. **Traces + metrics** are sent from backend services to **OTel Collector** (`4317/4318`).
2. OTel Collector forwards traces to **Tempo** and metrics to **Prometheus scrape endpoint**.
3. **Logs** are sent from backend services to **Alloy** (`4319/4320`) and then forwarded to **Loki**.
4. **Grafana** reads from Tempo/Loki/Prometheus using provisioned datasources.

## 3. Ports and Endpoints

Exposed by monitoring stack:
- Grafana UI: `3000`
- OTel Collector OTLP gRPC: `4317`
- OTel Collector OTLP HTTP: `4318`
- Alloy OTLP gRPC (logs): `4319`
- Alloy OTLP HTTP (logs): `4320`

Internal-only in compose network:
- Tempo API: `3200`
- Loki API: `3100`
- Prometheus API: `9090`

## 4. Start and Stop

From `monitoring/`:

```bash
docker compose up -d
```

Stop:

```bash
docker compose down
```

Grafana login:
- URL: `http://localhost:3000`
- User: `admin`
- Password: `admin`

## 5. Backend Integration Requirements

Every backend service must do all of the following:

1. Instrument app with OpenTelemetry SDK/agent.
2. Set stable service identity (`service.name`).
3. Export traces and metrics to OTel Collector (`4317` or `4318`).
4. Export logs through OTLP to Alloy (`4319` or `4320`).
5. Include trace context in logs when possible (`trace_id`, `span_id`).

If any one of these is missing, correlation in Grafana will be partial.

## 6. Recommended Environment Variables

Use these as baseline in each backend service:

```bash
OTEL_SERVICE_NAME=<service-name>
OTEL_RESOURCE_ATTRIBUTES=service.name=<service-name>,service.namespace=<team-or-domain>,deployment.environment=<env>

OTEL_TRACES_EXPORTER=otlp
OTEL_METRICS_EXPORTER=otlp
OTEL_LOGS_EXPORTER=otlp

# Traces + metrics -> OTel Collector
OTEL_EXPORTER_OTLP_ENDPOINT=http://<monitoring-host>:4318
# Optional explicit protocol
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf

# Logs -> Alloy (recommended explicit logs endpoint)
OTEL_EXPORTER_OTLP_LOGS_ENDPOINT=http://<monitoring-host>:4320/v1/logs
```

Notes:
- If using gRPC exporter, target `4317` for collector and `4319` for logs gateway.
- `service.name` must be non-empty and consistent across app instances.
- Prefer explicit `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT` so logs do not accidentally go to the traces/metrics path.

## 7. Docker Compose Integration Patterns

### Pattern A: Service runs on host machine
Use `localhost` as `<monitoring-host>`.

### Pattern B: Service runs in another compose project
Use one of these approaches:
1. Attach service to the same external Docker network as monitoring.
2. Use host gateway DNS (`host.docker.internal`) if supported.
3. Expose required monitoring ports and target host IP.

Minimum required reachability:
- `4317/4318` for traces/metrics
- `4319/4320` for logs

## 8. Verify Integration End-to-End

After wiring env vars:

1. Generate traffic to backend endpoints (GET/POST that hit business flow).
2. Verify logs in Grafana Explore (Loki datasource) using label filters.
3. Verify traces in Grafana Explore (Tempo datasource) using query like:
   - `{resource.service.name="<service-name>"}`
4. Open Drilldown > Traces and set time range to last 15-30 min.

Optional API-level verification from host:

```bash
# Tempo search
curl -s "http://localhost:3200/api/search?q=%7Bresource.service.name%3D%22<service-name>%22%7D&limit=5"

# Tempo TraceQL metrics query used by Drilldown
curl -sG "http://localhost:3200/api/metrics/query_range" \
  --data-urlencode 'q={nestedSetParent<0 && resource.service.name != nil} | rate() by(resource.service.name)' \
  --data-urlencode 'start=2026-05-18T00:00:00Z' \
  --data-urlencode 'end=2026-05-18T01:00:00Z' \
  --data-urlencode 'step=15s'
```

If this returns non-empty `series`, Drilldown backend is healthy.

## 9. Drilldown Requirements (Important)

Grafana Traces Drilldown requires Tempo metrics-generator local blocks. This repository already configures it in `tempo/config.yaml`.

Critical settings already enabled:
- `overrides.defaults.metrics_generator.processors` includes:
  - `service-graphs`
  - `span-metrics`
  - `local-blocks`
- `metrics_generator.processor.local_blocks` enabled
- `metrics_generator.traces_storage.path` configured

If you edit processor names, keep the hyphenated forms exactly as above.

## 10. Common Issues and Fixes

### Issue: Drilldown shows "No data for selected query"
Checklist:
1. Ensure backend is actually receiving traffic in selected time range.
2. Ensure service exports traces to collector.
3. Ensure `service.name` exists in spans.
4. Check Tempo logs for metrics-generator errors.
5. Verify query returns non-empty `series` via Tempo API.

### Issue: `trace_id` in logs is always empty
- App logging pattern is not wired to OTel context.
- Add MDC/context integration in logging framework so trace/span ids are injected.

### Issue: Logs visible but traces missing
- Logs pipeline can work while traces pipeline is broken.
- Re-check OTLP endpoint for traces (`4317/4318`) and exporter protocol.

### Issue: Traces visible in Explore but not Drilldown
- Usually metrics-generator config or time/filter mismatch.
- Confirm local-blocks processor is active and query window includes fresh traces.

## 11. Team Integration Checklist

For each new backend service, complete this checklist:
- [ ] OTel instrumentation added
- [ ] `service.name` set and stable
- [ ] Traces exported to collector
- [ ] Metrics exported to collector
- [ ] Logs exported to alloy
- [ ] Trace context appears in logs
- [ ] Service appears in Tempo search
- [ ] Service appears in Loki logs
- [ ] Drilldown shows span rate for the service

## 12. Security and Production Notes

Before non-local environments:
1. Change default Grafana credentials.
2. Restrict public exposure of ingestion ports.
3. Put Grafana behind auth proxy/SSO if needed.
4. Add persistent volumes backup strategy.
5. Apply retention, sampling, and cardinality controls.

---

If you modify monitoring config, always restart stack and re-verify with fresh traffic:

```bash
docker compose up -d
docker compose restart tempo otel-collector grafana alloy
```

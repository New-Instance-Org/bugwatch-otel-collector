# BugWatch OpenTelemetry Collector distribution

A drop-in OpenTelemetry Collector configuration that forwards your apps' **logs, traces, and metrics** to BugWatch over OTLP. Apps point their OTLP exporter at this collector; the collector forwards to BugWatch's OTLP-compatible ingestion (`/v1/logs`, `/v1/traces`, `/v1/metrics` — HTTP + gRPC).

> **Don't have a BugWatch project yet?** Create one at **[newinstance.cloud](https://www.newinstance.cloud)** and copy your project's central API key (`sk_live_…` / `sk_test_…`) from the dashboard — that's the `BUGWATCH_API_KEY` below.

## Quick start

```bash
export BUGWATCH_API_KEY="sk_live_xxx:your-secret"     # your central key (one project+env)
export BUGWATCH_HTTP="https://api.newinstance.cloud"  # dev: http://localhost:5050
docker compose up
```

Then configure your apps' OTLP exporter endpoint to this collector:
- gRPC: `http://<collector-host>:4317`
- HTTP: `http://<collector-host>:4318`

No app code changes beyond standard OTLP exporter config — the collector handles auth + routing to BugWatch.

## What it does

| Signal | Receiver | → BugWatch endpoint |
|---|---|---|
| Logs | OTLP (4317/4318) | `POST {BUGWATCH_HTTP}/v1/logs` |
| Traces | OTLP (4317/4318) | `POST {BUGWATCH_HTTP}/v1/traces` |
| Metrics | OTLP (4317/4318) | `POST {BUGWATCH_HTTP}/v1/metrics` |

Auth is the BugWatch central API key, sent as the `x-api-key` header (it binds telemetry to one project + environment, exactly like the native SDK). `memory_limiter` + `batch` processors give backpressure and efficient batching.

## Prometheus metrics (alternative to OTLP metrics)

If you already run Prometheus, push to BugWatch via `remote_write` instead of (or in addition to) the OTLP metrics pipeline:

```yaml
# prometheus.yml
remote_write:
  - url: "https://api.newinstance.cloud/api/v1/prom/write"
    headers:
      x-api-key: "sk_live_xxx:your-secret"
```

And query BugWatch as a Prometheus datasource in Grafana:
`https://api.newinstance.cloud/api/v1/prom` (supports `/query_range`, `/query`, `/label/__name__/values`).

## Jaeger UI

Point a Jaeger query/UI at `https://api.newinstance.cloud/jaeger/api` (`/services`, `/traces/{id}`, `/traces`) to browse BugWatch traces in Jaeger.

## Notes / scope

- This distribution is the **upstream `otel/opentelemetry-collector-contrib`** image + the `config.yaml` here — no forked binary. All routing targets are BugWatch endpoints proven live (OTLP logs/traces/metrics ingestion + Prometheus remote-write).
- To bump the collector version, change the image tag in `docker-compose.yml`.
- `config.yaml` is standard otelcol config; validate locally with `otelcol-contrib validate --config config.yaml` (requires the binary/Docker, not bundled here).

## Learn more

- **BugWatch / NewInstance:** https://www.newinstance.cloud
- Once you create a project, its dashboard shows a per-project setup guide with this exact config **pre-filled** (OTLP endpoint + API key already substituted).

## License

[Apache-2.0](./LICENSE).

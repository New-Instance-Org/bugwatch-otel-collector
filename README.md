# BugWatch OpenTelemetry Collector

A thin OpenTelemetry Collector relay that forwards your application's OTLP
**logs, traces, and metrics** to BugWatch. Your apps talk standard OTLP to
the collector; the collector adds your API key and forwards everything to
BugWatch's ingestion endpoints.

No forked binary. No custom code. Just the upstream
`otel/opentelemetry-collector-contrib:0.103.0` image with a ready-to-run
`config.yaml`.

---

## Data flow

```
Your app(s)
  │
  │  OTLP/gRPC  :4317
  │  OTLP/HTTP  :4318
  ▼
bugwatch-otel-collector
  │  memory_limiter + batch processors
  │
  │  OTLP/HTTP  (x-api-key: <your key>)
  ▼
BugWatch ingest
  POST ${BUGWATCH_HTTP}/v1/logs
  POST ${BUGWATCH_HTTP}/v1/traces
  POST ${BUGWATCH_HTTP}/v1/metrics
```

All three OTLP signal types (logs, traces, metrics) share the same two
receiver ports. The collector adds your `x-api-key` authentication header
before forwarding — your apps never need to know the key.

---

## Requirements

- Docker + Docker Compose **or** any host that can run the
  `otel/opentelemetry-collector-contrib` binary.
- A BugWatch project with an API key that has the `ingest:write` scope (see
  below).

---

## Quick start

**1. Get your API key**

Log in to [newinstance.cloud](https://www.newinstance.cloud) → your
organization → the project you want to send telemetry to → **API Keys** →
create or copy a central key. The key format is `<keyId>:<secret>` (e.g.
`sk_live_abc123:your-secret`). The key must have the `ingest:write` scope.

> The BugWatch project dashboard also has a **Setup guide** tab with this
> exact config pre-filled (endpoint + key already substituted).

**2. Run the collector**

```bash
export BUGWATCH_API_KEY="sk_live_abc123:your-secret"
export BUGWATCH_HTTP="https://api.newinstance.cloud"   # local dev: http://localhost:5050

docker compose up
```

The collector starts and listens on `:4317` (gRPC) and `:4318` (HTTP).

**3. Point your app at the collector**

Set your app's OTLP exporter endpoint to the collector host:

| Transport | Endpoint |
|-----------|----------|
| gRPC | `http://<collector-host>:4317` |
| HTTP | `http://<collector-host>:4318` |

When running locally via Docker Compose on the same machine: use
`http://localhost:4317` or `http://localhost:4318`.

No other app changes are needed — the collector handles auth and routing.

---

## Point your app at the collector

Set these standard OTLP environment variables (supported by all major OTel
SDKs):

```bash
# HTTP transport (simpler, works through load balancers)
OTEL_EXPORTER_OTLP_ENDPOINT=http://<collector-host>:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf

# --- or gRPC ---
OTEL_EXPORTER_OTLP_ENDPOINT=http://<collector-host>:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
```

### Example: Node.js

```ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { OTLPLogExporter } from "@opentelemetry/exporter-logs-otlp-http";

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: "http://localhost:4318/v1/traces",
  }),
  logRecordProcessor: new SimpleLogRecordProcessor(
    new OTLPLogExporter({ url: "http://localhost:4318/v1/logs" })
  ),
});
sdk.start();
```

No API key in the app — the collector inserts it before forwarding to
BugWatch.

### Example: Python

```python
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace.export import BatchSpanProcessor

exporter = OTLPSpanExporter(endpoint="http://localhost:4318/v1/traces")
provider.add_span_processor(BatchSpanProcessor(exporter))
```

### Example: Go

```go
exporter, _ := otlptracehttp.New(ctx,
    otlptracehttp.WithEndpoint("localhost:4318"),
    otlptracehttp.WithInsecure(),
)
```

> **Go `WithEndpoint` convention:** the Go OTLP SDK takes `host:port` only — it appends `/v1/traces` (or `/v1/logs`, `/v1/metrics`) automatically. The full-URL form (`http://localhost:4318/v1/traces`) used in the curl, Node.js, and Python examples above is correct for those clients but **not** for `otlptracehttp.WithEndpoint`.

---

## Config reference

`config.yaml` is a standard OTel Collector config with three sections:

### Receivers

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317   # accepts OTLP gRPC from any app
      http:
        endpoint: 0.0.0.0:4318   # accepts OTLP HTTP from any app
```

Both protocols are active by default. Use whichever your SDK supports.

### Processors

```yaml
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 512        # hard cap — collector drops data rather than OOM
    spike_limit_mib: 128  # extra headroom for bursts

  batch:
    send_batch_size: 512  # flush after 512 items
    timeout: 5s           # or after 5 seconds, whichever comes first

  resource:
    attributes:
      - key: collector
        value: bugwatch-otel-collector
        action: upsert    # stamps a resource attribute on every signal
```

The order in each pipeline is: `memory_limiter → resource → batch`. The
`memory_limiter` must come first so it can shed load before downstream
processing.

### Exporters

```yaml
exporters:
  otlphttp/bugwatch:
    logs_endpoint:    ${env:BUGWATCH_HTTP}/v1/logs
    traces_endpoint:  ${env:BUGWATCH_HTTP}/v1/traces
    metrics_endpoint: ${env:BUGWATCH_HTTP}/v1/metrics
    headers:
      x-api-key: ${env:BUGWATCH_API_KEY}   # central key — never in your app
    compression: gzip

  debug:
    verbosity: basic   # prints pipeline activity to stdout
```

### Pipelines

```yaml
service:
  pipelines:
    logs:
      receivers:  [otlp]
      processors: [memory_limiter, resource, batch]
      exporters:  [otlphttp/bugwatch]
    traces:
      receivers:  [otlp]
      processors: [memory_limiter, resource, batch]
      exporters:  [otlphttp/bugwatch]
    metrics:
      receivers:  [otlp]
      processors: [memory_limiter, resource, batch]
      exporters:  [otlphttp/bugwatch]
```

Each signal type has its own isolated pipeline. Add `debug` to any
exporter list temporarily to see what is flowing through.

---

## Environment variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `BUGWATCH_API_KEY` | Yes | — | Central key in `keyId:secret` format. Needs `ingest:write` scope. |
| `BUGWATCH_HTTP` | No | `https://api.newinstance.cloud` | BugWatch OTLP/HTTP base URL. |
| `BUGWATCH_GRPC` | No | `api.newinstance.cloud:4317` | BugWatch OTLP/gRPC host:port (only if you switch the exporter to gRPC). |

---

## Prometheus metrics (alternative for metrics)

If you already run Prometheus, use `remote_write` instead of (or alongside)
the OTLP metrics pipeline:

```yaml
# prometheus.yml
remote_write:
  - url: "https://api.newinstance.cloud/api/v1/prom/write"
    headers:
      x-api-key: "sk_live_abc123:your-secret"
```

Query BugWatch as a Prometheus datasource in Grafana:
`https://api.newinstance.cloud/api/v1/prom`
(supports `/query_range`, `/query`, `/label/__name__/values`).

---

## Jaeger UI

Point a Jaeger Query / UI at `https://api.newinstance.cloud/jaeger/api`
(`/services`, `/traces/{id}`, `/traces`) to browse BugWatch traces in Jaeger.

---

## Kubernetes deployment

The manifests in [`k8s/`](./k8s/) deploy the same image + config as a
Kubernetes Deployment. The collector runs as a shared relay — all pods in
the cluster point their OTLP exporter at the collector's ClusterIP Service.

### Files

| File | Purpose |
|------|---------|
| `k8s/namespace.yaml` | Dedicated `bugwatch` namespace |
| `k8s/secret.yaml` | Stores `BUGWATCH_API_KEY` as a K8s Secret |
| `k8s/configmap.yaml` | Mounts `config.yaml` into the collector |
| `k8s/deployment.yaml` | Collector Deployment (2 replicas) + Service |

### Apply

```bash
# 1. Set your key (base64-encode it for the Secret)
export BUGWATCH_API_KEY="sk_live_abc123:your-secret"

# 2. Apply in order
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml

# Patch the secret with your real key, then apply
sed "s|<base64-encoded-key>|$(echo -n "$BUGWATCH_API_KEY" | base64)|g" \
    k8s/secret.yaml | kubectl apply -f -

kubectl apply -f k8s/deployment.yaml

# 3. Verify
kubectl -n bugwatch rollout status deployment/bugwatch-otel-collector
kubectl -n bugwatch get pods
```

### Point your in-cluster apps at the collector

```bash
# gRPC
OTEL_EXPORTER_OTLP_ENDPOINT=http://bugwatch-otel-collector.bugwatch.svc.cluster.local:4317

# HTTP
OTEL_EXPORTER_OTLP_ENDPOINT=http://bugwatch-otel-collector.bugwatch.svc.cluster.local:4318
```

Or simply use the short name `bugwatch-otel-collector.bugwatch` if your
pods are DNS-searching the cluster domain.

---

## Config validation

Before deploying a modified `config.yaml`, validate it with the otelcol
binary:

```bash
# Using Docker (no local install required)
docker run --rm \
  -v "$(pwd)/config.yaml:/etc/otelcol/config.yaml:ro" \
  otel/opentelemetry-collector-contrib:0.103.0 \
  validate --config=/etc/otelcol/config.yaml

# Or if you have otelcol-contrib installed locally
otelcol-contrib validate --config config.yaml
```

A valid config prints nothing and exits 0. Any schema or reference error
is printed with the offending line.

---

## Scaling and resource limits

### Memory limiter

The `memory_limiter` processor in `config.yaml` caps the collector at
**512 MiB** with a 128 MiB spike allowance. When memory exceeds the limit
the collector drops the current batch and emits a log line — no OOM kill.

Tune for your workload:

```yaml
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 1024       # raise if you see frequent drop warnings
    spike_limit_mib: 256
```

### Batch processor

```yaml
processors:
  batch:
    send_batch_size: 1024  # items per export call
    timeout: 5s            # maximum wait before flushing a partial batch
```

Larger batches reduce HTTP round-trips to BugWatch at the cost of slightly
higher latency. For high-throughput services (>10k spans/s) raise
`send_batch_size` to 2048–4096.

### Docker Compose resource limits

Add resource constraints to `docker-compose.yml` for the collector service:

```yaml
services:
  bugwatch-otel-collector:
    # ... existing config ...
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 768M
        reservations:
          cpus: "0.25"
          memory: 256M
```

### Kubernetes replicas

The `k8s/deployment.yaml` runs **2 replicas** behind a ClusterIP Service.
Kubernetes distributes inbound connections across replicas. Scale up with:

```bash
kubectl -n bugwatch scale deployment bugwatch-otel-collector --replicas=4
```

Or add a HorizontalPodAutoscaler targeting CPU utilization. Each replica
is stateless — scaling is safe at any time.

---

## TLS

### Collector → BugWatch (outbound)

Production BugWatch (`https://api.newinstance.cloud`) is served over HTTPS.
The `otlphttp/bugwatch` exporter uses HTTPS by default when the URL scheme
is `https://` — no extra config is needed.

For a self-hosted or staging BugWatch instance with a self-signed certificate:

```yaml
exporters:
  otlphttp/bugwatch:
    logs_endpoint:    https://your-bugwatch/v1/logs
    traces_endpoint:  https://your-bugwatch/v1/traces
    metrics_endpoint: https://your-bugwatch/v1/metrics
    headers:
      x-api-key: ${env:BUGWATCH_API_KEY}
    tls:
      ca_file: /etc/ssl/certs/your-ca.crt   # mount the CA into the container
    compression: gzip
```

Mount the CA file into the Docker container:

```yaml
# docker-compose.yml
volumes:
  - ./config.yaml:/etc/otelcol/config.yaml:ro
  - ./your-ca.crt:/etc/ssl/certs/your-ca.crt:ro
```

### App → collector (inbound)

The collector's OTLP receivers accept plain HTTP/gRPC by default (suitable
for in-cluster or localhost use). To enable TLS on the receivers (e.g.
collector exposed on a public port):

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        tls:
          cert_file: /etc/ssl/certs/collector.crt
          key_file:  /etc/ssl/private/collector.key
      http:
        endpoint: 0.0.0.0:4318
        tls:
          cert_file: /etc/ssl/certs/collector.crt
          key_file:  /etc/ssl/private/collector.key
```

Mount the certificate and key files into the container the same way as the
CA file above.

> **Recommendation:** for Kubernetes deployments, terminate TLS at an
> ingress or service mesh (e.g. Istio mTLS) and keep the collector's
> inbound receivers plain HTTP inside the cluster.

---

## Troubleshooting

**No data in BugWatch after starting the collector**

1. Check collector logs: `docker compose logs bugwatch-otel-collector`
   — look for export errors or `Dropping data` messages.
2. Verify `BUGWATCH_API_KEY` is set and has `ingest:write` scope (the
   BugWatch dashboard shows scope on the API key detail page).
3. Confirm your app is sending to the right port and host. Test with:
   ```bash
   curl -v http://localhost:4318/v1/logs \
     -H "Content-Type: application/json" \
     -d '{}'
   ```
   A `400 Bad Request` (not a connection error) means the collector is
   reachable.
4. Temporarily add `debug` to a pipeline's exporters to log every payload:
   ```yaml
   exporters:
     debug:
       verbosity: detailed
   service:
     pipelines:
       logs:
         exporters: [otlphttp/bugwatch, debug]
   ```

**`BUGWATCH_API_KEY` not set error on startup**

The `docker-compose.yml` uses `${BUGWATCH_API_KEY:?set ...}` — Docker
Compose will refuse to start if the variable is missing. Export it before
running:
```bash
export BUGWATCH_API_KEY="sk_live_xxx:your-secret"
docker compose up
```

**`memory_limiter: dropping data` in collector logs**

The collector is receiving more data than it can hold in memory. Options:
- Raise `limit_mib` in `config.yaml` (and the container memory limit to match).
- Reduce the send rate from your apps (raise `OTEL_BSP_MAX_EXPORT_BATCH_SIZE`
  or `OTEL_BLRP_EXPORT_TIMEOUT` in the SDK to spread load).
- Scale to more replicas (Kubernetes) or more Docker containers behind a
  load balancer.

**Config validation fails**

Run the validation command (see [Config validation](#config-validation))
and fix any reported errors before restarting. Common issues:
- Referencing a processor/exporter in a pipeline that is not defined.
- Typo in an env var reference (must be `${env:VAR_NAME}` not `${VAR_NAME}`).

**Kubernetes pods stuck in `CrashLoopBackOff`**

```bash
kubectl -n bugwatch logs deployment/bugwatch-otel-collector
```

Most common cause: the Secret is missing or the key value was double
base64-encoded. Verify with:
```bash
kubectl -n bugwatch get secret bugwatch-api-key -o jsonpath='{.data.BUGWATCH_API_KEY}' | base64 -d
```
The decoded value should be `keyId:secret` — not another base64 string.

---

## Learn more

- **BugWatch / NewInstance:** https://www.newinstance.cloud
- **OpenTelemetry Collector docs:** https://opentelemetry.io/docs/collector/
- **otel/opentelemetry-collector-contrib image:** https://hub.docker.com/r/otel/opentelemetry-collector-contrib
- **OTLP specification:** https://opentelemetry.io/docs/specs/otlp/

---

## License

[Apache-2.0](./LICENSE)

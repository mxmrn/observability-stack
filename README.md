# Observability stack

A shared metrics, traces and logs stack for projects. It lives in its own
docker compose project and runs permanently; projects connect to it through the `observability` docker network.
The only host requirement is Docker.

## Running

```bash
cp .env.example .env            # optional: limits/retention for this machine
docker compose up -d            # creates the observability network and starts the stack
docker compose --profile smoke run --rm smoke   # check: 60 seconds of test traces
```

| What                           | Where                                                                       |
| ------------------------------ | --------------------------------------------------------------------------- |
| Grafana (no login)             | http://localhost:3000                                                       |
| Prometheus                     | http://localhost:9090 (Status → Targets shows what service discovery found) |
| OTLP for processes on the host | `localhost:4317` (gRPC), `localhost:4318` (HTTP)                            |
| OTLP for containers            | `otel-collector:4317` / `:4318`                                             |

Useful commands:

```bash
docker compose kill -s SIGHUP prometheus   # reload prometheus.yml and rules without a restart
docker compose restart otel-collector      # after editing the collector config (same for tempo, loki, alloy)
docker compose down                        # stop; data stays in volumes
docker compose down -v                     # stop and wipe all metrics/traces/logs
```

## Architecture

```
 services ──OTLP──► OTel Collector ─┬─► Tempo ──metrics-generator──┐ (RED from traces)
    │                               ├─► Prometheus ◄───────────────┘
    │                               └─► Loki
    │ /metrics ◄── Prometheus (docker_sd by container labels)
    │ stdout ────► Alloy ──► Loki
 cAdvisor, node-exporter ──► Prometheus     k6 ──remote_write──► Prometheus
                    Grafana: Prometheus + Tempo + Loki, linked to each other
```

| Component          | Role                                                                                                                             |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| **OTel Collector** | Single OTLP entry point. `memory_limiter` → adds `project`/`service` labels → `batch` → routes to backends.                      |
| **Prometheus**     | Metrics. Pull (container `/metrics`) + push (OTLP from the collector, remote_write from Tempo/k6/kind).                          |
| **Tempo**          | Traces. metrics-generator derives RED metrics (`traces_spanmetrics_*`) and the call graph (`traces_service_graph_*`) from spans. |
| **Loki**           | Logs. Indexes only labels (`project`, `service`, `container`); log text is not indexed.                                          |
| **Alloy**          | Reads container stdout through the Docker API and ships it to Loki. Extracts `trace_id` from JSON logs.                          |
| **cAdvisor**       | CPU, throttling, memory, network per container.                                                                                  |
| **node-exporter**  | Host resources.                                                                                                                  |
| **docker-proxy**   | Read-only Docker API access for Prometheus and Alloy (they never get `docker.sock`).                                             |
| **Grafana**        | Datasources and dashboards from `grafana/` (provisioning).                                                                       |

## Connecting a project

Full example: [`templates/compose.project.yml`](templates/compose.project.yml). In short:

```yaml
services:
  api:
    networks: [default, observability]
    labels:
      observability.scrape: "true"
      observability.port: "8080"
    environment:
      OTEL_EXPORTER_OTLP_ENDPOINT: http://otel-collector:4317
      OTEL_SERVICE_NAME: api
      OTEL_RESOURCE_ATTRIBUTES: service.namespace=myproject
networks:
  observability:
    external: true
```

### Container labels

| Label                                             | Meaning                                    |
| ------------------------------------------------- | ------------------------------------------ |
| `observability.scrape`                            | `"true"`: Prometheus scrapes the container |
| `observability.port`                              | metrics port (required)                    |
| `observability.path`                              | path, defaults to `/metrics`               |
| `observability.interval`                          | scrape interval, defaults to `15s`         |
| `observability.project` / `observability.service` | override `project`/`service`               |
| `observability.logs`                              | `"false"`: don't collect stdout into Loki  |

### The `project` / `service` convention

Every signal (metrics, traces, logs) carries these two labels. Dashboards and alerts filter on them.

- Pulled metrics and stdout logs: taken automatically from docker compose labels
  (`com.docker.compose.project` = `name:` in compose, `com.docker.compose.service` = service name).
- OTLP: `service.namespace` → `project`, `service.name` → `service` (done by the collector).
  So `service.namespace` must match the project's `name:`.

## Correlation: metric → trace → log

- **Metric → trace.** The p99 graph in the _Services (RED)_ dashboard shows exemplar dots; click one to open the trace in Tempo.
- **Trace → logs.** Each span has a _Logs for this span_ button: the service's logs around that time with that `trace_id`.
- **Log → trace.** If a service writes JSON logs with a `trace_id` field, Loki shows an _Open trace_ link on the line.
- **Trace → metrics.** _Related metrics_ on a span: RPS and p99 of that service.

For this to work, services must log the current span's `trace_id` (in Go: `span.SpanContext().TraceID()`).

## Dashboards and alerts

- `Services (RED)`: RPS, errors, p50/p95/p99 per service and operation, plus logs. Works for any service
  that sends traces, without dedicated metrics in code.
- `Resources: host & containers`: the top row answers "can I trust this measurement?":
  host CPU and what the stack itself consumes. Below: CPU, throttling, memory/limit, network, OOM per container.
- Alerts: `prometheus/rules/alerts.yml` (visible in Prometheus → Alerts and Grafana → Alerting).
  Notifications (Alertmanager) come with the SLO iteration.

**Everything as code:** dashboard edits made in the UI only last until Grafana re-provisions.
To keep them: Share → Export → JSON → put the file into `grafana/dashboards/<folder>/`.

## Resources and retention

- Retention: Prometheus 3 days / 5 GB, Tempo 72h, Loki 72h.
- Every component has a CPU/memory limit (overridable in `.env`). Idle, the stack uses ~0.05 CPU.
- During load tests watch the top row of the _Resources_ dashboard: if host CPU > 90%,
  you are measuring the laptop, not the service (alert `HostCPUSaturated`). Fixes: move k6 and/or the stack
  to a separate machine, or put a hard `cpus` limit on the load generator.

## Gotchas

- **Start order:** stack first, then projects. Otherwise: `network observability not found`.
- **Replicas and OTLP metrics:** every replica needs its own `service.instance.id`
  (e.g. the container hostname), otherwise their metrics merge into one jumping series.
  Pulled metrics don't have this problem: each container has its own `instance`.
- **Cardinality:** never put addresses, tx hashes or user ids into metric labels: every unique
  value is a new series in Prometheus. They belong in span attributes and logs.
- **Tempo log noise** `error calling scheduler ... no jobs found` is normal for monolithic
  Tempo 3: the backend worker polls the compaction scheduler and there is no work.
- **Docker Desktop (macOS/Windows):** cAdvisor and node-exporter show the resources of Docker's Linux VM,
  not of the machine itself. Everything else works the same.
- **Config edits not picked up / "no such file" on reload:** bind mounts point at the directory that existed
  when the container started. If this directory was deleted and re-created (re-clone, move, some git operations),
  running containers see an empty old copy. Fix: `docker compose up -d --force-recreate` (data in volumes is kept).
- **A target silently missing from Prometheus:** check Status → Targets and `docker compose logs prometheus`.
  Relabeling errors (e.g. a scrape timeout larger than the interval) are only logged, the target is just not created.

## Next: Kubernetes (kind)

The stack stays here in compose; a kind cluster connects to it as a client:
an in-cluster agent (Alloy/OTel Collector) discovers pods via the Kubernetes API and ships data to
this stack (OTLP to the collector, remote_write to Prometheus; the receiver is already enabled).
That way metric history survives cluster re-creation. To be set up when we get to the k8s topics.

# Docker Monitoring Stack

Local observability for Linux hosts running rootful Docker Engine:

- Grafana: dashboards and visualization
- Prometheus: metrics storage
- cAdvisor: container CPU, memory and network metrics
- Node Exporter: host metrics
- Loki and Grafana Alloy: container logs grouped by project/service in Logs Drilldown
- Tempo 2.9.0: OTLP traces with seven-day local retention and TraceQL metrics

## Requirements

Docker Engine (rootful), Docker Compose v2, and access to `/var/run/docker.sock`.
This configuration targets native Linux, not Docker Desktop or rootless Docker.
Run commands with `sudo` if your user cannot access the Docker daemon.

cAdvisor uses privileged mode and host mounts following its
[upstream Docker setup](https://github.com/google/cadvisor#quick-start-running-cadvisor-in-a-docker-container).
Alloy runs as container root to access the rootful Docker socket. Keep socket
permissions restricted; do not use `chmod 666`. Socket access grants control of Docker.

## Start

```bash
sudo systemctl enable --now docker
docker info --format '{{.DockerRootDir}}'
docker compose config --quiet
docker compose up -d
docker compose ps
```

If Docker's data directory differs from `/var/lib/docker`, set `DOCKER_DATA_ROOT`
in a local `.env` file to the path reported above before starting the stack.
The path is mounted at the same location inside cAdvisor.
Compose creates the `docker-monitoring` network automatically. Containers being
monitored do not need to join this network; discovery uses the host Docker API.

## Migrating an Existing Stack

Stop the old Podman monitoring containers before starting Docker if they still
occupy ports 3000, 9090 or 3100. Docker and Podman have separate container and
volume storage: old Podman Grafana, Prometheus and Loki data is not automatically
migrated. Keep the old volumes if you need that history.

If you previously ran the old Compose configuration using Docker, recreate the
services and remove the obsolete exporter:

```bash
docker compose up -d --remove-orphans --force-recreate
```

The Compose project name remains `observability` and named volume keys are
unchanged, so existing Docker volumes for that project are reused.
The provisioned dashboard is now **Docker Overview**; historical `podman_*`
metrics are not converted into cAdvisor metrics.

## Access

| Service | URL |
|---|---|
| Grafana | http://localhost:3000 |
| Prometheus | http://localhost:9090 |
| Loki | http://localhost:3100 |
| Tempo API | http://localhost:3200 |
| OTLP gRPC / HTTP | localhost:4317 / http://localhost:4318 |

Grafana's initial credentials are `admin` / `admin` (existing volumes retain
previous credentials). Change them after first login. The published ports bind
all host interfaces by default; this stack is intended for a trusted local host.

## Verify Metrics

Prometheus's `/targets` page should show `prometheus`, `node`, and `cadvisor` as UP.
In Grafana, open **Local Performance Lab → Docker Overview**.

```promql
up{job="cadvisor"}
```

```promql
sum by(name) (rate(container_cpu_usage_seconds_total{job="cadvisor",image!="",name!=""}[1m]))
```

```promql
container_memory_working_set_bytes{job="cadvisor",image!="",name!=""}
```

The dashboard counts observed containers, shows CPU in cores, memory working set
in bytes, and network traffic in bytes/second. cAdvisor Up is scrape availability,
not Docker HEALTHCHECK status. Stopped containers are not a persistent inventory. Memory % in Container Detail
shows N/A when Docker reports no positive memory limit.

Check collection directly:

```bash
docker compose logs --tail=50 cadvisor prometheus
docker compose exec -T prometheus wget -qO- http://cadvisor:8080/metrics
```

## Verify Logs

Alloy uses Docker discovery and the Docker logs API, with persistent read offsets
in `alloy-data`. See the [Alloy Docker source documentation](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.docker/).
On migration, Loki may reject old replayed log entries outside its accepted time
window; check recent logs separately. Containers must use a logging driver readable by `docker logs` (for example,
`json-file` or `local`). Logs written only to files inside a container are not collected.

In Grafana → Explore → Loki:

```logql
{container=~".+"}
```

```bash
docker compose logs --tail=50 alloy loki
```

## Project Logs and Drilldown

Open **Grafana → Drilldown → Logs**, select **Loki**, and use **Last 15 minutes**.
Filter `project` by the Docker Compose project name, then select `service_name`.
Alloy collects stdout/stderr from all containers on this Docker daemon, including
other projects. Compose labels become `project` and `service_name`; standalone
containers use their container name as `service_name`. The `container` label is
retained. New labels apply to newly collected entries, not historical logs.

```logql
{project="observability", service_name="grafana"}
```

Loki enables pattern ingestion, structured metadata, volume queries and log-level
detection as described in the [Logs Drilldown configuration](https://grafana.com/docs/grafana/latest/visualizations/simplified-exploration/logs/access/).
Applications should emit JSON with `level`, `message`, and optionally `trace_id`.
File-only logs require a separate file collector and are not discovered automatically.

## Traces with Tempo

Apply the configuration (existing volumes are retained):

```bash
docker compose up -d tempo grafana loki
docker compose restart alloy prometheus
```

Instrument your application with an OpenTelemetry SDK or automatic instrumentation.
Tempo does not generate application traces from container logs. For an application
running on the host, configure:

```bash
OTEL_SERVICE_NAME=api
OTEL_RESOURCE_ATTRIBUTES=service.namespace=my-project
OTEL_TRACES_EXPORTER=otlp
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
```

For a container, join the external `docker-monitoring` network and use
`http://tempo:4318` (or `tempo:4317` for gRPC). Set `OTEL_SERVICE_NAME` to the
Compose service name so trace-to-log links match `service_name`. Include the
active trace ID in application logs to use trace filtering. When using a proxy,
include `tempo` and `monitor-tempo` in the application's `NO_PROXY` / `no_proxy`.
Grafana includes these exclusions through Compose.

Open **Grafana → Explore → Tempo** or **Drilldown → Traces** and query:

```traceql
{ resource.service.name = "api" }
```

Tempo enables the `local-blocks` processor for TraceQL metrics used by Traces
Drilldown. Service-graph and span-metric export to Prometheus are not configured.
Prometheus scrapes Tempo's own operational metrics at `tempo:3200/metrics`.
OTLP and Tempo API host ports bind to loopback; containers use the shared network.
The configuration follows the [Tempo 2.9 example](https://github.com/grafana/tempo/blob/v2.9.0/example/docker-compose/local/tempo.yaml).

## Troubleshooting

### Logs are ingested but Drilldown queries time out

If Loki reports `error notifying frontend` with a Squid `403 Forbidden`, its
internal gRPC traffic is being sent through the proxy inherited from `.env`.
This local-filesystem deployment sets `NO_PROXY=*` and `no_proxy=*` for Loki so
advertised Docker IPs are reached directly. Recreate Loki with
`docker compose up -d loki`; a restart alone does not apply environment changes.

Grafana also always appends internal service names to its proxy exclusions,
because shell environment variables can override `.env` during Compose expansion.
The Loki datasource sends `X-Scope-OrgID: fake`, the single-tenant storage tenant,
because Loki 3.7.8's `drilldown-limits` endpoint otherwise returns `401 no org id`
even when authentication is disabled. Check both the volume and drilldown-limits
resources through Grafana when troubleshooting the Drilldown UI.

### Docker socket permission denied

```bash
ls -ln /var/run/docker.sock
sudo docker info
docker compose logs --tail=50 alloy cadvisor
```

Ensure you are targeting the rootful daemon, the socket exists, and Alloy uses
`user: "0:0"`. There is no `keep-id` mapping or dependency on `UID` or
`XDG_RUNTIME_DIR`. Recreate services after changing their Compose configuration.

### cAdvisor is up but container metrics are missing

Check cAdvisor logs, the Docker data directory mount, and access to host cgroups.
The service uses `cgroup: host`, privileged mode, `/sys`, and `/dev/kmsg`.
This setup requires a native Linux host with those resources available.

### Port already in use

Stop the old monitoring services that own the port, or change the host-side
port in `compose.yml`. Old Podman services are not stopped by Docker Compose.

## Stop

```bash
docker compose down
```

To also delete this Docker project's monitoring data:

```bash
docker compose down -v
```

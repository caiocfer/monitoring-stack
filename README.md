# Monitoring Stack

Helm chart (`Chart.yaml`) deploying a full observability stack in namespace `monitor`. Own charts pinned in `charts/`; deployed via Argo CD.

## Components

| Chart | Short name | Workloads | Role |
|---|---|---|---|
| victoria-metrics-cluster | `vmc` | StatefulSet `vmstorage` (1), Deployment `vmselect` (2), `vminsert` (2) | Time-series storage + query/ingest. vmstorage holds 8Gi PV; vminsert is the Prometheus-compatible write endpoint, vmselect the read/query endpoint |
| victoria-metrics-agent | `vma` | Deployment | `vmagent` scrapes cluster targets (API server, nodes, cAdvisor, annotated services, ArgoCD, exporters) every 60s and remote-writes to `vminsert:8480` |
| victoria-logs-single | `vls` | StatefulSet `vls-server` (1) | Log storage/query engine. 5Gi PV, 7d retention, endpoint `:9428`. Exposed as NodePort `30228` so the host's `systemd-journal-upload` can push node/OS/systemd logs into `/insert/journald` |
| victoria-logs-collector | `vlc` | DaemonSet | Collects container logs per node and ships them to `vls-server:9428` |
| grafana | — | Deployment (1) | Dashboards & visualization. Datasource `VictoriaMetrics` (via `vmselect:8481`), 3Gi PV, exposed as NodePort `30001` |
| gpu-exporter | — | DaemonSet | Exports GPU metrics on `:9835`, scraped by vmagent |
| prometheus-node-exporter | — | DaemonSet | Node hardware/OS metrics on `:9100`, scraped by vmagent |

## Services (namespace `monitor`)

| Service | Type | Port |
|---|---|---|
| monitoring-stack-grafana | NodePort | `80:30001` |
| monitoring-stack-vls-server | NodePort | `9428:30228` |
| monitoring-stack-vmc-vminsert | ClusterIP | `8480` |
| monitoring-stack-vmc-vmselect | ClusterIP | `8481` |
| monitoring-stack-vmc-vmstorage | ClusterIP (headless) | `8482,8401,8400` |
| monitoring-stack-gpu-exporter / gpu-exporter | ClusterIP | `9835` |
| monitoring-stack-prometheus-node-exporter | ClusterIP | `9100` |

## Data flow

```
node-exporter ─┐
gpu-exporter ──┤
cAdvisor ──────┤                    ┌─ vmselect (:8481) ── Grafana (:30001)
API server ────┤--> vmagent (:8429) │
annotated svc ─┘        │           └─ vminsert (:8480) ── vmstorage (8Gi PV)
                        │
                        └─(remote write)
                        --> vmc-vminsert:8480

vlc (DaemonSet)                ─┐
  (container/pod logs)          │--> vls-server:9428 (5Gi PV, 7d retention)
host systemd-journal-upload ────┘
  (node/OS/systemd logs, :30228/insert/journald)
```

## Access

- Grafana UI: `http://<node>:30001` — datasource: VictoriaMetrics (`vmselect:8481/select/0/prometheus`)
- VictoriaLogs UI: `http://monitoring-stack-vls-server:9428` (cluster-internal)

## Storage

| PVC | Size | StorageClass | Owner |
|---|---|---|---|
| monitoring-stack-grafana | 3Gi | local-path | Grafana |
| server-volume-monitoring-stack-vls-server-0 | 5Gi | local-path | VictoriaLogs |
| vmstorage-volume-monitoring-stack-vmc-vmstorage-0 | 8Gi | local-path | VM storage |

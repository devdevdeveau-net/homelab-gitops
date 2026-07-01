# Runbook — alloy

Grafana Alloy collection agent (DaemonSet) that ships the cluster's **logs** (every
pod's stdout/stderr → Pi Loki) and **metrics** (node-exporter, kube-state-metrics,
cAdvisor → Pi Prometheus via remote_write). Chart `1.10.0` (app v1.17.0), namespace
`monitoring`, wave 1. The whole config lives inline in `apps/alloy.yaml`
(`alloy.configMap.content`); Pi endpoints come from the wave-0 `pi-endpoints`
ConfigMap via `envFrom`.

Depends on [monitoring](monitoring.md) (wave 0): the scrape targets and the
endpoints ConfigMap must exist first.

## Check health

```sh
kubectl -n monitoring get ds alloy                       # desired == ready (one per node)
kubectl -n monitoring logs ds/alloy | grep -i "remote_write\|error"
kubectl -n monitoring port-forward ds/alloy 12345:12345  # → http://127.0.0.1:12345 component graph
```
The UI's `prometheus.remote_write` / `loki.write` components show send errors and
queue backlog — the fastest way to see if the Pi is reachable.

## Common changes

- **Add a scrape job or relabel** → edit the `config.alloy` block in
  `apps/alloy.yaml`, push. (Config is in the Argo app values, not a manifests dir.)
- **Change where data ships** → edit `pi-endpoints` (see [monitoring](monitoring.md)),
  not the Alloy values.

## Troubleshooting

| Symptom | Cause |
|---|---|
| no metrics on the Pi | remote_write target down — Pi Prometheus receiver flag / `9090` port (infra-pi side) |
| no logs in Loki | `loki.write` endpoint / Loki `3100` not published on the Pi |
| Alloy CrashLoop on start | bad `config.alloy` syntax in the app values, or pi-endpoints missing (wave ordering) |
| metrics arrive without `cluster=k3s` | `external_labels` block dropped from remote_write config |

Related: [monitoring](monitoring.md)

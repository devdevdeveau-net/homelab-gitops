# Runbook — monitoring

Wave-0 scrape targets + endpoint config for the cross-Pi observability bridge.
Path `manifests/monitoring`, namespace `monitoring`. Three resources:

- **node-exporter** DaemonSet — host metrics for the k3s node (hostNetwork/hostPID),
  pinned to match the Docker Pi's exporter so metric names line up.
- **kube-state-metrics** — cluster object state.
- **pi-endpoints** ConfigMap — the **single source of truth** for where telemetry
  ships: `PROMETHEUS_URL` / `LOKI_URL` pointing at the Docker Pi (LAN IP).

[alloy](alloy.md) (wave 1) scrapes the first two and reads pi-endpoints via
`envFrom`. The data leaves the cluster stamped `cluster=k3s` (vs `cluster=pi-infra`
on the Docker Pi) so dashboards can tell the two hosts apart.

## Re-IP the Pi (most common change)

Edit **only** `manifests/monitoring/pi-endpoints.yaml` and push. Alloy restarts
pick up the new endpoints automatically.

## Verify the bridge end-to-end

```sh
kubectl -n monitoring get pods                    # node-exporter (per node) + ksm Running
# On the Docker Pi's Prometheus, confirm cluster series arriving:
#   count({cluster="k3s"})           # >0 means remote_write is landing
```

## Troubleshooting

| Symptom | Cause |
|---|---|
| no `cluster="k3s"` series on the Pi | Alloy not shipping — see [alloy](alloy.md); or the Pi side isn't ready |
| Pi rejects remote_write | Docker Pi `prometheus.yml` missing `--web.enable-remote-write-receiver` or `9090`/`3100` not published |
| ksm metrics missing | ksm pod down, or Alloy's KSM scrape job mis-targeted |

Pi-side prerequisites live in the **infra-pi** repo (compose ports + receiver flag).

Related: [alloy](alloy.md)

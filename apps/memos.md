# Runbook — memos

Lightweight self-hosted notes app. Path `manifests/memos`, namespace `memos`,
image `neosmemo/memos:0.26.0`, port 5230, data on a 1Gi `local-path` PVC at
`/var/opt/memos`, served at `https://memos.vault.REPLACE_ME_WITH_DOMAIN`.

Stateless app over a single SQLite-backed volume — the simplest app in the repo.

## Backup / DR

Everything (DB + uploads) is under `/var/opt/memos`:

```sh
kubectl -n memos exec deploy/memos -- tar czf - /var/opt/memos > memos-$(date +%F).tgz
```
Restore = scale to 0, replace the volume contents, scale back.

## Routine

```sh
kubectl -n memos get pods
kubectl -n memos logs deploy/memos
kubectl -n memos get certificate memos-cert        # TLS via letsencrypt-prod
```

## Troubleshooting

| Symptom | Check |
|---|---|
| 404/502 at the host | IngressRoute + `memos-tls` Certificate ready |
| pod Pending | `memos-data` PVC bound (`local-path`) |
| data missing after restart | confirm the PVC mounted at `/var/opt/memos` |

Upgrades: bump the image tag in `manifests/memos/memos.yaml`; Argo redeploys on push.

# Runbook — vaultwarden

Bitwarden-compatible password manager (the **human** password vault — unrelated to
HashiCorp [vault](vault.md) despite the name). Path `manifests/vaultwarden`,
namespace `vaultwarden`, image `vaultwarden/server:1.35.4`, data on a 1Gi
`local-path` PVC at `/data`, served at `https://vault.REPLACE_ME_WITH_DOMAIN`.

> ⚠️ This holds your passwords (and the Vault unseal keys / root token). Treat its
> backup as the most important in the cluster — and note the bootstrap loop:
> Vaultwarden stores Vault's keys, Vault backs ESO. Don't make Vaultwarden's own
> secrets depend on ESO.

## Backup / DR (do this regularly)

All state is the SQLite DB + attachments under `/data`:

```sh
kubectl -n vaultwarden exec deploy/vaultwarden -- tar czf - /data > vw-$(date +%F).tgz
```
Restore = scale to 0, replace `/data` in the PVC, scale back. Keep backups
off-cluster.

## Admin panel & token

`/admin`, authenticated by `ADMIN_TOKEN` from the `vaultwarden-admin` Secret —
now **ESO-managed** (Vault path `secret/vaultwarden/admin`, `Merge`). To rotate:

```sh
kubectl -n vault exec -i vault-0 -- vault kv put secret/vaultwarden/admin admin-token=<new>
kubectl -n vaultwarden rollout restart deploy/vaultwarden     # picks up the new value
```

## Notes / troubleshooting

- `SIGNUPS_ALLOWED=true` currently — flip to `false` in the manifest once your
  accounts exist, to stop open registration.
- 502/cert errors → check the `vaultwarden-tls` Certificate (`kubectl -n vaultwarden
  get certificate`) and the IngressRoute; cert-manager issues via `letsencrypt-prod`.
- Pod won't start → `describe pod`, check the `vaultwarden-data` PVC bound.

Related: [external-secrets-config](external-secrets-config.md) (admin-token migration)

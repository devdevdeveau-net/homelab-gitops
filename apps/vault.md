# Runbook — vault

HashiCorp Vault Community Edition, the secret store of record for the cluster
(ESO reads from it). Chart `0.33.0` (app v2.0.2), namespace `vault`, **standalone**
mode = `file` storage backend on a 2Gi `local-path` PVC (NVMe). UI on, Agent
injector off. No external ingress (`vault.REPLACE_ME_WITH_DOMAIN` belongs to Vaultwarden).

First-time setup is in [memory/vault-bootstrap](../memory/vault-bootstrap.md) — do
**not** re-run `operator init` on a live Vault.

## The one thing to remember: it comes up SEALED

Every pod restart (node reboot, chart upgrade, OOM) leaves Vault **sealed** — it
serves nothing and ESO syncs start failing — until manually unsealed. Auto-unseal
is a deferred decision.

```sh
kubectl -n vault exec -i vault-0 -- vault status            # Sealed: true/false
kubectl -n vault exec -it vault-0 -- vault operator unseal  # run 3x, different keys (threshold 3/5)
```

## Routine operations

```sh
# Port-forward for CLI/UI from your laptop (no ingress):
kubectl -n vault port-forward svc/vault 8200:8200
#   then http://127.0.0.1:8200/ui , or: export VAULT_ADDR=http://127.0.0.1:8200

# Write / read a secret (KV v2 at secret/):
kubectl -n vault exec -i vault-0 -- vault kv put secret/<path> key=value
kubectl -n vault exec -i vault-0 -- vault kv get secret/<path>
```

## Backup / DR

Standalone uses the **file** backend, so `raft snapshot` does **not** apply —
back up the data directory instead (best while sealed for a consistent copy):

```sh
kubectl -n vault exec vault-0 -- tar czf - /vault/data > vault-data-$(date +%F).tgz
```

Also keep the **unseal keys + root token** safe and off-cluster (they live in
Vaultwarden). Losing them = losing Vault. Restore = new pod + restore `/vault/data`
+ unseal.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| ESO secrets stop syncing, `vault status` Sealed=true | pod restarted | unseal (above) |
| ESO `permission denied` | role missing policy | check `vault read auth/kubernetes/role/eso` has `token_policies=[eso-read]` |
| `vault-0` CrashLoop | PVC / storage | `kubectl -n vault describe pod vault-0`, check the `local-path` PVC |

## Deferred ("hard decisions") — revisit later
HA/raft multi-node · auto-unseal (Azure Key Vault) · hot snapshots (needs raft) ·
external vs in-cluster placement (currently in-cluster = shares fate with k3s).

Related: [external-secrets](external-secrets.md) · [external-secrets-config](external-secrets-config.md) · [memory/vault-bootstrap](../memory/vault-bootstrap.md)

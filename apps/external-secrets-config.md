# Runbook — external-secrets-config

ESO "content" app (wave 1): the cluster-scoped `ClusterSecretStore` **vault-kv**
plus the `ExternalSecret`s for namespaces that have no manifests dir of their own
(ARC, cert-manager). Path `manifests/external-secrets-config`. Resources declare
their own namespace, so the app's `external-secrets` destination is just a default.

## Check the store is healthy

```sh
kubectl get clustersecretstore vault-kv                  # want STATUS=Valid
kubectl describe clustersecretstore vault-kv             # shows the Vault login error if Invalid
```
`Invalid` almost always means Vault is sealed, unreachable, or the `eso` role
lost its `token_policies` — see [vault](vault.md).

## Add / migrate a secret (the standard pattern)

1. **Triage** — only migrate secrets *you* hand-made. **Skip** cert-manager TLS &
   account keys, k8s SA tokens, `helm.sh/release.*`, Argo internals
   (`argocd-secret`/`-redis`/`-initial-admin`), and especially
   `argocd/homelab-gitops-repo` (Argo's git key — managing it via ESO is a
   circular bootstrap).
2. **Load Vault** (generic; preserves all keys + multiline values):
   ```sh
   kubectl -n <ns> get secret <name> -o json \
     | jq '.data | map_values(@base64d)' \
     | kubectl -n vault exec -i vault-0 -- vault kv put secret/<path> -
   ```
3. **Author the `ExternalSecret`** — `creationPolicy: Merge` (co-manage, don't
   own/clobber the existing Secret), `dataFrom.extract` for multi-key secrets
   (reproduces original key names). Put it in the workload's manifests dir if it
   has one (e.g. `manifests/vaultwarden/`), else here.
4. **Verify**:
   ```sh
   kubectl -n <ns> get externalsecret <name>             # SecretSynced / Ready=True
   ```

Currently managed here: `arc-github-app` (arc-runners), `cloudflare-api-token-secret`
(cert-manager). `vaultwarden-admin` lives in `manifests/vaultwarden/`.

## Troubleshooting

| Symptom | Cause |
|---|---|
| ES `SecretSynced=False`, "key not found" | value not in Vault, or wrong `remoteRef.key`/path |
| ES Ready but workload still broken | key *name* mismatch — use `dataFrom.extract` to keep names |
| store `Invalid` | Vault sealed / role policy — [vault](vault.md) |

Related: [vault](vault.md) · [external-secrets](external-secrets.md) · [memory/vault-bootstrap](../memory/vault-bootstrap.md)

# Runbook — external-secrets

External Secrets Operator (ESO), the **sync layer** that reads from Vault and
projects native k8s Secrets. Not a store itself. Chart `2.7.0`, namespace
`external-secrets`, `installCRDs: true`, ServerSideApply (CRD annotation-limit,
same rationale as [arc-controller](arc-controller.md)). Wave 0 — its CRDs
(`ClusterSecretStore`, `ExternalSecret`) must exist before
[external-secrets-config](external-secrets-config.md) (wave 1) and any workload
`ExternalSecret`.

## Health checks

```sh
kubectl -n external-secrets get pods                     # controller, webhook, cert-controller
kubectl get crd | grep external-secrets.io               # CRDs installed?
kubectl -n external-secrets logs deploy/external-secrets # controller logs
```

## How the pieces fit

```
Vault (store) ──read──> ESO controller ──writes──> k8s Secret ──> workload
                          ▲ binding: ClusterSecretStore `vault-kv`
                          ▲ per-secret: ExternalSecret (in the workload's ns)
```

ESO is **opt-in per secret**: it only manages a Secret that an `ExternalSecret`
names. It never touches existing Secrets otherwise. To add one, see
[external-secrets-config](external-secrets-config.md).

## Troubleshooting

| Symptom | Where to look | Likely cause |
|---|---|---|
| an `ExternalSecret` not `Ready` | `kubectl -n <ns> describe externalsecret <name>` | store invalid, or key/path missing in Vault |
| ALL ExternalSecrets failing | `kubectl get clustersecretstore vault-kv` | Vault sealed / auth broken (see [vault](vault.md)) |
| CRDs `OutOfSync` churn after upgrade | Argo app diff | re-sync; SSA is already set |

## Upgrades
Bump `targetRevision` in `external-secrets.yaml`. CRDs ship with the chart
(`installCRDs`), so a chart bump upgrades them too.

Related: [vault](vault.md) · [external-secrets-config](external-secrets-config.md)

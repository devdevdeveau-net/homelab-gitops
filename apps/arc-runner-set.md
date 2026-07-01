# Runbook — arc-runner-set

The `pi-arc` runner scale set. OCI Helm chart `gha-runner-scale-set` `0.12.1`
(lockstep with [arc-controller](arc-controller.md)), namespace `arc-runners`,
wave 1. Org `REPLACE_ME_WITH_GITHUB_ORG`, `minRunners: 0` / `maxRunners: 2`, **dind** mode
(privileged runner + Docker sidecar; no RWX volume needed). Auth via the
`arc-github-app` Secret — now **ESO-managed** (see
[external-secrets-config](external-secrets-config.md)).

## Use it

In a workflow: `runs-on: pi-arc`. With `minRunners: 0`, no runner pods exist until a
job arrives (zero idle RAM); a listener pod stays up to watch for jobs.

```sh
kubectl -n arc-runners get pods                  # 1 listener idle; ephemeral runners appear per job
kubectl -n arc-runners get autoscalingrunnerset pi-arc -o wide
```

## #1 failure mode: runners never register (403)

Symptom: no listener pod, `STATE` blank, controller logs show
`403 "Resource not accessible by integration"` on the registration-token call.
Cause: the **GitHub App** needs **Organization → Self-hosted runners: Read & write**,
*and* that permission must be **accepted** on the org installation (GitHub queues
new permissions until the owner approves). Full detail:
[memory/arc-github-app-permission](../memory/arc-github-app-permission.md).

Auth (installation token) succeeding while only the registration-token call 403s is
the tell-tale.

## Other troubleshooting

| Symptom | Cause |
|---|---|
| jobs queue, never pick up | listener down, or `runs-on` ≠ `pi-arc` |
| runner pod can't run Docker | dind sidecar issue — `describe pod`, check privileged |
| bad creds after a secret change | `arc-github-app` ExternalSecret — verify it's SecretSynced ([external-secrets-config](external-secrets-config.md)) |
| version skew errors | controller + scale set must match chart version |

## Scaling
Edit `minRunners`/`maxRunners` in `arc-runner-set.yaml` and push.

Related: [arc-controller](arc-controller.md) · [memory/arc-github-app-permission](../memory/arc-github-app-permission.md) · [external-secrets-config](external-secrets-config.md)

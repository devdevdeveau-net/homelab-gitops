# Runbook — arc-controller

Actions Runner Controller (ARC) — the controller half. OCI Helm chart
`gha-runner-scale-set-controller` `0.12.1`, namespace `arc-systems`, release `arc`
(fixes the SA name `arc-gha-rs-controller` that the scale set references). Wave 0:
installs the CRDs the [arc-runner-set](arc-runner-set.md) (wave 1) needs.

OCI pulls require the registry registered with Argo CD
(`bootstrap/arc-oci-repo.yaml`, applied manually like `root.yaml`).

## Two Argo gotchas (already handled — don't remove)

Both are documented in [memory/arc-crd-serverside-apply](../memory/arc-crd-serverside-apply.md):

1. **ServerSideApply=true** — ARC CRDs are huge (autoscalingrunnersets ~1.1MB) and
   overflow the 262144-byte client-side `last-applied-configuration` annotation.
2. **ignoreDifferences on `/spec/preserveUnknownFields`** — the API server strips
   the deprecated default the chart still sets, so without this the four CRDs sit
   permanently OutOfSync and selfHeal re-syncs every minute.

⚠️ If a sync to the **pinned** revision already failed client-side, Argo will **not**
auto-retry the same revision (selfHeal included) — do a manual sync to break out. A
version bump is a fresh revision, so it self-recovers.

## Health & upgrades

```sh
kubectl -n arc-systems get pods
kubectl -n arc-systems logs deploy/arc-gha-rs-controller
```
Controller and scale set **must** be the same chart version — bump
`arc-controller.yaml` and `arc-runner-set.yaml` together.

Related: [arc-runner-set](arc-runner-set.md) · [memory/arc-crd-serverside-apply](../memory/arc-crd-serverside-apply.md)

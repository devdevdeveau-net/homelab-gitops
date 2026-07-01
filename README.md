# homelab-gitops

GitOps configuration for a homelab Kubernetes (k3s) cluster, reconciled by
[Argo CD](https://argo-cd.readthedocs.io/) using the **app-of-apps** pattern.

There is no build step and no test suite. Argo CD on the cluster pulls this repo
and applies the manifests; changes take effect by committing and pushing. With
`syncPolicy.automated` (prune + selfHeal), the cluster continuously converges to
whatever is on `main`.

> **Sanitized template.** This repo has been de-identified for public sharing.
> Every environment-specific value (domain, GitHub org, registry, IP addresses)
> has been replaced with a `REPLACE_ME_WITH_*` placeholder. It will **not** work
> until you substitute your own values — see [Configuration](#configuration).
> No secrets are committed: all secret material is pulled at runtime from Vault
> via External Secrets Operator (see [Secrets](#secrets)).

## Architecture

```
bootstrap/root.yaml   ─► watches apps/ ─► each apps/<name>.yaml ─► manifests/<name>/
   (applied once, by hand)                (one Argo CD Application)   (the real k8s resources)
```

- **`bootstrap/root.yaml`** — the root `Application`. Applied once, manually, to
  seed the cluster. It watches `apps/`.
- **`apps/*.yaml`** — one Argo CD `Application` per workload (all in the `argocd`
  namespace). Most point at a `manifests/<name>/` path; a few instead pull a Helm
  (OCI) chart (e.g. the ARC runner stack). Ordering between apps uses
  `argocd.argoproj.io/sync-wave` annotations.
- **`manifests/<name>/`** — the actual Kubernetes resources for each app.

Adding a new app = add `manifests/<name>/<name>.yaml` **and** `apps/<name>.yaml`
pointing at it. The root app auto-discovers the new `apps/` entry, which then
deploys the manifest.

## Workloads

Each app has a detailed runbook in `apps/<name>.md`. Quick tour, by area:

### Secrets platform

- **vault** — HashiCorp Vault (Community Edition), the cluster's secret store of
  record; standalone `file` backend on a `local-path` PVC, UI on, injector off.
  Comes up **sealed** after any restart and must be manually unsealed before
  anything depending on secrets works. First-time init is a documented manual step.
- **external-secrets** — External Secrets Operator (ESO), the sync layer that reads
  from Vault and projects native Kubernetes `Secret`s. Installs its own CRDs
  (`ClusterSecretStore`, `ExternalSecret`); wave 0 so they exist before anything
  references them.
- **external-secrets-config** — the ESO "content": the cluster-scoped
  `ClusterSecretStore` (`vault-kv`) that binds ESO to Vault, plus the
  `ExternalSecret`s for namespaces that have no manifests dir of their own (ARC,
  cert-manager).

### Applications

- **vaultwarden** — Bitwarden-compatible password manager (the human password
  vault; unrelated to HashiCorp Vault). SQLite + attachments on a `local-path` PVC,
  TLS via cert-manager, admin token sourced from Vault through ESO. Holds the most
  important state in the cluster — back it up.
- **memos** — lightweight self-hosted notes app; a single SQLite-backed volume
  behind an ingress. The simplest app in the repo, and a good template for adding
  your own.
- **hello** — a trivial demo web container (built from `hello-build/`) used as a
  target for load tests and chaos experiments. Runs 3 replicas so failures degrade
  gracefully instead of causing a full outage.

### CI runners

- **arc-controller** — the controller half of GitHub's Actions Runner Controller
  (ARC), installed from an OCI Helm chart. Wave 0: installs the CRDs the runner
  scale set needs. Uses ServerSideApply because the CRDs are too large for
  client-side apply.
- **arc-runner-set** — an autoscaling set of self-hosted GitHub Actions runners
  (dind mode, `minRunners: 0` so nothing idles until a job arrives). Authenticates
  to your GitHub org via a GitHub App whose credentials come from Vault through ESO.

### Observability

- **monitoring** — wave-0 scrape targets and endpoint config: a node-exporter
  DaemonSet, kube-state-metrics, and a `pi-endpoints` ConfigMap that is the single
  source of truth for where telemetry ships (external Prometheus + Loki).
- **alloy** — Grafana Alloy DaemonSet that ships the cluster's logs and metrics to
  the external Prometheus/Loki targets defined by `monitoring`. Data is stamped with
  a `cluster` label so a central Grafana can tell hosts apart.

### Load testing

- **k6-operator** — Grafana's k6 operator; wave 0 so its `TestRun` CRD exists before
  any test is created.
- **loadtest** — a k6 `TestRun` that hammers the `hello` service and streams results
  to the external Prometheus. Intentionally **not** auto-synced (a `TestRun` is
  one-shot) — sync it manually to fire a run.

### Chaos engineering

- **chaos-mesh** — the Chaos Mesh operator + CRDs (wave 0), configured for k3s's
  containerd socket so experiments actually take effect.
- **chaos** — the chaos experiments themselves (e.g. pod-kill, network latency
  against `hello`). Intentionally **not** auto-synced so experiments only run when
  you sync, and stop when you delete them.

## Prerequisites

The cluster is assumed to already provide, out of band:

- **Argo CD** (in the `argocd` namespace) — install it **server-side**
  (`kubectl apply --server-side -f <argocd-install>`); its own CRDs are too large
  for client-side apply (see [Server-side apply](#server-side-apply--large-crds))
- **cert-manager** with a `letsencrypt-prod` `ClusterIssuer` (DNS-01 via
  Cloudflare in the original setup)
- **Traefik** with a `websecure` entrypoint
- the **`local-path`** storage class
- **Vault** + **External Secrets Operator (ESO)** for secret material
  (see [Secrets](#secrets); the `external-secrets` and `vault` apps here install
  ESO and Vault, but the one-time Vault bootstrap is manual)

## Host tuning (Raspberry Pi 5 / small nodes)

> **Required for the full stack.** The complete workload (k3s + Argo CD + all the
> controllers, ESO, ARC, chaos-mesh, monitoring, etc.) exhausts the kernel's
> default **inotify** limits on a Pi-class node. Every controller, informer, Argo
> `Application`, ESO watch, and Helm release consumes inotify *instances* and
> *watches*; the stock limits are tiny, and once they run out you get pods and
> controllers crash-looping with `too many open files` — even though the real
> shortage is inotify handles, not file descriptors.

This is **host-level OS tuning, per node** — it is *not* managed by GitOps and must
be applied on the box(es) directly.

Raise the limits via sysctl. Create `/etc/sysctl.d/99-k3s-limits.conf`:

```conf
# inotify — the usual culprit under a large controller/Argo workload
fs.inotify.max_user_instances = 8192      # default is often 128
fs.inotify.max_user_watches   = 524288    # default is often 8192
# system-wide open file handles
fs.file-max                   = 2097152
```

Apply without a reboot:

```sh
sudo sysctl --system
```

You may also need to lift the open-file **ulimit** for the k3s service itself
(file descriptors, distinct from inotify). If k3s runs under systemd:

```sh
sudo systemctl edit k3s      # add:  [Service]\n  LimitNOFILE=1048576
sudo systemctl daemon-reload && sudo systemctl restart k3s
```

### What raising these limits actually does — and what can go wrong

These numbers are **ceilings, not reservations.** Raising them consumes nothing by
itself — the kernel only uses memory as watches and file handles are actually
created. So changing the numbers is low-risk *on its own*.

The real trade-off: you are **removing a safety guardrail.** Those caps exist to
stop a single runaway or leaky process from consuming unbounded kernel resources.
With them raised:

- **Kernel memory can grow much larger before anything stops it.** Each inotify
  watch pins a small amount of *unswappable* kernel memory (~1 KB on 64-bit). At
  `max_user_watches = 524288`, a process that leaked watches could tie up on the
  order of ~0.5 GB of RAM that **cannot be paged out** — significant on an 8 GB Pi.
  The old low cap would have killed that early with `too many open files`; now it
  won't.
- **The failure mode gets worse, not better.** Instead of one pod cleanly
  crash-looping with an obvious error, you can get node-wide memory pressure, the
  kernel **OOM-killer reaping random processes**, or a sluggish/unstable node that's
  harder to diagnose.
- **It's node-wide.** This affects *every* process on the host, not just k3s.

Practical guidance:

- **Don't over-set.** The values above are sized for this workload on a Pi 5.
  Bumping them 10× "to be safe" is exactly what strips the guardrail for no benefit.
  If a node still hits the limit, raise incrementally and re-check.
- **Watch memory after applying** instead of assuming it's fine:
  ```sh
  free -h                                    # overall RAM headroom
  cat /proc/sys/fs/inotify/max_user_watches  # the ceiling you set
  # rough count of inotify watches in use across the system:
  find /proc/*/fd -lname 'anon_inode:inotify' 2>/dev/null | wc -l
  ```
- If you're not comfortable with the trade-off, the safer alternative is to **run
  less** — trim the app set (drop chaos-mesh, ARC, k6, etc.) so the stock limits
  suffice, rather than lifting the caps to fit everything on one small node.

> The values above are conservative, widely-used starting points — tune to your
> hardware; there's no single "correct" number. **If you already hand-tuned a node
> and want to recover exactly what you set**, read the live values back:
>
> ```sh
> sysctl fs.inotify.max_user_instances fs.inotify.max_user_watches fs.file-max
> grep -rHE 'inotify|file-max|file_max' /etc/sysctl.conf /etc/sysctl.d/ 2>/dev/null
> systemctl cat k3s 2>/dev/null | grep -i limit          # LimitNOFILE, if set
> cat /proc/sys/fs/inotify/max_user_watches               # currently-effective value
> ```

## Server-side apply & large CRDs

Several charts here ship CRDs too large for Kubernetes' client-side
`last-applied-configuration` annotation (262144-byte limit), so their Argo
`Application`s set `syncOptions: [ ServerSideApply=true ]`. This is **already
configured** — listed here so you know why, and so you don't remove it:

| App | Notes |
|---|---|
| `arc-controller` | ARC CRDs (~1.1MB); also needs `ignoreDifferences` on `/spec/preserveUnknownFields` or the CRDs churn OutOfSync forever |
| `external-secrets` | bundled CRDs (`installCRDs: true`) |
| `k6-operator` | the `TestRun` CRD |
| `chaos-mesh` | large chaos CRDs |

Everything else installs no large CRDs and applies fine client-side (`alloy`,
`vault`, `arc-runner-set`, and all the git-path manifest apps).

Note this also applies **one level up**: install Argo CD itself with
`kubectl apply --server-side` for the same reason (see [Prerequisites](#prerequisites)).
`bootstrap/root.yaml` is small, so client-side apply is fine there.

## Configuration

Before deploying, replace every placeholder token with your own value. Each token
appears in the manifests and is documented here:

| Placeholder | What it is | Example | Where |
|---|---|---|---|
| `REPLACE_ME_WITH_DOMAIN` | Base domain used for all ingress hostnames / TLS certs | `example.com` | `manifests/*`, `apps/vault.yaml` |
| `REPLACE_ME_WITH_GITHUB_ORG` | GitHub org/user that owns this repo (Argo `repoURL`) **and** the org the self-hosted Actions runners register to | `my-org` | `bootstrap/root.yaml`, `apps/*.yaml` |
| `REPLACE_ME_WITH_METRICS_HOST` | Host/IP of your external Prometheus + Loki (remote-write / push targets) | `10.0.0.10` | `manifests/monitoring/pi-endpoints.yaml`, `manifests/loadtest/loadtest.yaml` |
| `REPLACE_ME_WITH_REGISTRY` | Container registry (and path) the `hello` demo image is pushed to | `ghcr.io/my-org` | `manifests/hello/hello.yaml` |

### Fill them in

From the repo root, set your values and run one substitution pass:

```sh
DOMAIN="example.com"
GITHUB_ORG="my-org"
METRICS_HOST="10.0.0.10"
REGISTRY="ghcr.io/my-org"

grep -rl 'REPLACE_ME_WITH_' . --include='*.yaml' --include='*.md' \
  | xargs perl -pi -e "
      s/REPLACE_ME_WITH_DOMAIN/$DOMAIN/g;
      s/REPLACE_ME_WITH_GITHUB_ORG/$GITHUB_ORG/g;
      s/REPLACE_ME_WITH_METRICS_HOST/$METRICS_HOST/g;
      s/REPLACE_ME_WITH_REGISTRY/$REGISTRY/g;
    "

# sanity check — should print nothing:
grep -rn 'REPLACE_ME_WITH_' . --include='*.yaml' --include='*.md'
```

> Prefer to keep the placeholders in git (so the public repo stays generic) and
> inject real values only at deploy time? Layer a Kustomize overlay or a private
> branch on top instead of committing the substitutions.

## Bootstrap

1. Fill in the placeholders (above) and push to your fork's `main`.
2. Register the OCI Helm registry with Argo CD (needed for the ARC charts):
   ```sh
   kubectl apply -f bootstrap/arc-oci-repo.yaml
   ```
3. Apply the root app:
   ```sh
   kubectl apply -f bootstrap/root.yaml
   ```

Argo CD takes over from there, syncing everything under `apps/`.

## Secrets

**Nothing sensitive is committed to this repo.** Secrets are stored in Vault and
projected into Kubernetes `Secret`s at runtime by External Secrets Operator:

- `manifests/external-secrets-config/cluster-secret-store.yaml` binds ESO to the
  in-cluster Vault (`ClusterSecretStore` named `vault-kv`).
- `ExternalSecret` resources reference Vault KV paths — they contain **paths and
  key names only, never values**. Populate the referenced Vault paths yourself
  (each file's header comment shows the `vault kv put` command).

The one-time Vault bootstrap (KV v2 mount, Kubernetes auth, the `eso` role/policy)
is a manual prerequisite and is not automated here.

## Per-app manifest convention

A typical HTTP app (see `manifests/memos/`, `manifests/vaultwarden/`) bundles, in
one multi-doc YAML file:

- a `PersistentVolumeClaim` on `storageClassName: local-path`
- a `Deployment` + `Service`
- a cert-manager `Certificate` issued by the `letsencrypt-prod` `ClusterIssuer`
- a Traefik `IngressRoute` on the `websecure` entrypoint, host-routed under
  `*.REPLACE_ME_WITH_DOMAIN`, terminating TLS with the cert-manager secret

## Validating changes

There is no CI. Before pushing, sanity-check YAML locally, e.g.:

```sh
kubectl apply --dry-run=client -f manifests/<name>/<name>.yaml
```

## License

Provided as-is, for reference. Adapt freely for your own homelab.

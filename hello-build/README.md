# hello-build

Build context for the `hello` app's image. This dir is **not** under `manifests/`
so Argo CD never tries to apply it — it's just source for the container.

## Build & push (manual, do this once now)

The Pi cluster is arm64, so build for arm64. From this directory:

```sh
docker buildx build --platform linux/arm64 \
  -t REPLACE_ME_WITH_REGISTRY/hello:1.0 --push .
```

`--push` goes straight to the registry (no local load needed). The registry has
no auth, so no `docker login` — but that also means anyone who can reach
`/v2` can push, so keep it LAN/tunnel-gated.

To ship a new version: bump the tag here **and** in
`manifests/hello/hello.yaml`, rebuild, push, commit. (Argo keys off the image
tag; it won't notice a re-push to the same tag.)

## Later: build in CI on your pi-arc runners

Your ARC scale set (`pi-arc`, dind, arm64) can build this instead of your laptop.
A workflow in the app repo:

```yaml
jobs:
  build:
    runs-on: pi-arc
    steps:
      - uses: actions/checkout@v4
      - run: |
          docker build -t REPLACE_ME_WITH_REGISTRY/hello:${{ github.sha }} hello-build
          docker push REPLACE_ME_WITH_REGISTRY/hello:${{ github.sha }}
```

That closes the loop: push code → pi-arc builds arm64 → pi registry → Argo deploys.
```

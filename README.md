# harness-fde-gitops

Kubernetes manifests for [harness-fde-app](https://github.com/sharmaamanrajesh/harness-fde-app).

This repository is the **source of truth for what is deployed**. The CI/CD
pipeline never injects an image tag directly into the cluster — it opens a pull
request against this repo, and the merged state is what Harness applies.

## Layout

```
base/                    shared Deployment + Service
overlays/dev/            namespace dev,     2 replicas, NodePort 30080
overlays/staging/        namespace staging, 2 replicas, NodePort 30081
overlays/prod/           namespace prod,    3 replicas, NodePort 30082, +PDB
```

Kustomize overlays differ only in namespace, replica count, `ENVIRONMENT`, the
NodePort, and the image tag. Everything else is inherited from `base/`, so a
change to probes or security context applies to all three environments at once.

## Promotion model

The image tag in each overlay's `kustomization.yaml` is the promotion unit:

```yaml
images:
  - name: harness-demo-app
    newName: docker.io/sharmaamanrajesh/harness-demo-app
    newTag: "<commit-sha>"
```

An image is built and scanned **once**, then the same immutable tag is promoted
dev → staging → prod by bumping `newTag` in the next overlay. Nothing is
rebuilt between environments, so what passed the security gate is exactly what
reaches production.

## Rendering locally

```bash
kubectl kustomize overlays/dev            # render
kubectl apply -k overlays/dev --dry-run=server
kubectl apply -k overlays/dev             # apply
```

## Cluster bootstrap

Namespaces are managed outside this repo (cluster concern, not app concern):

```bash
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod
```

## Design notes

- **`maxUnavailable: 0`, `maxSurge: 1`** — full capacity is held throughout a
  rollout; a new pod must pass its readiness probe before an old one goes away.
- **`topologySpreadConstraints` with `ScheduleAnyway`** — spreads replicas
  across both workers. `DoNotSchedule` would deadlock the rollout, because the
  surge pod makes a 2/1 split unavoidable on a two-node cluster.
- **`preStop` sleep** — Endpoint removal and SIGTERM happen in parallel, so a
  terminating pod can still receive traffic. Measured over a rolling update:
  6 failed requests / 6409 without the hook, 1 / 8795 with it.
- **Non-root, pinned UID 1001** — `runAsNonRoot` cannot verify a username, only
  a numeric UID, so the image pins one rather than relying on the base image.
- **`readOnlyRootFilesystem: true`** — the app writes nothing at runtime.
- **PodDisruptionBudget in prod only** — environment differentiation without
  making dev/staging brittle to node maintenance.

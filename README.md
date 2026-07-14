# stockpit-config — GitOps Desired State

This repository is the **single source of truth** for what runs on the cluster. [ArgoCD](https://argo-cd.readthedocs.io/) watches it and reconciles the cluster to match — with auto-sync, prune and self-heal enabled. Nobody runs `kubectl apply` here: **a commit to this repo *is* a deployment**, and `git revert` *is* a rollback.

> 📦 **Application code, CI/CD pipeline, and full project documentation live in the main repo: [joseajrgr/stockpit](https://github.com/joseajrgr/stockpit)** — including architecture, design decisions and the incident log.

## How changes arrive here

There are only two legitimate writers:

1. **Humans**, editing manifests (replicas, resources, new components) via commits.
2. **`ci-bot`** — the CI pipeline in [stockpit](https://github.com/joseajrgr/stockpit) commits new image tags (pinned to the build's commit SHA) after tests pass and Trivy's vulnerability gate clears. Browse the history: bot deploys and human changes are cleanly distinguishable.

```
stockpit (code) ──CI──► ghcr.io images ──ci-bot commit──► THIS REPO ──ArgoCD pull──► k3s
```

## Layout

```
apps/stockpit/
├── namespace.yaml         # everything lives in the `stockpit` namespace
├── postgres.yaml          # StatefulSet + headless Service, local-path PVC
├── api.yaml               # API Deployment (probes, preStop drain, non-root) + Service + Ingress
├── servicemonitor.yaml    # Prometheus scrape config for the API
├── exporter.yaml          # Go business-metrics exporter: Deployment + Service + ServiceMonitor
└── alerts.yaml            # PrometheusRule: low-stock (warning) & API-down (critical)

terraform/                 # platform layer as code (kube-prometheus-stack, Vault)
                           # adopted via `terraform import` — apps are ArgoCD's job,
                           # platform is Terraform's job
```

## Secrets

No credentials are committed to this repo. The database Secret is created out-of-band:

```bash
kubectl -n stockpit create secret generic stockpit-db-credentials \
  --from-literal=username=app --from-literal=password=<yours>
```

A Vault instance runs in-cluster with credentials in KV v2; External Secrets Operator integration is the next planned step. (This Secret's journey — from committed anti-pattern, to out-of-band, towards Vault — is documented in this repo's own Git history.)

## Bootstrapping from zero

1. `cd terraform && terraform init && terraform apply` — platform (monitoring, Vault)
2. Install ArgoCD and create an Application: repo = this one, path = `apps/stockpit`, auto-sync + prune + self-heal
3. Create the DB Secret (above)
4. Push code to [stockpit](https://github.com/joseajrgr/stockpit) — the pipeline does the rest

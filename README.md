# genesis-cluster

Personal homelab Kubernetes cluster running on bare-metal hardware, managed with GitOps via FluxCD. Every change to infrastructure and applications is version-controlled and automatically reconciled — no manual kubectl applies in production.

## Hardware

| Component | Details |
|---|---|
| Node | Single node (control-plane) |
| OS | Ubuntu 24.04 LTS |
| Kubernetes | v1.33 (kubeadm) |
| Hostname | genesis |

## Architecture
```
Internet → Cloudflare (TLS) → Cloudflare Tunnel → Traefik → Apps
                                                       ↑
                                              FluxCD reconciles
                                              from this repo
```

Public traffic never hits the home IP directly — Cloudflare Tunnels handle all ingress without exposing any ports.

## Repository structure
```
genesis-cluster/
├── apps/                        # Application workloads
│   ├── base/                    # Environment-agnostic manifests
│   │   └── personal-website/    # enyuanzwerver.com
│   └── staging/                 # Staging overlays
├── clusters/                    # Flux entrypoints per cluster
│   └── staging/                 # Flux Kustomizations for staging
├── infrastructure/              # Platform controllers
│   └── controllers/
│       ├── base/                # Base Helm releases
│       │   ├── cloudflared/     # Cloudflare Tunnel daemon
│       │   ├── renovate/        # Automated dependency updates
│       │   ├── sealed-secrets/  # Encrypted secrets in git
│       │   └── traefik/         # Ingress controller
│       └── staging/             # Staging overlays
└── monitoring/                  # Observability stack
    └── controllers/
        ├── base/
        │   └── kube-prometheus-stack/   # Prometheus + Grafana
        └── staging/
```

## Stack

| Layer | Tool | Purpose |
|---|---|---|
| Cluster | kubeadm | Bare-metal Kubernetes |
| GitOps | FluxCD | Automated reconciliation |
| Ingress | Traefik | Reverse proxy and routing |
| Tunnel | Cloudflare Tunnels | Zero-trust public exposure |
| Secrets | Sealed Secrets | Encrypted secrets in git |
| Observability | Prometheus + Grafana | Metrics and dashboards |
| Dependency updates | Renovate | Automated Helm chart updates |

## GitOps flow
```
git push → Flux detects change → kustomize build → kubectl apply
```

Flux reconciles every hour. To force an immediate reconcile:
```bash
flux reconcile kustomization flux-system --with-source
```

## Repo conventions

- `base/` — environment-agnostic configuration, no cluster-specific values
- `staging/` — Kustomize overlays with environment-specific patches
- All secrets are encrypted with Sealed Secrets before committing — never plain text in git
- HelmRepositories live in the same namespace as their HelmRelease

## Apps

| App | Namespace | URL |
|---|---|---|
| personal-website | personal-website | [enyuanzwerver.com](https://enyuanzwerver.com) |
| Traefik dashboard | traefik | traefik.staging.local:31631 |
| Grafana | monitoring | internal only |

## Secrets management

Secrets are encrypted using [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets). To seal a new secret:
```bash
kubectl create secret generic my-secret \
  --namespace=my-namespace \
  --from-literal=key=value \
  --dry-run=client -o yaml | \
  kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=sealed-secrets \
  --cert=/tmp/sealed-secrets-cert.pem \
  --format yaml > sealed-secret.yaml
```

Always fetch the current cert before sealing:
```bash
kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=sealed-secrets > /tmp/sealed-secrets-cert.pem
```

## Useful commands
```bash
# check all Flux kustomizations
flux get kustomizations

# check all Helm releases
flux get helmreleases -A

# check a specific namespace
kubectl get all -n <namespace>

# force reconcile
flux reconcile kustomization flux-system --with-source

# check sealed secrets controller logs
kubectl logs -n sealed-secrets -l app.kubernetes.io/name=sealed-secrets
```

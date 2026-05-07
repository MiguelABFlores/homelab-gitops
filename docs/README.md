# homelab-gitops

> A production-grade, GitOps-managed Kubernetes platform running on a single Proxmox VE host.
> Live demo: **[miguelabf-devops.com](https://miguelabf-devops.com)**

---

## Overview

This repository is the single source of truth for a self-hosted Kubernetes platform built for hands-on DevOps practice. Every application, configuration change, and infrastructure component is defined declaratively in Git and continuously reconciled by ArgoCD. Manual `kubectl apply` commands are not part of the normal workflow — if it's not in Git, it doesn't exist in the cluster.

The platform runs on a single Proxmox VE hypervisor, hosting a 5-node Kubernetes cluster with real distributed storage, a private container registry, full observability, and zero-trust public exposure via Cloudflare Tunnel — the same architecture patterns used in production environments.

---

## Platform Stack

| Layer | Component | Version | Purpose |
|---|---|---|---|
| Hypervisor | Proxmox VE | 8.x | VM management, cloud-init templates |
| Kubernetes | kubeadm | v1.36.0 | Cluster bootstrap and management |
| CNI | Cilium | Latest | eBPF networking, network policies |
| Load Balancer | MetalLB | v0.15.3 | Bare-metal LoadBalancer IPs |
| Ingress | Traefik | v3.7 | HTTP/S routing, Gateway API |
| Storage | Longhorn | v1.11.2 | Distributed block storage, replication |
| GitOps | ArgoCD | v3.4 | Continuous deployment, drift detection |
| Registry | Harbor | v1.19 | Private OCI registry, Trivy scanning |
| Observability | kube-prometheus-stack | Latest | Prometheus, Grafana, Alertmanager |
| Dashboard | Headlamp | v0.41 | Kubernetes web UI |
| Public Exposure | Cloudflare Tunnel | Latest | Zero-trust ingress, TLS at edge |

---

## Repository Structure

```
homelab-gitops/
├── README.md
├── bootstrap/
│   └── root-app.yaml          # App-of-Apps root — the only manual kubectl apply
├── apps/
│   ├── whoami/                # Test app (Deployment + Service + Ingress)
│   ├── portfolio/             # Personal portfolio site
│   ├── monitoring/            # kube-prometheus-stack Helm values
│   ├── harbor/                # Harbor registry Helm values
│   ├── cloudflared/           # Cloudflare Tunnel deployment
│   └── headlamp/              # Headlamp dashboard Helm values
├── argocd/
│   └── apps/                  # ArgoCD Application manifests (managed by root-app)
│       ├── whoami.yaml
│       ├── portfolio.yaml
│       ├── monitoring.yaml
│       ├── harbor.yaml
│       ├── cloudflared.yaml
│       └── headlamp.yaml
└── docs/
    ├── architecture.md        # Full architecture reference
    ├── installation.md        # From zero to running cluster
    ├── operations.md          # Day-to-day operational runbook
    └── troubleshooting.md     # Common issues and fixes
```

---

## Infrastructure

### Host

| Spec | Value |
|---|---|
| CPU | Intel Core i5-13500H (16 threads) |
| RAM | 32 GB |
| Storage | 1 TB |
| Hypervisor | Proxmox VE |

### Cluster Nodes

| Node | Role | IP | vCPU | RAM |
|---|---|---|---|---|
| k8s-cp-01 | Control Plane | 192.168.0.202 | 2 | 4 GB |
| k8s-w-01 | Worker | 192.168.0.203 | 2 | 6 GB |
| k8s-w-02 | Worker | 192.168.0.204 | 2 | 6 GB |
| k8s-w-03 | Worker | 192.168.0.205 | 2 | 6 GB |
| k8s-w-04 | Worker | 192.168.0.206 | 2 | 6 GB |

### Network

| Resource | Value |
|---|---|
| LAN Subnet | 192.168.0.0/24 |
| MetalLB Pool | 192.168.0.240 – 192.168.0.250 |
| Traefik IP | 192.168.0.240 |
| Pod CIDR | 10.244.0.0/16 |

---

## GitOps Workflow

This platform uses the **App-of-Apps** pattern. A single root Application (`bootstrap/root-app.yaml`) watches the `argocd/apps/` directory and automatically creates, updates, or deletes Application resources when the directory contents change.

**Deploying a new application:**
```bash
# 1. Add application manifests
mkdir apps/my-app
vim apps/my-app/deployment.yaml

# 2. Add an ArgoCD Application manifest
vim argocd/apps/my-app.yaml

# 3. Push to Git — ArgoCD handles the rest
git add .
git commit -m "Add my-app"
git push
```

**Updating a running application:**
```bash
# Edit the manifest, push — ArgoCD syncs within 3 minutes (or immediately on manual refresh)
vim apps/my-app/deployment.yaml
git commit -am "Update my-app image to v2.0"
git push
```

**Drift correction:** If someone manually modifies a resource (`kubectl edit`, `kubectl scale`, etc.), ArgoCD detects the drift and reverts it within ~30 seconds due to `selfHeal: true`.

---

## Services

| Service | Internal URL | Public URL |
|---|---|---|
| Portfolio | portfolio.192.168.0.240.nip.io | [miguelabf-devops.com](https://miguelabf-devops.com) |
| ArgoCD | argocd.192.168.0.240.nip.io | — |
| Grafana | grafana.192.168.0.240.nip.io | — |
| Harbor | harbor.192.168.0.240.nip.io | — |
| Longhorn UI | longhorn.192.168.0.240.nip.io | — |
| Headlamp | headlamp.192.168.0.240.nip.io | — |
| Prometheus | prometheus.192.168.0.240.nip.io | — |

---

## Documentation

- [Architecture](docs/architecture.md) — Design decisions, component interactions, network flow
- [Installation](docs/installation.md) — Reproduce this cluster from scratch
- [Operations](docs/operations.md) — Day-to-day runbook: upgrades, scaling, backups
- [Troubleshooting](docs/troubleshooting.md) — Known issues and their fixes

---

## Roadmap

- [ ] cert-manager — Internal TLS for all services
- [ ] Sealed Secrets — Encrypted secrets in Git
- [ ] Loki — Centralized log aggregation
- [ ] Velero — Cluster backups to S3-compatible storage
- [ ] Gitea + Gitea Actions — Self-hosted Git and CI/CD
- [ ] HA Control Plane — Second control plane node with virtual IP

---

## Author

**Miguel Flores** — [www.miguelabf-devops.com](https://www.miguelabf-devops.com) · [LinkedIn](https://www.linkedin.com/in/mabrisenof/) · [GitHub](https://github.com/MiguelABFlores)

> *Built as a hands-on DevOps learning platform after transitioning from an SRE role at Oracle. Every component was deliberately chosen to reflect real production patterns.*

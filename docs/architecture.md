# Architecture

This document describes the design decisions, component interactions, and traffic flows of the homelab platform.

---

## Design Principles

**GitOps first.** Every resource in the cluster is defined in Git. The cluster state is always a function of the repository state. This enables reproducibility, audit trails, and rollback via `git revert`.

**Production patterns, homelab constraints.** Component choices reflect what you'd find in real enterprise environments (Cilium, Harbor, Longhorn, ArgoCD) rather than the simplest possible tools. The tradeoff is higher operational complexity, which is intentional for learning purposes.

**Single-node Proxmox, multi-node Kubernetes.** The hypervisor is a single physical machine, but the Kubernetes layer simulates a real multi-node cluster with actual distributed storage, pod scheduling across nodes, and node failure scenarios.

---

## Component Architecture

### Networking Stack

```
Physical NIC (vmbr0 bridge)
    │
    ├── Proxmox host (192.168.0.x)
    └── Kubernetes VMs (192.168.0.202–.206)
            │
            └── Cilium CNI (eBPF)
                    ├── Pod network: 10.244.0.0/16
                    └── Service network: 10.96.0.0/12
```

**Cilium** was chosen over Flannel/Calico for its eBPF datapath (lower overhead, richer observability), native network policy support, and because it reflects current industry adoption. It runs in standard mode alongside kube-proxy rather than replacing it, keeping the setup compatible with standard K8s assumptions.

**MetalLB** operates in Layer 2 (ARP) mode, announcing LoadBalancer IPs from the pool `192.168.0.240–250` on the LAN. This is the simplest bare-metal load balancer mode and requires no BGP router. The single IP `192.168.0.240` is statically assigned to Traefik.

**Traefik** serves as the single ingress point for all HTTP/S traffic. It reads standard `networking.k8s.io/v1 Ingress` resources and also supports the modern Gateway API (`gateway.networking.k8s.io`). All internal services use `*.192.168.0.240.nip.io` hostnames, which resolve to `192.168.0.240` via the free nip.io DNS service — no local DNS configuration required.

### Storage

```
Longhorn v1.11.2
    ├── Manager DaemonSet (all 4 workers)
    ├── Instance Manager (per worker)
    ├── CSI Driver
    └── Default StorageClass (3 replicas)
            │
            └── Volumes stored on worker node local disks
                Replicated across 3 of the 4 workers
```

**Longhorn** provides distributed block storage with `ReadWriteOnce` PVCs replicated across worker nodes. The default replica count is 3, matching the standard etcd quorum pattern. If one worker fails, all volumes remain accessible from the two surviving replicas.

**Important:** Harbor components (registry, jobservice, trivy) use `strategy: Recreate` instead of `RollingUpdate`. This is required because RWO volumes can only be mounted by one pod at a time — a RollingUpdate would create a new pod before the old one terminates, causing a `Multi-Attach` error that blocks the rollout indefinitely.

### GitOps Layer

```
GitHub (homelab-gitops repo)
    │
    └── ArgoCD (watches repo every 3 minutes)
            │
            ├── root-app (App-of-Apps)
            │       └── watches argocd/apps/
            │               ├── Creates Application: whoami
            │               ├── Creates Application: monitoring
            │               ├── Creates Application: harbor
            │               ├── Creates Application: portfolio
            │               ├── Creates Application: cloudflared
            │               └── Creates Application: headlamp
            │
            └── Each Application
                    └── Deploys resources from apps/<name>/
                        or from upstream Helm chart + local values
```

The **App-of-Apps** pattern means `bootstrap/root-app.yaml` is the only manifest ever applied with `kubectl apply` manually. After that, adding a new application is purely a Git operation: create the manifests in `apps/`, add an Application CR in `argocd/apps/`, push — ArgoCD handles the rest within 3 minutes.

ArgoCD is configured with `selfHeal: true` and `prune: true` on all Applications:
- `selfHeal` reverts manual drift (someone runs `kubectl edit`) within ~30 seconds
- `prune` deletes resources removed from Git (prevents orphaned resources)

### Container Registry

```
Harbor v1.19
    ├── Core (API server)
    ├── Portal (web UI)
    ├── Registry (OCI distribution)
    ├── JobService (async job runner)
    ├── Trivy (vulnerability scanner — auto-scans on push)
    ├── Database (PostgreSQL internal)
    └── Redis (internal)
```

Harbor stores images on a Longhorn PVC (10Gi). All worker nodes are configured to pull from Harbor over HTTP via containerd's `hosts.toml` mechanism (containerd v2.2.3+ from Docker's official `containerd.io` package required — Ubuntu's bundled `containerd` v2.2.1 has a bug in `hosts.toml` resolution).

### Observability

```
kube-prometheus-stack
    ├── Prometheus
    │       ├── Scrapes: node-exporter, kube-state-metrics, kubelet, cadvisor
    │       ├── Retention: 7 days / 10GiB
    │       └── Storage: Longhorn PVC (15Gi)
    ├── Grafana
    │       ├── ~30 pre-built dashboards
    │       └── Storage: Longhorn PVC (5Gi)
    ├── Alertmanager
    │       └── Storage: Longhorn PVC (2Gi)
    ├── node-exporter (DaemonSet — one per node)
    └── kube-state-metrics
```

Note: `kubeControllerManager`, `kubeScheduler`, `kubeProxy`, and `kubeEtcd` scraping is disabled. In a kubeadm cluster, these components bind to `127.0.0.1` by default and are not reachable by Prometheus. Enabling them requires patching the static pod manifests — a deliberate future exercise.

---

## Traffic Flow: Public Request

A request to `https://miguelabf-devops.com/about` flows as follows:

```
1. Browser resolves miguelabf-devops.com
        → Cloudflare DNS returns Cloudflare edge IP (CNAME flattening)

2. Browser connects to Cloudflare edge (Dallas PoP)
        → TLS terminated by Cloudflare (free managed cert)
        → DDoS protection, WAF applied

3. Cloudflare routes request through the tunnel
        → Outbound-only encrypted connection from cloudflared pods to Cloudflare
        → No inbound firewall rules required on the home router

4. cloudflared (2 replicas in namespace: cloudflared)
        → Receives request
        → Forwards to: http://traefik.traefik.svc.cluster.local:80
        → Passes Host header: miguelabf-devops.com

5. Traefik (192.168.0.240, namespace: traefik)
        → Matches Ingress rule: host=miguelabf-devops.com → service=portfolio
        → Forwards to: portfolio service (namespace: portfolio)

6. portfolio Service
        → ClusterIP load balancing across 2 nginx pod replicas

7. nginx pod
        → Serves static HTML/CSS/JS built from Next.js export
        → Response travels back through the same path

Total path: Browser → CF Edge → Tunnel → cloudflared → Traefik → Service → Pod
```

**Why Traefik instead of routing directly to the portfolio service?**
By routing through Traefik, the same Cloudflare Tunnel can serve multiple public services (Grafana, Harbor, etc.) without needing additional tunnels. New services are added by creating an Ingress resource — no Cloudflare reconfiguration needed.

---

## Traffic Flow: Internal Request

A request to `http://portfolio.192.168.0.240.nip.io`:

```
1. DNS: *.192.168.0.240.nip.io → 192.168.0.240 (nip.io public DNS)

2. MetalLB: 192.168.0.240 → Traefik pod (ARP announcement on LAN)

3. Traefik: matches host=portfolio.192.168.0.240.nip.io
        → Routes to portfolio Service

4. Service → Pod (same as public flow from step 6)
```

---

## Image Build and Deployment Flow

```
Developer machine
    │
    └── docker buildx build --platform linux/amd64,linux/arm64 --push
            │
            └── Harbor (harbor.192.168.0.240.nip.io/portfolio/site:v1.0.0)
                    │
                    └── Trivy auto-scans image on push
                            │
                            └── ArgoCD syncs Deployment with new image tag
                                    │
                                    └── kubelet pulls image via containerd
                                            → hosts.toml: HTTP pull from Harbor
                                            → Deployed to portfolio pods
```

**Multi-architecture builds are required** when building on Apple Silicon (arm64) for deployment to Intel nodes (amd64). The `docker buildx` toolchain with a properly configured buildkitd (`[registry."harbor..."] http = true, insecure = true`) handles this.

---

## Security Posture

| Area | Current State | Planned |
|---|---|---|
| Public TLS | Cloudflare edge cert | cert-manager internal CA |
| Internal TLS | HTTP (insecure) | cert-manager + self-signed CA |
| Registry TLS | HTTP + containerd hosts.toml | HTTPS after cert-manager |
| Secrets in Git | Plaintext passwords in values.yaml | Sealed Secrets |
| Network policies | None (Cilium installed, policies not configured) | Namespace isolation |
| RBAC | Default cluster-admin for ArgoCD | Least-privilege service accounts |

The current state is appropriate for a private homelab on a trusted LAN. None of the internal services are exposed to the internet. Production hardening (cert-manager, Sealed Secrets, network policies) is in the roadmap.

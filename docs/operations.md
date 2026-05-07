# Operations Runbook

Day-to-day procedures for operating the cluster. All commands assume you have a working `~/.kube/config` and are logged into the ArgoCD CLI.

---

## Daily Health Check

```bash
# Node status
kubectl get nodes

# Pods with issues (excludes Running and Completed)
kubectl get pods -A | grep -v -E 'Running|Completed'

# ArgoCD application status
argocd app list

# Longhorn volume health
kubectl get volumes.longhorn.io -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,STATE:.status.state,ROBUSTNESS:.status.robustness'
```

All nodes should be `Ready`. All ArgoCD apps should be `Synced` and `Healthy`. All Longhorn volumes should be `attached` or `detached` (never `faulted` or `degraded`).

---

## Deploying Changes

### Update an existing application

```bash
# Edit the manifest in Git
vim apps/portfolio/deployment.yaml

# Commit and push
git commit -am "Update portfolio to v2.0"
git push

# ArgoCD auto-syncs within 3 minutes, or force it:
argocd app sync portfolio
```

### Add a new application

See the [Installation guide](installation.md#quick-reference-add-a-new-application).

### Roll back an application

```bash
# Via ArgoCD CLI (rolls back to previous sync)
argocd app rollback portfolio

# Or via Git (the GitOps way — always prefer this)
git revert HEAD
git push
# ArgoCD will sync the reverted state
```

---

## Scaling

### Scale a deployment temporarily (reverted by ArgoCD)

```bash
# This will be reverted by ArgoCD selfHeal within ~30 seconds
kubectl scale deployment portfolio -n portfolio --replicas=5
```

### Scale permanently

```bash
# Edit the replicas in Git
vim apps/portfolio/deployment.yaml   # change replicas: 2 to replicas: 3
git commit -am "Scale portfolio to 3 replicas"
git push
```

### Add a new worker node

1. Clone the Ubuntu template in Proxmox (VMID 9000)
2. Start the VM, find the IP
3. Run the full node prep script from [Installation §3](installation.md#3-prepare-all-nodes)
4. Run Longhorn prerequisites from [Installation §3.1](installation.md#31-longhorn-prerequisites-all-nodes)
5. Configure containerd for Harbor from [Installation §5.6](installation.md#56-configure-harbor-image-pulls-on-all-nodes)
6. Generate a join command on the control plane and join:

```bash
# On control plane
kubeadm token create --print-join-command

# On new worker node
sudo kubeadm join <cp-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

7. Verify the node joined:

```bash
kubectl get nodes
kubectl get pods -n longhorn-system -o wide | grep <new-node-name>
```

---

## Upgrading Kubernetes

**Always upgrade one minor version at a time.** Never skip versions.

```bash
# 1. Upgrade kubeadm on control plane
ssh ubuntu@192.168.0.202
sudo apt update
sudo apt install -y kubeadm=1.37.0-*   # target version
kubeadm version

# 2. Plan the upgrade
sudo kubeadm upgrade plan

# 3. Apply the upgrade
sudo kubeadm upgrade apply v1.37.0

# 4. Upgrade kubelet on control plane
sudo apt install -y kubelet=1.37.0-* kubectl=1.37.0-*
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# 5. Upgrade workers one at a time
# On control plane:
kubectl drain k8s-w-01 --ignore-daemonsets --delete-emptydir-data

# On the worker node:
sudo apt update
sudo apt install -y kubeadm=1.37.0-*
sudo kubeadm upgrade node
sudo apt install -y kubelet=1.37.0-* kubectl=1.37.0-*
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# On control plane — bring worker back
kubectl uncordon k8s-w-01

# Repeat for each worker
```

---

## Upgrading a Platform Component

All Helm-based components (Harbor, monitoring, Headlamp) are upgraded by bumping the `targetRevision` in the ArgoCD Application manifest:

```bash
# Example: upgrade Harbor from 1.19.0 to 1.20.0
vim argocd/apps/harbor.yaml
# Change: targetRevision: 1.19.0
# To:     targetRevision: 1.20.0

git commit -am "Upgrade Harbor to 1.20.0"
git push
```

ArgoCD will sync the change. Monitor the rollout:

```bash
kubectl get pods -n harbor -w
argocd app get harbor
```

**Note for Harbor:** Components with RWO volumes (jobservice, registry, trivy) use `strategy: Recreate`. During an upgrade, each component will have ~10 seconds of downtime as the old pod terminates and the new one starts. This is by design.

---

## Certificate Rotation

ArgoCD generates certificates on installation. They expire after ~1 year. To rotate:

```bash
# Check current cert expiry
kubectl -n argocd get secret argocd-secret \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates
```

Rotation is handled by deleting the secret and restarting ArgoCD (it auto-generates new certs):

```bash
kubectl -n argocd delete secret argocd-secret
kubectl -n argocd rollout restart deployment argocd-server
```

---

## Node Maintenance

### Drain a node for maintenance

```bash
# Evict all pods, don't schedule new ones
kubectl drain k8s-w-02 --ignore-daemonsets --delete-emptydir-data

# Do maintenance (reboot, update OS, etc.)
ssh ubuntu@192.168.0.204 "sudo apt update && sudo apt upgrade -y && sudo reboot"

# Wait for node to come back
kubectl wait --for=condition=Ready node/k8s-w-02 --timeout=300s

# Re-enable scheduling
kubectl uncordon k8s-w-02
```

### Force-delete a stuck pod

```bash
kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0
```

Use this when a pod is stuck in `Terminating` state for more than 2-3 minutes.

### Fix a Multi-Attach volume error

Symptoms: pod stuck in `ContainerCreating` with `Multi-Attach error for volume`.

```bash
# Find which component is affected
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | grep Multi-Attach

# Force-delete ALL pods for that component (both old and new)
kubectl delete pods -n <namespace> -l component=<component> --force --grace-period=0

# Watch recovery
kubectl get pods -n <namespace> -l component=<component> -w
```

This is safe because the `Recreate` strategy ensures only one pod runs at a time.

---

## Harbor Operations

### Push an image to Harbor

```bash
# Login
docker login harbor.192.168.0.240.nip.io

# Build multi-arch (required when building on Apple Silicon for Intel cluster)
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t harbor.192.168.0.240.nip.io/<project>/<image>:<tag> \
  --push \
  .
```

### Verify an image is in Harbor

```bash
curl -s -u admin:<password> \
  http://harbor.192.168.0.240.nip.io/api/v2.0/projects/<project>/repositories/<image>/artifacts \
  | python3 -m json.tool | grep -E 'digest|size|created'
```

### View vulnerability scan results

```bash
curl -s -u admin:<password> \
  "http://harbor.192.168.0.240.nip.io/api/v2.0/projects/<project>/repositories/<image>/artifacts/<tag>/additions/vulnerabilities" \
  | python3 -m json.tool | grep -E 'severity|package|version' | head -40
```

---

## Observability

### Useful PromQL queries

```promql
# Node memory usage
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

# CPU utilization per node
1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))

# Pods per namespace
count by (namespace) (kube_pod_info)

# Top 5 pods by CPU
topk(5, sum by (pod) (rate(container_cpu_usage_seconds_total{container!=""}[5m])))

# Pods that restarted in the last hour
sum by (pod, namespace) (changes(kube_pod_container_status_restarts_total[1h])) > 0

# PVC usage
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes * 100
```

### Recommended Grafana dashboards to import

| ID | Name | Use |
|---|---|---|
| 1860 | Node Exporter Full | Per-node CPU/RAM/disk/network |
| 15760 | Kubernetes Views / Pods | Pod-level resource view |
| 15757 | Kubernetes Views / Global | Cluster overview |
| 15758 | Kubernetes Views / Namespaces | Namespace breakdown |

Import via: Grafana → Dashboards → New → Import → enter ID.

---

## Backup and Restore

> Velero is not yet installed. Until it is, the cluster is not backed up. This is a known gap in the roadmap.

**Manual etcd backup** (run on control plane):

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db

ls -lh /tmp/etcd-snapshot-*.db
```

Copy the snapshot off the cluster:

```bash
scp ubuntu@192.168.0.202:/tmp/etcd-snapshot-*.db ~/backups/
```

---

## Useful Aliases

Add to `~/.zshrc` or `~/.bashrc`:

```bash
alias k=kubectl
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A'
alias kgn='kubectl get nodes -o wide'
alias kga='kubectl get all'
alias kdp='kubectl describe pod'
alias kl='kubectl logs'
alias klf='kubectl logs -f'
alias kargocd='argocd --grpc-web'

# Context switching
alias kns='kubectl config set-context --current --namespace'

# Handy functions
kexec() { kubectl exec -it "$1" -- /bin/sh; }
klogs() { kubectl logs -f "$1" --tail=100; }
```

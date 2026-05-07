# Troubleshooting

Known issues, their root causes, and verified fixes. Each entry describes what was observed, why it happened, and how to resolve it.

---

## containerd: TLS error pulling from Harbor

**Symptom:**

```
failed to pull image "harbor.192.168.0.240.nip.io/...":
tls: failed to verify certificate: x509: certificate is valid for
<traefik-internal-cert>, not harbor.192.168.0.240.nip.io
```

**Root cause:**

containerd is attempting HTTPS for the Harbor registry, falling through to Traefik's self-signed default certificate. Two possible sub-causes:

1. The `hosts.toml` file is missing or not being read
2. The node is running `containerd` v2.2.1 from Ubuntu's package repos, which has a bug in `hosts.toml` resolution

**Fix:**

First, check the containerd version on the affected node:

```bash
ssh ubuntu@<node-ip> "containerd --version"
```

If the version is from Ubuntu's package (`containerd github.com/containerd/containerd/v2 2.2.1`), upgrade to the Docker official package:

```bash
ssh ubuntu@<node-ip> 'bash -s' <<'EOF'
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update && sudo apt install -y containerd.io
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo sed -i "s|config_path = ''|config_path = '/etc/containerd/certs.d'|g" /etc/containerd/config.toml
sudo mkdir -p /etc/containerd/certs.d/harbor.192.168.0.240.nip.io
sudo tee /etc/containerd/certs.d/harbor.192.168.0.240.nip.io/hosts.toml <<'TOML'
server = "http://harbor.192.168.0.240.nip.io"
[host."http://harbor.192.168.0.240.nip.io"]
  capabilities = ["pull", "resolve"]
  skip_verify = true
TOML
sudo systemctl restart containerd
EOF
```

Verify:

```bash
ssh ubuntu@<node-ip> "sudo crictl pull harbor.192.168.0.240.nip.io/portfolio/site:v1.0.0 2>&1 | tail -1"
# Expected: Image is up to date for sha256:...
```

**Prevention:** Always install `containerd.io` from Docker's official repo, not Ubuntu's bundled `containerd`.

---

## Harbor: Multi-Attach error / pod stuck in ContainerCreating

**Symptom:**

```
Multi-Attach error for volume "pvc-xxx": Volume is already used by pod(s) harbor-registry-OLD
```

Pod stays in `ContainerCreating` for hours while an old pod is in `Terminating`.

**Root cause:**

Harbor components with RWO (ReadWriteOnce) PVCs (registry, jobservice, trivy) use `RollingUpdate` strategy by default. This creates a new pod before the old one terminates, but RWO volumes can only be mounted by one pod at a time — the new pod can never start.

**Immediate fix:**

```bash
# Force-delete all pods for the affected component
kubectl delete pods -n harbor -l component=<registry|jobservice|trivy> --force --grace-period=0

# Watch recovery
kubectl get pods -n harbor -l component=<component> -w
```

**Permanent fix:**

Ensure `strategy: Recreate` is set for all Harbor components with PVCs in `apps/harbor/values.yaml`:

```yaml
jobservice:
  strategy:
    type: Recreate

registry:
  strategy:
    type: Recreate

trivy:
  strategy:
    type: Recreate
```

Push to Git and force an ArgoCD sync.

---

## ArgoCD: ApplicationSet CrashLoopBackOff

**Symptom:**

```
kubectl get pods -n argocd | grep applicationset
argocd-applicationset-controller   0/1   CrashLoopBackOff
```

Logs show:

```
error: failed to get restmapping: no matches for kind "ApplicationSet" in version "argoproj.io/v1alpha1"
```

**Root cause:**

The `applicationsets.argoproj.io` CRD is missing from the cluster. This happens when ArgoCD is installed with a manifest that omits the CRD, or when the CRD was accidentally pruned.

**Fix:**

```bash
# Apply the CRD using server-side apply (required — CRD is too large for client-side)
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/v3.4.1/manifests/crds/applicationset-crd.yaml

# Restart the controller
kubectl -n argocd rollout restart deployment argocd-applicationset-controller

# Verify
kubectl get crd applicationsets.argoproj.io
kubectl logs -n argocd deploy/argocd-applicationset-controller --tail=5
```

Logs should show `Starting workers` without any `failed to get restmapping` errors.

---

## ArgoCD: ComparisonError — repository not found

**Symptom:**

```
Failed to load target state: failed to generate manifest for source 1 of 2:
rpc error: code = Unknown desc = failed to list refs: repository not found
```

**Root causes:**

1. Typo in the `repoURL` field of the Application manifest (e.g., `MiguelABFlores>` instead of `MiguelABFlores`)
2. The GitHub PAT has expired or has insufficient permissions
3. The repo was made private after the credential was added

**Fix:**

Check the Application manifest in Git:

```bash
curl -s https://raw.githubusercontent.com/MiguelABFlores/homelab-gitops/main/argocd/apps/<app>.yaml \
  | grep repoURL
```

If the URL is correct, verify the repo credential:

```bash
argocd repo list
# Check STATUS column — should be "Successful"
```

If the credential is stale, re-add it:

```bash
argocd repo rm https://github.com/MiguelABFlores/homelab-gitops.git
argocd repo add https://github.com/MiguelABFlores/homelab-gitops.git \
  --username MiguelABFlores \
  --password <new-pat>
```

Patch the Application if the URL had a typo:

```bash
kubectl patch application <app-name> -n argocd --type=json \
  -p='[{"op": "replace", "path": "/spec/source/repoURL", "value": "https://github.com/MiguelABFlores/homelab-gitops.git"}]'
```

---

## kubectl: connection refused / localhost:8080

**Symptom:**

```
The connection to the server localhost:8080 was refused
```

**Root causes:**

1. Running kubectl without a kubeconfig (e.g., in a fresh shell or as root)
2. The KUBECONFIG environment variable is not set
3. `~/.kube/config` points to the wrong server IP

**Fix:**

```bash
# Check current kubeconfig
kubectl config view | grep server

# If missing or wrong, re-copy from control plane
scp ubuntu@192.168.0.202:/home/ubuntu/.kube/config ~/.kube/config

# Verify it points to the correct IP
grep server ~/.kube/config
# Should show: server: https://192.168.0.202:6443

# Test
kubectl get nodes
```

---

## Headlamp: "You don't have permissions to view this resource"

**Symptom:**

Headlamp loads but shows permission errors when browsing resources.

**Root cause:**

The token used to authenticate doesn't have sufficient RBAC permissions. This usually means the ServiceAccount used for the token doesn't have a ClusterRoleBinding with `cluster-admin`.

**Fix:**

```bash
# Find the ServiceAccount that the Headlamp pod uses
SA=$(kubectl get pod -n headlamp -l app.kubernetes.io/name=headlamp \
  -o jsonpath='{.items[0].spec.serviceAccountName}')
echo "ServiceAccount: $SA"

# Check if it has a ClusterRoleBinding
kubectl get clusterrolebinding | grep headlamp

# Create or fix the ClusterRoleBinding
kubectl create clusterrolebinding headlamp-cluster-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=headlamp:$SA \
  --dry-run=client -o yaml | kubectl apply -f -

# Generate a new token
kubectl create token $SA -n headlamp --duration=8760h
```

Paste the new token in the Headlamp login page.

---

## Longhorn: Volume stuck in Attaching state

**Symptom:**

A pod is stuck in `ContainerCreating` and Longhorn shows the volume as `attaching` for more than 5 minutes.

**Root cause:**

Usually happens after a node restart or network interruption. Longhorn's volume controller is waiting for the previous attachment to be fully cleaned up.

**Fix:**

```bash
# Find the stuck volume
kubectl get volumes.longhorn.io -n longhorn-system \
  -o custom-columns='NAME:.metadata.name,STATE:.status.state' | grep -v detach | grep -v attach

# Force-detach the volume (safe — Longhorn will re-attach to the correct node)
kubectl patch volumes.longhorn.io <volume-name> -n longhorn-system \
  --type=merge -p '{"spec":{"nodeID":""}}'

# Alternatively, delete and recreate the stuck pod
kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0
```

If the issue persists, restart the Longhorn instance manager on the affected node:

```bash
kubectl delete pod -n longhorn-system \
  $(kubectl get pods -n longhorn-system -o wide | grep instance-manager | grep <node-name> | awk '{print $1}')
```

---

## Node: NotReady after maintenance

**Symptom:**

A node shows `NotReady` after a reboot or maintenance.

**Diagnosis:**

```bash
kubectl describe node <node-name> | grep -A 20 "Conditions:"
ssh ubuntu@<node-ip> "sudo systemctl status kubelet | tail -20"
ssh ubuntu@<node-ip> "sudo journalctl -u kubelet -n 50 --no-pager"
```

**Common fixes:**

containerd not running:

```bash
ssh ubuntu@<node-ip> "sudo systemctl restart containerd && sudo systemctl restart kubelet"
```

Swap re-enabled after reboot (check if `/etc/fstab` has a swap entry that wasn't commented out):

```bash
ssh ubuntu@<node-ip> "sudo swapoff -a && swapon --show"
```

Kernel modules not loaded after reboot:

```bash
ssh ubuntu@<node-ip> "sudo modprobe overlay && sudo modprobe br_netfilter && sudo systemctl restart kubelet"
```

---

## Cloudflare Tunnel: 404 from miguelabf-devops.com

**Symptom:**

`curl -I https://miguelabf-devops.com` returns `HTTP/2 404` with `server: cloudflare`.

**Root cause:**

Traefik has no Ingress rule matching `miguelabf-devops.com`. The tunnel is working (Cloudflare responds) but the public hostnames are not configured in the Ingress, or the portfolio app isn't running.

**Diagnosis:**

```bash
# Check if the Ingress has the public hostname
kubectl get ingress -n portfolio -o yaml | grep -A3 "rules:"

# Check if cloudflared pods are running and connected
kubectl get pods -n cloudflared
kubectl logs -n cloudflared -l app=cloudflared --tail=20 | grep -i "connection\|error"

# Test routing from inside the cluster
kubectl run curltest --image=curlimages/curl --rm -it --restart=Never -- \
  curl -sI -H "Host: miguelabf-devops.com" http://traefik.traefik.svc.cluster.local
```

**Fix:**

If the Ingress is missing the public hostname, update `apps/portfolio/ingress.yaml` to include `miguelabf-devops.com` as a host and push to Git.

If cloudflared pods aren't connecting, check the token Secret exists:

```bash
kubectl get secret cloudflared-token -n cloudflared
```

If missing, recreate it (token is in the Cloudflare dashboard under Zero Trust → Networks → Tunnels):

```bash
kubectl -n cloudflared create secret generic cloudflared-token \
  --from-literal=token='<your-tunnel-token>'
kubectl rollout restart deployment -n cloudflared cloudflared
```

---

## kubectl apply: "Too long: may not be more than 262144 bytes"

**Symptom:**

```
The CustomResourceDefinition "..." is invalid: metadata.annotations: Too long: may not be more than 262144 bytes
```

**Root cause:**

`kubectl apply` stores the full resource in an annotation for change tracking (`last-applied-configuration`). Large CRDs like ArgoCD's ApplicationSet exceed the 256KB annotation limit.

**Fix:**

Use server-side apply instead:

```bash
kubectl apply --server-side -f <resource.yaml>
```

Server-side apply tracks ownership on the server rather than in an annotation, bypassing the size limit entirely.

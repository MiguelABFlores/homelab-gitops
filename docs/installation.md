# Installation

This guide reproduces the full platform from scratch. Follow the sections in order.

**Time estimate:** 3–4 hours for a first-time setup. Subsequent rebuilds take ~45 minutes with the scripts here.

---

## Prerequisites

- Proxmox VE 8.x installed on the host machine
- SSH access to the Proxmox host as root
- A workstation with: `kubectl`, `helm`, `argocd` CLI, `git`, `docker` (with buildx)
- A GitHub account with a fork/clone of this repository
- A Cloudflare account (free tier) for public exposure

---

## 1. Proxmox Setup

### 1.1 Fix package sources (remove enterprise repo warning)

On the Proxmox host:

```bash
# Replace enterprise repo with no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-enterprise.list
apt update && apt upgrade -y
```

### 1.2 Build the Ubuntu cloud-init template

```bash
cd /var/lib/vz/template/iso

# Download Ubuntu 24.04 cloud image
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# Install tools and customize the image
apt install -y libguestfs-tools
virt-customize -a noble-server-cloudimg-amd64.img --install qemu-guest-agent
virt-customize -a noble-server-cloudimg-amd64.img --run-command 'systemctl enable qemu-guest-agent'
virt-customize -a noble-server-cloudimg-amd64.img --truncate /etc/machine-id

# Create the template VM (VMID 9000 by convention)
qm create 9000 \
  --name ubuntu-2404-template \
  --memory 2048 \
  --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --ostype l26

qm set 9000 --scsihw virtio-scsi-pci \
  --scsi0 local-lvm:0,import-from=/var/lib/vz/template/iso/noble-server-cloudimg-amd64.img
qm set 9000 --ide2 local-lvm:cloudinit
qm set 9000 --boot c --bootdisk scsi0
qm set 9000 --serial0 socket --vga serial0
qm set 9000 --agent enabled=1

# Set cloud-init defaults (replace with your SSH public key)
mkdir -p ~/.ssh
cat > ~/.ssh/authorized_keys   # paste your public key, Ctrl+D
qm set 9000 --ciuser ubuntu
qm set 9000 --sshkeys ~/.ssh/authorized_keys
qm set 9000 --ipconfig0 ip=dhcp

# Resize disk to 32GB
qm resize 9000 scsi0 +30G

# Convert to template
qm template 9000
```

---

## 2. Provision Cluster VMs

### 2.1 Clone nodes from the template

```bash
# Control plane
qm clone 9000 201 --name k8s-cp-01 --full
qm set 201 --memory 4096 --cores 2
qm set 201 --ipconfig0 ip=dhcp
qm start 201

# Workers
for vmid in 202 203 204 205; do
  qm clone 9000 $vmid --name k8s-w-0$((vmid-201)) --full
  qm set $vmid --memory 6144 --cores 2
  qm set $vmid --ipconfig0 ip=dhcp
  qm start $vmid
done
```

Wait ~45 seconds for cloud-init, then find the IPs:

```bash
qm guest cmd 201 network-get-interfaces
# repeat for 202-205
```

Set DHCP reservations in your router for all 5 nodes. Update your `~/.ssh/config` with the actual IPs.

### 2.2 Set hostnames

```bash
ssh ubuntu@<cp-ip>   "sudo hostnamectl set-hostname k8s-cp-01"
ssh ubuntu@<w01-ip>  "sudo hostnamectl set-hostname k8s-w-01"
ssh ubuntu@<w02-ip>  "sudo hostnamectl set-hostname k8s-w-02"
ssh ubuntu@<w03-ip>  "sudo hostnamectl set-hostname k8s-w-03"
ssh ubuntu@<w04-ip>  "sudo hostnamectl set-hostname k8s-w-04"
```

---

## 3. Prepare All Nodes

Run this script on **all 5 nodes** (control plane and all workers):

```bash
# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# Install containerd.io from Docker's official repo (NOT the Ubuntu package)
# IMPORTANT: Ubuntu's bundled 'containerd' v2.2.1 has a hosts.toml bug.
# Always use 'containerd.io' from Docker repos (v2.2.3+).
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y containerd.io

# Configure containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo sed -i "s|config_path = ''|config_path = '/etc/containerd/certs.d'|g" /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Kubernetes apt repo (v1.36)
sudo apt install -y apt-transport-https gpg
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

### 3.1 Longhorn prerequisites (all nodes)

```bash
sudo apt install -y open-iscsi nfs-common cryptsetup dmsetup
sudo systemctl enable --now iscsid
sudo modprobe iscsi_tcp
echo 'iscsi_tcp' | sudo tee /etc/modules-load.d/iscsi_tcp.conf
sudo systemctl stop multipathd.socket multipathd
sudo systemctl disable multipathd.socket multipathd
```

---

## 4. Bootstrap the Cluster

### 4.1 Initialize the control plane

On **k8s-cp-01** only:

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<cp-01-ip> \
  --node-name=k8s-cp-01
```

Save the `kubeadm join` command from the output.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 4.2 Install Cilium CNI

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --fail --remote-name-all \
  https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz

cilium install
cilium status --wait
```

### 4.3 Join workers

On each worker node:

```bash
sudo kubeadm join <cp-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

If the token expired, regenerate on cp-01: `kubeadm token create --print-join-command`

### 4.4 Verify cluster

```bash
kubectl get nodes -o wide
# All nodes should be Ready
```

### 4.5 Copy kubeconfig to workstation

```bash
scp ubuntu@<cp-ip>:/home/ubuntu/.kube/config ~/.kube/config
sed -i 's|server: https://.*|server: https://<cp-ip>:6443|' ~/.kube/config
kubectl get nodes
```

---

## 5. Install Platform Components

Install Helm on the control plane (also useful on workstation):

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 5.1 MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml

kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=120s

cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.240-192.168.0.250   # adjust to your LAN
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - homelab-pool
EOF
```

### 5.2 Traefik

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
kubectl create namespace traefik

# See apps/traefik/values.yaml for full values
helm install traefik traefik/traefik \
  --namespace traefik \
  --values apps/traefik/values.yaml   # from this repo

# Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/standard-install.yaml
```

### 5.3 Longhorn

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.11.2/deploy/longhorn.yaml

# Wait for all pods to be Running (~5 minutes)
kubectl -n longhorn-system get pods -w

# Set 3 replicas (matches our 4-worker setup)
kubectl patch storageclass longhorn -p '{"parameters":{"numberOfReplicas":"3"}}'
kubectl -n longhorn-system patch setting default-replica-count --type=merge -p '{"value":"3"}'
```

### 5.4 ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Apply the ApplicationSet CRD (large CRD requires server-side apply)
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/v3.4.1/manifests/crds/applicationset-crd.yaml

# Disable internal TLS so Traefik can route to it
kubectl -n argocd patch configmap argocd-cmd-params-cm \
  --type merge -p '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deployment argocd-server
```

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo
```

Create the Ingress:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd
  namespace: argocd
spec:
  ingressClassName: traefik
  rules:
  - host: argocd.192.168.0.240.nip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 80
EOF
```

Connect ArgoCD to this repository:

```bash
argocd login argocd.192.168.0.240.nip.io --insecure --grpc-web --username admin

argocd repo add https://github.com/MiguelABFlores/homelab-gitops.git \
  --username MiguelABFlores \
  --password <github-pat>
```

### 5.5 Deploy everything via App-of-Apps

This is the last manual `kubectl apply`:

```bash
kubectl apply -f https://raw.githubusercontent.com/MiguelABFlores/homelab-gitops/main/bootstrap/root-app.yaml
```

ArgoCD will detect all Application manifests in `argocd/apps/` and deploy them automatically. Monitor progress:

```bash
argocd app list
watch kubectl get pods -A
```

### 5.6 Configure Harbor image pulls on all nodes

After Harbor is running, configure containerd on each node to pull over HTTP:

```bash
for ip in 192.168.0.202 192.168.0.203 192.168.0.204 192.168.0.205 192.168.0.206; do
  ssh ubuntu@$ip 'bash -s' <<'EOF'
sudo mkdir -p /etc/containerd/certs.d/harbor.192.168.0.240.nip.io
sudo tee /etc/containerd/certs.d/harbor.192.168.0.240.nip.io/hosts.toml > /dev/null <<'TOML'
server = "http://harbor.192.168.0.240.nip.io"

[host."http://harbor.192.168.0.240.nip.io"]
  capabilities = ["pull", "resolve"]
  skip_verify = true
TOML
sudo systemctl restart containerd
EOF
done
```

---

## 6. Cloudflare Tunnel Setup

### 6.1 Create the tunnel

1. Go to [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) → Networks → Tunnels
2. Create a tunnel named `homelab`
3. Copy the tunnel token

### 6.2 Create the token Secret in the cluster

```bash
kubectl create namespace cloudflared
kubectl -n cloudflared create secret generic cloudflared-token \
  --from-literal=token='<your-tunnel-token>'
```

The cloudflared Deployment is managed by ArgoCD from `apps/cloudflared/`. It will pick up the Secret automatically once it's synced.

### 6.3 Configure public hostnames in Cloudflare

In the tunnel configuration → Public Hostnames:

| Hostname | Service |
|---|---|
| `miguelabf-devops.com` | `http://traefik.traefik.svc.cluster.local:80` |
| `www.miguelabf-devops.com` | `http://traefik.traefik.svc.cluster.local:80` |

### 6.4 Add DNS records in Cloudflare

In Cloudflare DNS → Add record:

- Type: `CNAME`, Name: `@`, Target: `<tunnel-id>.cfargotunnel.com`, Proxied: ✅
- Type: `CNAME`, Name: `www`, Target: `<tunnel-id>.cfargotunnel.com`, Proxied: ✅

---

## 7. Verify the Full Stack

```bash
# All nodes Ready
kubectl get nodes

# All pods Running
kubectl get pods -A | grep -v Running | grep -v Completed

# All ArgoCD apps Synced/Healthy
argocd app list

# Public site responding
curl -I https://miguelabf-devops.com
# Expected: HTTP/2 200, server: cloudflare

# Internal services
curl -s http://portfolio.192.168.0.240.nip.io | head -5
curl -I http://argocd.192.168.0.240.nip.io
```

---

## Quick Reference: Add a New Application

```bash
# 1. Create application manifests
mkdir apps/my-app
cat > apps/my-app/deployment.yaml << 'EOF'
# ... your Deployment YAML
EOF

# 2. Create the ArgoCD Application
cat > argocd/apps/my-app.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/MiguelABFlores/homelab-gitops.git
    targetRevision: main
    path: apps/my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

# 3. Push — ArgoCD handles the rest
git add apps/my-app argocd/apps/my-app.yaml
git commit -m "Add my-app"
git push
```

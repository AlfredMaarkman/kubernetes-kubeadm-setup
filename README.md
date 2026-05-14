# ☸️ Kubernetes Cluster Setup on Ubuntu (kubeadm)

> A complete, production-grade guide to installing and configuring a Kubernetes cluster on Ubuntu Linux using **kubeadm**.

---
###Option 1:  You can use the folder and execute it per order for installation.
---

###Option 2: you can follow this process down ⬇️
---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation Steps](#installation-steps)
  - [1. Disable Swap](#1-disable-swap)
  - [2. Set Hostnames](#2-set-hostnames)
  - [3. Kernel Modules & Sysctl](#3-kernel-modules--sysctl-settings)
  - [4. Install Container Runtime](#4-install-container-runtime-containerd)
  - [5. Install kubeadm, kubelet, kubectl](#5-install-kubeadm-kubelet-kubectl)
  - [6. Initialize Control Plane](#6-initialize-the-control-plane)
  - [7. Install CNI Network Plugin](#7-install-a-cni-network-plugin)
  - [8. Join Worker Nodes](#8-join-worker-nodes)
  - [9. Verify the Cluster](#9-verify-the-cluster)
- [Quick Reference](#quick-reference)
- [Uninstalling Minikube](#uninstalling-minikube)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

This repository documents the step-by-step process of setting up a **production-grade Kubernetes cluster** on Ubuntu using `kubeadm`. It covers everything from system preparation to a fully operational multi-node cluster.

| Tool | Purpose |
|------|---------|
| `kubeadm` | Bootstrap the Kubernetes cluster |
| `kubelet` | Node agent that runs on every node |
| `kubectl` | CLI to interact with the cluster |
| `containerd` | Container runtime |
| `Calico / Flannel` | CNI network plugin |

---

## Architecture

```
┌─────────────────────────────────────┐
│          Control Plane Node         │
│  ┌─────────┐  ┌────────────────┐   │
│  │  API    │  │   etcd         │   │
│  │ Server  │  │  (state store) │   │
│  └─────────┘  └────────────────┘   │
│  ┌──────────────┐  ┌────────────┐  │
│  │  Scheduler   │  │ Controller │  │
│  │              │  │  Manager   │  │
│  └──────────────┘  └────────────┘  │
└─────────────────────────────────────┘
            │           │
     ┌──────┘           └──────┐
     ▼                         ▼
┌──────────────┐       ┌──────────────┐
│ Worker Node 1│       │ Worker Node 2│
│  ┌────────┐  │       │  ┌────────┐  │
│  │kubelet │  │       │  │kubelet │  │
│  └────────┘  │       │  └────────┘  │
│  ┌────────┐  │       │  ┌────────┐  │
│  │  Pods  │  │       │  │  Pods  │  │
│  └────────┘  │       │  └────────┘  │
└──────────────┘       └──────────────┘
```

---

## Prerequisites

Before you begin, ensure each node meets the following requirements:

- ✅ **OS**: Ubuntu 20.04 / 22.04 / 24.04 LTS
- ✅ **CPU**: Minimum **2 vCPUs** per node
- ✅ **RAM**: Minimum **2 GB** per node
- ✅ **Disk**: At least **20 GB** free space
- ✅ **Network**: Full network connectivity between all nodes
- ✅ **Unique** hostname, MAC address, and product UUID per node
- ✅ **Root/sudo** access on all nodes
- ✅ **Swap** disabled (required by Kubernetes)

---

## Installation Steps

### 1. Disable Swap

Kubernetes requires swap to be disabled:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

> ⚠️ The `sed` command comments out the swap entry in `/etc/fstab` to persist across reboots.

---

### 2. Set Hostnames

Run on each respective node:

```bash
# On control plane node
sudo hostnamectl set-hostname "k8s-master"

# On worker node 1
sudo hostnamectl set-hostname "k8s-worker1"

# On worker node 2
sudo hostnamectl set-hostname "k8s-worker2"
```

Optionally update `/etc/hosts` on each node:

```bash
sudo nano /etc/hosts
# Add entries like:
# 192.168.1.10  k8s-master
# 192.168.1.11  k8s-worker1
# 192.168.1.12  k8s-worker2
```

---

### 3. Kernel Modules & Sysctl Settings

Run on **all nodes**:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

---

### 4. Install Container Runtime (containerd)

Run on **all nodes**:

```bash
sudo apt update
sudo apt install -y containerd

# Generate default config
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup (required)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

### 5. Install kubeadm, kubelet, kubectl

Run on **all nodes**:

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg

# Add Kubernetes apt repository (v1.32)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable kubelet
```

> 💡 `apt-mark hold` prevents unintended upgrades that could break cluster compatibility.

---

### 6. Initialize the Control Plane

Run **only on the master node**:

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

> Use `--pod-network-cidr=10.244.0.0/16` if you plan to use **Flannel** instead of Calico.

After initialization, set up `kubectl` access:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

### 7. Install a CNI Network Plugin

Run **on the master node** after initialization:

**Option A — Calico (Recommended):**
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

**Option B — Flannel:**
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Wait for all pods to become ready:
```bash
kubectl get pods -n kube-system --watch
```

---

### 8. Join Worker Nodes

After `kubeadm init` completes, a join command is printed. Run it on **each worker node**:

```bash
sudo kubeadm join <master-ip>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

If the token has expired, regenerate it on the master:

```bash
kubeadm token create --print-join-command
```

---

### 9. Verify the Cluster

Run on the **master node**:

```bash
# Check all nodes are Ready
kubectl get nodes

# Check all system pods are Running
kubectl get pods -A
```

Expected output:
```
NAME           STATUS   ROLES           AGE   VERSION
k8s-master     Ready    control-plane   10m   v1.32.x
k8s-worker1    Ready    <none>          5m    v1.32.x
k8s-worker2    Ready    <none>          5m    v1.32.x
```

🎉 **Your cluster is up and running!**

---

## Quick Reference

| Command | Description |
|--------|-------------|
| `kubectl get nodes` | List all nodes and their status |
| `kubectl get pods -A` | List all pods across all namespaces |
| `kubectl describe node <name>` | Detailed info about a node |
| `kubectl drain <node>` | Safely evict all pods from a node |
| `kubectl cordon <node>` | Mark node as unschedulable |
| `kubectl uncordon <node>` | Re-enable scheduling on a node |
| `kubeadm token create --print-join-command` | Generate new worker join command |
| `kubeadm reset` | Tear down the cluster on a node |

---

## Uninstalling Minikube

If you previously had Minikube installed, here's how to fully remove it:

```bash
# Stop and delete cluster
minikube stop
minikube delete --all --purge

# Remove binary
sudo rm -f /usr/local/bin/minikube

# Delete config and data
rm -rf ~/.minikube
rm -rf ~/.kube   # Only if not used by other clusters

# If installed via apt
sudo apt remove minikube -y && sudo apt autoremove -y

# If installed via snap
sudo snap remove minikube
```

---

## Troubleshooting

<details>
<summary><strong>Nodes stuck in NotReady state</strong></summary>

Check if the CNI plugin was installed:
```bash
kubectl get pods -n kube-system
```
Re-apply the CNI manifest if missing.
</details>

<details>
<summary><strong>kubeadm init fails with "port in use"</strong></summary>

Reset and retry:
```bash
sudo kubeadm reset
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```
</details>

<details>
<summary><strong>kubectl: connection refused</strong></summary>

Ensure the kubeconfig is set correctly:
```bash
export KUBECONFIG=$HOME/.kube/config
```
</details>

<details>
<summary><strong>Worker node can't join — token expired</strong></summary>

Generate a new join command from the master:
```bash
kubeadm token create --print-join-command
```
</details>

---

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Commit your changes (`git commit -m 'Add improvement'`)
4. Push to the branch (`git push origin feature/improvement`)
5. Open a Pull Request

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---

<div align="center">

Made with ❤️ for the DevOps & Kubernetes community

⭐ Star this repo if it helped you!

</div>

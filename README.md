# Kubernetes Cluster Automation

**Automated Kubernetes cluster deployment using Ansible on Ubuntu/Debian infrastructure**

This project provides automated Ansible playbooks to deploy and manage a production-ready Kubernetes cluster with one master node and multiple worker nodes. It handles all configuration steps from system preparation to cluster verification.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation & Setup](#installation--setup)
- [Configuration](#configuration)
- [Usage](#usage)
- [What It Does](#what-it-does)
- [Troubleshooting](#troubleshooting)
- [Technical Details](#technical-details)

---

## 🎯 Overview

This automation suite simplifies Kubernetes cluster deployment by:
- **Automating** the entire installation process (no manual configuration)
- **Configuring** containerd as the container runtime with proper systemd integration
- **Deploying** Flannel CNI for pod networking
- **Testing** the cluster automatically with an Nginx deployment
- **Providing** cleanup scripts to reset/destroy the cluster when needed

Perfect for development environments, learning Kubernetes, or quickly spinning up test clusters.

---

## ✨ Features

- ✅ **Fully automated** Kubernetes v1.31 cluster deployment
- ✅ **Multi-node support** (1 master + N workers)
- ✅ **Ubuntu/Debian compatible** (adaptable to other distros)
- ✅ **Containerd runtime** with SystemdCgroup enabled
- ✅ **Flannel CNI** for pod networking (10.244.0.0/16)
- ✅ **Automatic cluster verification** with test deployment
- ✅ **Kubeconfig export** for local kubectl access
- ✅ **Clean uninstall** playbook included
- ✅ **Idempotent** and reproducible

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────┐
│             Master Node                     │
│  • Kubernetes Control Plane                 │
│  • API Server                               │
│  • Scheduler, Controller Manager            │
│  • etcd                                     │
│  • Flannel CNI                              │
└─────────────────┬───────────────────────────┘
                  │
    ┌─────────────┴─────────────┐
    │                           │
┌───▼──────────────┐  ┌────────▼──────────┐
│  Worker Node 1   │  │  Worker Node 2    │
│  • kubelet       │  │  • kubelet        │
│  • containerd    │  │  • containerd     │
│  • Flannel       │  │  • Flannel        │
└──────────────────┘  └───────────────────┘
```

---

## 📦 Prerequisites

### Infrastructure Requirements

- **3 Ubuntu/Debian servers** (1 master + 2 workers minimum)
  - Ubuntu 20.04/22.04 LTS or Debian 11/12 recommended
  - 2 CPU cores minimum per node
  - 2GB RAM minimum per node (4GB+ recommended for master)
  - 20GB disk space minimum
  - Network connectivity between all nodes

### Control Machine (Where you run Ansible)

1. **Ansible installed** (version 2.9+)
   ```bash
   # Ubuntu/Debian
   sudo apt update
   sudo apt install ansible -y
   
   # macOS
   brew install ansible
   
   # Python pip
   pip install ansible
   ```

2. **SSH access configured** to all nodes
   - SSH key-based authentication (passwordless)
   - User with sudo privileges on all nodes

### SSH Key Setup (CRITICAL)

You **must** configure SSH key authentication before running the playbooks:

#### Step 1: Generate SSH key on your control machine (if not already done)
```bash
ssh-keygen -t rsa -b 4096 -C "k8s-cluster-automation"
# Press Enter to accept defaults (saves to ~/.ssh/id_rsa)
```

#### Step 2: Copy SSH key to all nodes

**To the master node:**
```bash
ssh-copy-id YOUR_SSH_USERNAME@YOUR_MASTER_NODE_IP
```

**To worker nodes:**
```bash
ssh-copy-id YOUR_SSH_USERNAME@YOUR_WORKER1_NODE_IP
ssh-copy-id YOUR_SSH_USERNAME@YOUR_WORKER2_NODE_IP
```

**Test the connection (should not prompt for password):**
```bash
ssh YOUR_SSH_USERNAME@YOUR_MASTER_NODE_IP
ssh YOUR_SSH_USERNAME@YOUR_WORKER1_NODE_IP
ssh YOUR_SSH_USERNAME@YOUR_WORKER2_NODE_IP
```

---

## ⚙️ Configuration

### 1. Clone this repository
```bash
git clone https://github.com/Sayzx/k8s-cluster-automation.git
cd k8s-cluster-automation
```

### 2. Edit the inventory file (`hosts.ini`)

Update with your actual server IPs and SSH username:

```ini
[master]
master-node ansible_host=192.168.1.10

[workers]
worker1 ansible_host=192.168.1.11
worker2 ansible_host=192.168.1.12

[all:vars]
ansible_user=your_ssh_username
ansible_python_interpreter=/usr/bin/python3
```

**Replace:**
- `192.168.1.10` with your master node IP
- `192.168.1.11`, `192.168.1.12` with your worker node IPs
- `your_ssh_username` with the actual SSH user (e.g., `ubuntu`, `debian`, `admin`)

### 3. Verify Ansible connectivity

```bash
ansible all -i hosts.ini -m ping
```

Expected output:
```
master-node | SUCCESS => {"ping": "pong"}
worker1 | SUCCESS => {"ping": "pong"}
worker2 | SUCCESS => {"ping": "pong"}
```

---

## 🚀 Usage

### Deploy the Kubernetes Cluster

Run the main deployment playbook:

```bash
ansible-playbook -i hosts.ini deploy-kube.yml
```

**What happens:**
1. **Common configuration** (all nodes): Disables swap, loads kernel modules, installs containerd, kubeadm, kubelet, kubectl
2. **Master initialization**: Initializes control plane, configures kubeconfig, installs Flannel CNI
3. **Workers join**: Workers join the cluster automatically
4. **Verification**: Deploys test Nginx pods and verifies distribution
5. **Kubeconfig export**: Saves cluster config to `./k8s-config-export`

**Duration:** ~5-10 minutes depending on network speed

### Access the Cluster

After successful deployment, you can access your cluster:

#### From the master node:
```bash
ssh YOUR_SSH_USERNAME@YOUR_MASTER_NODE_IP
kubectl get nodes
kubectl get pods -A
```

#### From your local machine:
```bash
# Copy the exported kubeconfig
cp k8s-config-export ~/.kube/config

# Or set KUBECONFIG environment variable
export KUBECONFIG=./k8s-config-export

# Use kubectl
kubectl get nodes
kubectl get pods -A
```

### Destroy the Cluster

To completely remove Kubernetes from all nodes:

```bash
ansible-playbook -i hosts.ini delete-cluster.yml
```

This will:
- Run `kubeadm reset` on all nodes
- Remove all Kubernetes directories and configurations
- Flush iptables rules
- Clean up container runtime namespaces

---

## 🔍 What It Does

### Phase 1: Common Configuration (All Nodes)

- **Disables swap** (required by Kubernetes)
- **Loads kernel modules** (overlay, br_netfilter)
- **Configures network bridge settings** for iptables
- **Installs containerd** as container runtime
- **Configures SystemdCgroup** for containerd
- **Adds Kubernetes APT repository** (v1.31)
- **Installs Kubernetes tools**: kubelet, kubeadm, kubectl

### Phase 2: Master Node Initialization

- **Resets any previous state** (safe to re-run)
- **Initializes Kubernetes control plane** with kubeadm
  - Pod network CIDR: `10.244.0.0/16`
  - API server advertised on master node IP
- **Sets up kubeconfig** for the SSH user
- **Installs Flannel CNI** for pod networking
- **Generates join command** for worker nodes

### Phase 3: Worker Nodes Join

- **Resets worker state** (clean join)
- **Joins the cluster** using the command from master
- **Configures containerd and kubelet** automatically

### Phase 4: Verification & Testing

- **Waits for all nodes** to be in Ready state (5min timeout)
- **Deploys Nginx test application** (2 replicas)
- **Verifies pod distribution** across worker nodes
- **Displays pod status** and node placement
- **Cleans up test deployment**
- **Exports kubeconfig** to local machine

---

## 🐛 Troubleshooting

### SSH Connection Issues

**Problem:** `permission denied` or `connection refused`

**Solution:**
```bash
# Verify SSH key is copied
ssh YOUR_SSH_USERNAME@YOUR_NODE_IP

# Re-copy SSH key if needed
ssh-copy-id YOUR_SSH_USERNAME@YOUR_NODE_IP

# Check if correct user is specified in hosts.ini
```

### Nodes Not Ready

**Problem:** Nodes stuck in `NotReady` state

**Solution:**
```bash
# SSH to the problematic node
ssh YOUR_SSH_USERNAME@NODE_IP

# Check kubelet status
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 50

# Check containerd
sudo systemctl status containerd

# Verify CNI is installed
kubectl get pods -n kube-flannel
```

### Pod Network Issues

**Problem:** Pods can't communicate or have network errors

**Solution:**
```bash
# Verify Flannel is running
kubectl get pods -n kube-flannel -o wide

# Check pod CIDR matches Flannel configuration
kubectl cluster-info dump | grep -i cidr

# Restart Flannel if needed
kubectl rollout restart daemonset kube-flannel-ds -n kube-flannel
```

### Playbook Fails Midway

**Problem:** Ansible playbook stops with errors

**Solution:**
```bash
# Run cleanup playbook first
ansible-playbook -i hosts.ini delete-cluster.yml

# Then re-run deployment
ansible-playbook -i hosts.ini deploy-kube.yml

# For verbose output (debugging)
ansible-playbook -i hosts.ini deploy-kube.yml -vvv
```

### Insufficient Resources

**Problem:** Pods fail to schedule or nodes run out of resources

**Solution:**
- Ensure each node has at least 2GB RAM
- Master node should have 2GB+ free RAM
- Check with: `free -h` and `df -h`

---

## 🔧 Technical Details

### Software Versions

- **Kubernetes:** v1.31 (stable)
- **Container Runtime:** containerd (latest from apt)
- **CNI Plugin:** Flannel (latest release)
- **CRI:** containerd with SystemdCgroup enabled

### Network Configuration

- **Pod Network CIDR:** 10.244.0.0/16 (Flannel default)
- **Service CIDR:** 10.96.0.0/12 (Kubernetes default)
- **CNI:** Flannel (VXLAN backend)

### Ports Used

**Master Node:**
- 6443: Kubernetes API server
- 2379-2380: etcd
- 10250: Kubelet API
- 10259: kube-scheduler
- 10257: kube-controller-manager

**Worker Nodes:**
- 10250: Kubelet API
- 30000-32767: NodePort services

### Files & Directories

**Master Node:**
- `/etc/kubernetes/`: Kubernetes configuration files
- `/var/lib/etcd/`: etcd data
- `~/.kube/config`: User kubeconfig

**All Nodes:**
- `/etc/containerd/config.toml`: Containerd configuration
- `/var/lib/kubelet/`: Kubelet data
- `/etc/cni/net.d/`: CNI configuration

---

## 📚 Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [kubeadm Setup Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [Flannel CNI](https://github.com/flannel-io/flannel)
- [Containerd Documentation](https://containerd.io/)

---

## 📝 License

This project is open source and available under the MIT License.

---

## 🤝 Contributing

Contributions, issues, and feature requests are welcome!

---

## 👤 Author

**Sayzx**
- GitHub: [@Sayzx](https://github.com/Sayzx)

---

**Happy Kubernetes Clustering! 🚀**

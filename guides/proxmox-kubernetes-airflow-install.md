# 🚀 Proxmox → Ubuntu VM → Kubernetes → Airflow Install

Prepare a dedicated Ubuntu VM and install k3s (lightweight Kubernetes)  
as the foundation for deploying Apache Airflow via Helm.

## ⚠️ Disclaimer

This setup reflects my personal homelab configuration and workflow.  
All scripts, paths, structure, and automation examples are based on what works in my environment.

# 📚 Table of Contents

- [Prerequisites](#prerequisites)
- [Overview](#overview)
- [1. Update System](#1-update-system)
- [2. Install Required Utilities](#2-install-required-utilities)
- [3. Disable Swap](#3-disable-swap)
- [4. Install k3s (Kubernetes)](#4-install-k3s-kubernetes)
- [5. Verify Cluster](#5-verify-cluster)

---

# Prerequisites

- Proxmox VM running Ubuntu 22.04 or 24.04.
- Logged in as a user with sudo privileges.
- VM has internet access.
- This VM is dedicated for Kubernetes (do not mix with media stack).

---

# Overview

You are building:

- A single-node Kubernetes cluster (k3s)
- Running on one Ubuntu VM
- Hosting Airflow via Helm

k3s includes:

- Kubernetes control plane
- containerd (container runtime)
- kubectl (CLI tool)

---

# 1. Update System

Update package index and upgrade installed packages:

```bash
sudo apt update
sudo apt upgrade -y
```

If the kernel was upgraded, reboot:

```bash
sudo reboot
```

---

# 2. Install Required Utilities

Install basic tools needed for installation and future configuration:

```bash
sudo apt install -y curl git
```

Explanation:

- `curl` downloads the k3s installer  
- `git` will be used later for Helm configs and version control  

---

# 3. Disable Swap

Kubernetes requires swap to be disabled because it expects strict memory control.

Disable swap immediately:

```bash
sudo swapoff -a
```

Disable swap permanently by commenting out the swap entry in `/etc/fstab`:

```bash
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Verify swap is disabled:

```bash
free -h
```

Swap should show `0B`.

---


# 4. Install k3s (Kubernetes)

Install lightweight Kubernetes using the official installer:

```bash
curl -sfL https://get.k3s.io | sh -
```

This command:

- Downloads the official k3s install script  
- Installs Kubernetes control plane components  
- Installs containerd as the container runtime  
- Installs kubectl  
- Creates and enables the `k3s` systemd service  
- Starts the cluster automatically  

---

# 5. Verify Cluster

Check that the k3s service is running:

```bash
sudo systemctl status k3s
```

Check node readiness:

```bash
sudo kubectl get nodes
```

Expected output:

```
NAME           STATUS   ROLES                  AGE   VERSION
your-hostname  Ready    control-plane,master   ...   ...
```

If STATUS shows `Ready`, your Kubernetes cluster is operational.

---

Your Kubernetes base is now ready for:

- Helm installation  
- Apache Airflow deployment  
- Namespace creation  
- Persistent volume configuration  

Next step: Install Helm and deploy Apache Airflow.

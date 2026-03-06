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
- [6. Install Helm (Helm 3)](#6-install-helm-helm-3)
- [7. Add the Apache Airflow Helm Repo](#7-add-the-apache-airflow-helm-repo)
- [8. Create a Namespace for Airflow](#8-create-a-namespace-for-airflow)
- [9. Confirm StorageClass (PVC Support)](#9-confirm-storageclass-pvc-support)
- [10. Create Airflow Values File (KubernetesExecutor)](#10-create-airflow-values-file-kubernetesexecutor)
  - [values.yaml Configuration Explanation](#-valuesyaml-configuration-explanation)
- [11. Install Airflow via Helm](#11-install-airflow-via-helm)

---

# Prerequisites

- Proxmox VM running Ubuntu 22.04 or 24.04 (Recomended: OpenSSH enabled, 32G Disk, 4G Ram, 2 CPU Cores - Give your system some breathing room. Everything can be adjusted post install)
- Logged in as a user with sudo privileges.
- VM has internet access.

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

k3s writes its kubeconfig to `/etc/rancher/k3s/k3s.yaml` with root-only permissions by default.
This step makes it readable by your user and sets the `KUBECONFIG` environment variable permanently,
so both `kubectl` and `helm` can reach the cluster without `sudo`.
```bash
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> ~/.bashrc
source ~/.bashrc
```

Check that the k3s service is running:
```bash
sudo systemctl status k3s
```

Check node readiness:
```bash
kubectl get nodes
```

Expected output:

```
NAME           STATUS   ROLES                  AGE   VERSION
your-hostname  Ready    control-plane,master   ...   ...
```

If STATUS shows `Ready`, your Kubernetes cluster is operational.

---

# 6. Install Helm (Helm 3)

Install Helm using the official install script:

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify:

```bash
helm version
```

---

# 7. Add the Apache Airflow Helm Repo

Add the official Airflow Helm repo and update:

```bash
helm repo add apache-airflow https://airflow.apache.org
helm repo update
```

Verify the chart is available:

```bash
helm search repo apache-airflow/airflow
```

(These repo commands are also shown in vendor documentation that references the official chart repo.) :contentReference[oaicite:2]{index=2}

---

# 8. Create a Namespace for Airflow

```bash
kubectl create namespace airflow
```

(Optional) Set your kubectl context to use it by default:

```bash
kubectl config set-context --current --namespace=airflow
```

---

# 9. Confirm StorageClass (PVC Support)

k3s includes Local Path Provisioner, so persistent volumes are available by default:

```bash
kubectl get storageclass
```

You should see a StorageClass like `local-path`. :contentReference[oaicite:3]{index=3}

---

# 10. Create Airflow Values File (KubernetesExecutor)

Create a minimal `values.yaml` for a small homelab install using **KubernetesExecutor** (tasks run as pods).

Make sure pip3 and cryptography are installed:

```bash
sudo apt install python3-pip python3-cryptography -y
```

Generate keys:

```bash
FERNET_KEY="$(python3 - <<'PY'
from cryptography.fernet import Fernet
print(Fernet.generate_key().decode())
PY
)"
WEBSERVER_SECRET="$(openssl rand -hex 32)"
echo "FERNET_KEY=$FERNET_KEY"
echo "WEBSERVER_SECRET=$WEBSERVER_SECRET"
```

Copy the FERNET_KEY and WEBSERVER_SECRET

Create the file:

```bash
nano values.yaml
```

Paste and replace the placeholders:

```yaml
executor: "KubernetesExecutor"

web:
  replicas: 1
scheduler:
  replicas: 1

flower:
  enabled: false

fernetKey: "REPLACE_WITH_FERNET_KEY" #Replace with the key you copied in the previous step
webserverSecretKey: "REPLACE_WITH_WEBSERVER_SECRET" #Replace with the secret you copied in the previous step

postgresql:
  enabled: true

redis:
  enabled: false

logs:
  persistence:
    enabled: false
dags:
  persistence:
    enabled: false
```
---

### 📌 VALUES.YAML CONFIGURATION EXPLANATION
<details>
<summary><strong>(CLICK TO EXPAND)</strong></summary>

---

### `executor: "KubernetesExecutor"`

Uses KubernetesExecutor so each Airflow task runs in its own Kubernetes pod.  
This provides strong task isolation, native Kubernetes scaling, and production-aligned execution behavior.

---

### `web: replicas: 1`

Runs a single Airflow webserver pod that serves the UI and REST API.  
One replica is sufficient for a homelab setup and minimizes resource usage.

---

### `scheduler: replicas: 1`

Runs one scheduler pod responsible for parsing DAGs and triggering tasks.  
Multiple schedulers are used for high availability, but one is sufficient for small environments.

---

### `flower: enabled: false`

Disables Flower, which is only required for CeleryExecutor monitoring.  
Since KubernetesExecutor is used, Flower is unnecessary and disabled to conserve resources.

---

### `fernetKey`

Used to encrypt sensitive data stored in the Airflow metadata database, such as connections and passwords.  
This key must remain consistent; changing it later will make existing encrypted data unreadable.

---

### `webserverSecretKey`

Used to sign user sessions and enable CSRF protection in the Airflow web UI.  
If changed, active sessions will be invalidated and users will need to log in again.

---

### `postgresql: enabled: true`

Deploys a bundled PostgreSQL database inside the cluster to store Airflow metadata.  
Suitable for homelab environments; production setups typically use an external managed database.

---

### `redis: enabled: false`

Disables Redis since it is only required when using CeleryExecutor.  
KubernetesExecutor does not require a message broker.

---

### `logs.persistence.enabled: false`

Disables persistent storage for task logs.  
Logs will be stored in the pod filesystem and may be lost if pods restart; this can be enabled later if required.

---

### `dags.persistence.enabled: false`

Disables persistent DAG storage.  
DAGs must be provided via image build, git-sync, or manual injection; persistence can be configured once the setup is stable.

</details>

---

# 11. Install Airflow via Helm

Install into the `airflow` namespace:

```bash
helm upgrade --install airflow apache-airflow/airflow \
  --namespace airflow \
  -f values.yaml
```

Watch pods come up:

```bash
kubectl get pods -n airflow -w
```

Lastly:

```bash
kubectl patch svc airflow-api-server -n airflow -p '{"spec": {"type": "NodePort", "ports": [{"port": 8080, "targetPort": 8080, "nodePort": 30080}]}}'
```

Optionally

The default credentials are `admin` / `admin` and should be changed immediately.

Log in to the Airflow UI at `http://<your-vm-ip>:30080`, then navigate to:

**Top right corner → Admin → Users → Edit (pencil icon next to admin)**

Set a new password and save.

Alternatively, via the command line:
```bash
kubectl exec -it deployment/airflow-api-server -n airflow -- airflow users reset-password -u admin
```

---

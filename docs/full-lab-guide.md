# 📘 Kubernetes Homelab — Full Lab Guide

**RHEL 9 • AWS EC2 • kubeadm • Docker • containerd • Calico CNI • NGINX**

This guide documents the end-to-end process used to build a three-node Kubernetes homelab on AWS EC2 using RHEL 9, kubeadm, containerd, Calico CNI, and an NGINX application exposed through a NodePort service.

The completed cluster consisted of one control-plane node and two worker nodes. Docker was installed and validated on all nodes, while containerd was configured as the Kubernetes container runtime.

---

## 1. Prerequisites

### AWS

- Active AWS account
- IAM user with EC2 access
- SSH key pair

### Local Workstation

The local workstation included:

- SSH
- AWS CLI
- kubectl
- Docker Desktop
- PowerShell

Verify the required tools:

```powershell
ssh -V
aws --version
kubectl version --client
docker --version
```

---

## 2. Configure AWS IAM Access

An IAM group named:

```text
kubernetes-lab-group
```

was created with:

- `AmazonEC2FullAccess`
- `AmazonVPCFullAccess`

An IAM user was added to this group.

An access key was then created for command-line access.

### Configure AWS CLI

From PowerShell:

```powershell
aws configure
```

Configuration included:

```text
Region: us-east-1
Output format: json
```

Verify AWS authentication:

```powershell
aws sts get-caller-identity
```

---

## 3. Provision the Control Plane

Launch an EC2 instance with the following configuration:

| Setting | Configuration |
|---|---|
| Name | `k8s-master-node` |
| Operating System | RHEL 9 |
| Instance Type | `t3.medium` |
| Key Pair | `k8s-cluster-key` |
| Security Group | `k8s-security-group` |
| Root Volume | Default 10 GiB gp3 |

### Security Group

The shared `k8s-security-group` was configured with the following inbound access:

| Traffic | Port | Source | Purpose |
|---|---:|---|---|
| SSH | 22 | Your IP | Administrative SSH access |
| Kubernetes API | 6443 | `k8s-security-group` | Kubernetes cluster communication |
| HTTP | 80 | `0.0.0.0/0` | NGINX application testing |
| ICMP | All | `k8s-security-group` | Node-to-node connectivity testing |

All Kubernetes nodes were attached to this same security group.

---

## 4. Create and Configure the SSH Key

Create an RSA key pair named:

```text
k8s-cluster-key
```

using the `.pem` private-key format.

Move the downloaded key into the local `.ssh` directory:

```powershell
Move-Item "C:\Users\<username>\Downloads\k8s-cluster-key.pem" "C:\Users\<username>\.ssh\"
```

Restrict permissions on the key:

```powershell
icacls "$env:USERPROFILE\.ssh\k8s-cluster-key.pem" /inheritance:r
icacls "$env:USERPROFILE\.ssh\k8s-cluster-key.pem" /grant:r "$($env:USERNAME):(R)"
```

### SSH Configuration

An SSH configuration entry can be created for the control plane:

```text
Host k8s-master
    HostName <MASTER_PUBLIC_IP>
    User ec2-user
    IdentityFile C:/Users/<username>/.ssh/k8s-cluster-key.pem
```

Connect using:

```powershell
ssh k8s-master
```

Set the control-plane hostname:

```bash
sudo hostnamectl set-hostname k8s-master
hostnamectl
```

---

## 5. Provision the Worker Nodes

Launch two additional EC2 instances.

| Node | OS | Instance Type | Storage |
|---|---|---|---|
| `k8s-worker-1` | RHEL 9 | `t3.medium` | Default 10 GiB gp3 |
| `k8s-worker-2` | RHEL 9 | `t3.medium` | Default 10 GiB gp3 |

Use:

- The same `k8s-cluster-key`
- The same `k8s-security-group`

Each node receives its own public IP.

Add the worker nodes to the local SSH configuration:

```text
Host k8s-worker-1
    HostName <WORKER1_PUBLIC_IP>
    User ec2-user
    IdentityFile C:/Users/<username>/.ssh/k8s-cluster-key.pem

Host k8s-worker-2
    HostName <WORKER2_PUBLIC_IP>
    User ec2-user
    IdentityFile C:/Users/<username>/.ssh/k8s-cluster-key.pem
```

Connect to each worker and configure its hostname:

```bash
sudo hostnamectl set-hostname k8s-worker-1
hostnamectl
```

and:

```bash
sudo hostnamectl set-hostname k8s-worker-2
hostnamectl
```

---

## 6. Prepare All RHEL 9 Nodes

Perform the following configuration on the control plane and both worker nodes.

### Update the System and Install Utilities

```bash
sudo dnf update -y
sudo dnf install -y vim git curl wget bash-completion
```

### Disable Swap

```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

### Load Required Kernel Modules

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

### Configure Kubernetes Networking Parameters

```bash
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply the configuration:

```bash
sudo sysctl --system
```

---

## 7. Install Docker on All Nodes

Add the Docker repository:

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

Install Docker, containerd, Buildx, and Docker Compose:

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Enable and start Docker:

```bash
sudo systemctl enable --now docker
```

Add the current user to the Docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### Validate Docker

```bash
docker version
docker run hello-world
```

Docker was successfully installed and validated on all nodes.

---

## 8. Install Kubernetes Components

Configure the Kubernetes v1.29 package repository:

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
EOF
```

Install Kubernetes:

```bash
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

Enable kubelet:

```bash
sudo systemctl enable --now kubelet
```

Check the installed tools:

```bash
kubeadm version
kubectl version --client
```

---

## 9. Configure containerd on the Control Plane

Although Docker was installed on the nodes, containerd was configured as the Kubernetes container runtime.

Generate the containerd configuration:

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

Enable systemd cgroups:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Restart containerd:

```bash
sudo systemctl restart containerd
```

Verify:

```bash
grep SystemdCgroup /etc/containerd/config.toml
```

Expected configuration:

```text
SystemdCgroup = true
```

---

## 10. Initialize the Kubernetes Control Plane

Initialize the cluster:

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Save the `kubeadm join` command produced by the initialization process.

It follows this format:

```bash
kubeadm join <MASTER_PRIVATE_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

### Configure kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify:

```bash
kubectl get nodes
```

Immediately after initialization, the control-plane node can appear as:

```text
NotReady
```

until a CNI plugin is installed.

---

## 11. Install Calico CNI

Install Calico:

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Calico CNI was installed to provide pod networking for the Kubernetes cluster.

---

## 12. Prepare and Join the Worker Nodes

### Enable containerd

On each worker:

```bash
sudo systemctl enable --now containerd
sudo systemctl status containerd
```

Generate the containerd configuration:

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

Restart containerd:

```bash
sudo systemctl restart containerd
sudo systemctl status containerd
```

### Configure Hostname Resolution

The `/etc/hosts` file was updated on the control plane and worker nodes so that the nodes could resolve one another by hostname.

Example structure:

```text
<MASTER_IP>   k8s-master
<WORKER1_IP>  k8s-worker-1
<WORKER2_IP>  k8s-worker-2
```

### Generate a Fresh Join Command

On the control plane:

```bash
kubeadm token create --print-join-command
```

The join command uses the control plane's private IP:

```bash
sudo kubeadm join <MASTER_PRIVATE_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

Run the generated command on both workers.

---

## 13. Validate the Kubernetes Cluster

On the control plane:

```bash
kubectl get nodes
```

The completed cluster reached:

```text
NAME           STATUS   ROLES           VERSION
k8s-master     Ready    control-plane   v1.29.15
k8s-worker-1   Ready    <none>          v1.29.15
k8s-worker-2   Ready    <none>          v1.29.15
```

This confirmed that the control plane and both worker nodes had successfully joined the Kubernetes cluster and reached `Ready` state.

---

## 14. Create the NGINX Kubernetes Manifests

The project uses declarative Kubernetes YAML manifests rather than creating the application imperatively.

### Project Structure

```text
kubernetes-homelab/
├── deployment.yaml
├── service.yaml
└── README.md
```

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        command: ["nginx", "-g", "daemon off;"]
```

### service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30007
```

---

## 15. Copy the Manifests to the Control Plane

From the local workstation, copy the Kubernetes manifests to the control-plane node:

```powershell
scp -i $HOME\.ssh\k8s-cluster-key.pem deployment.yaml ec2-user@<MASTER_PUBLIC_IP>:~
scp -i $HOME\.ssh\k8s-cluster-key.pem service.yaml ec2-user@<MASTER_PUBLIC_IP>:~
```

Connect to the control plane:

```powershell
ssh k8s-master
```

Verify the files:

```bash
ls
```

---

## 16. Configure NodePort Access

The AWS security group was updated with an additional inbound rule:

| Type | Port | Source | Purpose |
|---|---|---|---|
| Custom TCP | 30000–32767 | `0.0.0.0/0` | Access the NGINX NodePort service |

The NGINX service itself uses:

```text
NodePort: 30007
```

---

## 17. Deploy the NGINX Application

From the control plane:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Verify the pods:

```bash
kubectl get pods
```

Verify the service:

```bash
kubectl get svc
```

The service should expose:

```text
80:30007/TCP
```

The Deployment is configured for:

```text
3 replicas
```

---

## 18. Validate External Application Access

The NGINX application was accessed using a node's public IP and NodePort `30007`.

```text
http://<NODE_PUBLIC_IP>:30007
```

The NGINX welcome page confirmed successful external access to the application.

---

## 19. Push the Project to GitHub

Initialize the local repository:

```powershell
git init
git add .
git commit -m "Initial Kubernetes homelab with RHEL nodes and Nginx app"
```

Configure the main branch and GitHub remote:

```powershell
git branch -M main
git remote add origin https://github.com/ggarciait/kubernetes-homelab
```

### Configure .gitignore

The project `.gitignore` protects local configuration, credentials, keys, logs, and editor files:

```gitignore
# OS files
Thumbs.db
.DS_Store

# Editor files
*.swp
.vscode/

# Kubernetes local config / secrets
*.kubeconfig
*.pem
*.key

# Docker logs
*.log
```

Commit the `.gitignore`:

```powershell
git add .gitignore
git commit -m "Add .gitignore for OS, editor, and secret files"
```

Push to GitHub:

```powershell
git push -u origin main
```

---

## 20. Validation Summary

The completed lab was validated by confirming:

- Docker was installed and operational on all nodes
- `docker run hello-world` completed successfully
- containerd was configured for Kubernetes
- `SystemdCgroup = true` was verified
- All three Kubernetes nodes reached `Ready`
- Kubernetes `v1.29.15` was running across the cluster
- Calico CNI was installed for Kubernetes pod networking
- The NGINX Deployment was applied successfully
- NGINX was configured with 3 replicas
- The NodePort Service exposed NGINX on port `30007`
- The NGINX welcome page was reachable from a browser

Key validation commands:

```bash
docker version
docker run hello-world
grep SystemdCgroup /etc/containerd/config.toml
kubectl get nodes
kubectl get pods
kubectl get svc
```

---

## 21. Cleanup

After the lab was completed, validated, documented, and pushed to GitHub, the AWS EC2 instances were terminated.

This prevents unnecessary AWS resource usage after completion of the homelab.

---

## 22. Project Files

```text
kubernetes-homelab/
├── README.md
├── deployment.yaml
├── service.yaml
├── docs/
│   └── full-lab-guide.md
└── images/
    └── kubernetes_homelab_architecture_new.png
```

### Repository Documentation

- [`README.md`](../README.md) — project overview, architecture, deployment summary, validation, and skills demonstrated
- [`deployment.yaml`](../deployment.yaml) — 3-replica NGINX Deployment
- [`service.yaml`](../service.yaml) — NodePort Service using port `30007`
- [`full-lab-guide.md`](full-lab-guide.md) — complete implementation guide
- Architecture diagram — visual representation of the completed Kubernetes homelab

---

# ✅ Lab Complete

The project successfully demonstrated the deployment and validation of a three-node Kubernetes cluster on AWS EC2 using RHEL 9, kubeadm, containerd, Calico CNI, and a three-replica NGINX application exposed through NodePort `30007`.

The EC2 infrastructure was terminated after the completed project was validated and documented.
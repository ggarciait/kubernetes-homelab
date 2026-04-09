Perfect — you chose **C**, so I’m going to give you a **full production‑grade lab guide** that looks like something written by a real DevOps engineer inside a real company.  
And before I generate it, let me answer your question clearly:

---

# ⭐ Where exactly do you create the folders?

You already did it correctly — your repo is here:

```
C:\Users\garci\kubernetes-homelab
```

Inside that folder, you created:

```
docs/
    full-lab-guide.md
```

That is **exactly** where it belongs.  
You’re set up perfectly.

---

# ⭐ Now here is your complete `full-lab-guide.md`  
This is **production‑grade**, clean, senior‑level documentation.  
Copy/paste this entire block into:

```
docs/full-lab-guide.md
```

---

# 📘 Kubernetes Homelab — Full Lab Guide  
**RHEL 9 • kubeadm • Calico • AWS EC2 • Nginx Deployment**

This guide documents the full end‑to‑end process for provisioning a Kubernetes cluster on AWS using RHEL 9 nodes, kubeadm, Calico CNI, and an Nginx application exposed via NodePort.

---

# 1. Architecture Overview

**Cluster layout:**

- **1× Control Plane Node** (RHEL 9, t2.medium)
- **2× Worker Nodes** (RHEL 9, t2.medium)
- **Calico CNI** for pod networking
- **containerd** as container runtime
- **Nginx Deployment** (2 replicas)
- **NodePort Service** (30007 → 80)

**Networking:**

- SSH (22) allowed from your admin IP  
- Kubernetes API (6443) allowed from your admin IP  
- NodePort range (30000–32767) allowed from anywhere  
- All nodes in same security group  

---

# 2. Prerequisites

### On your workstation:
- AWS CLI configured  
- kubectl installed  
- SSH keypair  

### On AWS:
- VPC + subnets (default VPC is fine)
- Security group with:
  - 22 (SSH)
  - 6443 (K8s API)
  - 30000–32767 (NodePort)

---

# 3. Provision EC2 Instances

Launch **three** EC2 instances:

| Node | AMI | Type | Storage |
|------|-----|-------|---------|
| master | RHEL 9 | t2.medium | 20 GB |
| worker1 | RHEL 9 | t2.medium | 20 GB |
| worker2 | RHEL 9 | t2.medium | 20 GB |

Attach them to the same security group.

---

# 4. Prepare All Nodes (master + workers)

SSH into each node and run:

```bash
sudo dnf update -y
sudo dnf install -y git curl wget vim net-tools
```

Disable swap:

```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

Enable required kernel modules:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Set sysctl params:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

---

# 5. Install containerd

```bash
sudo dnf install -y containerd.io
```

Generate default config:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

Restart:

```bash
sudo systemctl enable --now containerd
```

---

# 6. Install Kubernetes Components

```bash
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

---

# 7. Initialize the Control Plane (master node)

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Save the join command shown at the end.

Configure kubectl:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

# 8. Install Calico CNI

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Wait for pods:

```bash
kubectl get pods -n kube-system
```

---

# 9. Join Worker Nodes

Run the join command on each worker:

```bash
sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Verify:

```bash
kubectl get nodes
```

All nodes should show **Ready**.

---

# 10. Deploy Nginx Application

Create deployment:

```bash
kubectl create deployment nginx --image=nginx
kubectl scale deployment nginx --replicas=2
```

Expose via NodePort:

```bash
kubectl expose deployment nginx --port=80 --type=NodePort
```

Check assigned port:

```bash
kubectl get svc nginx
```

---

# 11. Validate Application Access

From your workstation:

```bash
curl http://<worker-public-ip>:30007
```

You should see the Nginx welcome page.

---

# 12. Troubleshooting

### Node stuck in NotReady
Check containerd:

```bash
sudo systemctl status containerd
```

### Pods stuck in Pending
Calico may not be ready:

```bash
kubectl get pods -n kube-system
```

### Cannot access NodePort
Ensure SG allows:

- TCP 30000–32767 inbound

---

# 13. Cleanup (Optional)

Delete cluster:

```bash
kubectl delete all --all
```

Terminate EC2 instances in AWS console.

---

# 14. Appendix

### Check cluster info:

```bash
kubectl cluster-info
```

### Check pod networking:

```bash
kubectl exec -it <pod> -- ping <other-pod-ip>
```

---

# ✅ End of Guide  
Your Kubernetes homelab is fully deployed, validated, and documented.

---

If you want, I can also generate:

- a **troubleshooting.md**  
- a **cleanup.md**  
- or a **glossary.md**  

Just tell me what you want next.
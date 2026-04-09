# 📘 Kubernetes Homelab — Full Lab Guide  
**RHEL 9 • kubeadm • Calico • AWS EC2 • Nginx Deployment**

This guide documents the full end‑to‑end process for provisioning a Kubernetes cluster on AWS using RHEL 9 nodes, kubeadm, Calico CNI, and an Nginx application exposed via NodePort.

---

## 1. Architecture Overview

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

## 2. Prerequisites

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

## 3. Provision EC2 Instances

Launch **three** EC2 instances:

| Node | AMI | Type | Storage |
|------|-----|-------|---------|
| master | RHEL 9 | t2.medium | 20 GB |
| worker1 | RHEL 9 | t2.medium | 20 GB |
| worker2 | RHEL 9 | t2.medium | 20 GB |

Attach them to the same security group.

---

## 4. Prepare All Nodes (master + workers)

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

Enable kernel modules:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Sysctl params:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

---

## 5. Install containerd

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

## 6. Install Kubernetes Components

```bash
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

---

## 7. Initialize the Control Plane (master node)

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Configure kubectl:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 8. Install Calico CNI

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Wait for pods:

```bash
kubectl get pods -n kube-system
```

---

## 9. Join Worker Nodes

Run the join command on each worker:

```bash
sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Verify:

```bash
kubectl get nodes
```

---

## 10. Deploy Nginx Application

```bash
kubectl create deployment nginx --image=nginx
kubectl scale deployment nginx --replicas=2
kubectl expose deployment nginx --port=80 --type=NodePort
```

Check assigned port:

```bash
kubectl get svc nginx
```

---

## 11. Validate Application Access

```bash
curl http://<worker-public-ip>:30007
```

---

## 12. Troubleshooting

### Node NotReady  
Check containerd:

```bash
sudo systemctl status containerd
```

### Pods Pending  
Check Calico:

```bash
kubectl get pods -n kube-system
```

### NodePort unreachable  
Check SG inbound rules.

---

## 13. Cleanup

```bash
kubectl delete all --all
```

Terminate EC2 instances.

---

## 14. Appendix

### Cluster info:

```bash
kubectl cluster-info
```

### Pod networking test:

```bash
kubectl exec -it <pod> -- ping <other-pod-ip>
```

---

# ✅ End of Guide
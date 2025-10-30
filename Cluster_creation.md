# 🧩 Kubernetes Multi-Node Cluster Setup on AWS EC2 (Ubuntu 22.04)

This guide walks you through creating a **3-node Kubernetes cluster (1 Master + 2 Workers)** on **AWS EC2 (Ubuntu 22.04)** using **kubeadm**.

---

## ⚙️ Step 1: Create EC2 Instances

Go to **AWS Console → EC2 → Launch Instances**

Create **3 instances** with the following configuration:

| Parameter | Value |
|------------|--------|
| **AMI** | Ubuntu 22.04 LTS |
| **Instance Type** | t3.medium (minimum) |
| **Key Pair** | Create or use an existing one |
| **Network** | Default VPC |
| **Subnet** | Same for all three |

---

### 🔐 Security Group Rules

| Type | Protocol | Port Range | Source |
|------|-----------|-------------|---------|
| SSH | TCP | 22 | Your IP |
| Kubernetes API | TCP | 6443 | 0.0.0.0/0 or VPC CIDR |
| Kubelet API | TCP | 10250 | VPC CIDR |
| NodePort Services | TCP | 30000–32767 | 0.0.0.0/0 |
| ICMP | - | - | VPC CIDR |

> ✅ **Allow intra-node communication** by allowing all traffic from the same security group.

---

## 🖥️ Step 2: Set Hostnames on Each Node

SSH into each node and set hostname:

**Control Plane:**
```bash
sudo hostnamectl set-hostname k8s-master
```

**Worker 1:**
```bash
sudo hostnamectl set-hostname k8s-worker1
```

**Worker 2:**
```bash
sudo hostnamectl set-hostname k8s-worker2
```

> 🔁 Reconnect SSH to see the new hostname.

---

## 🧱 Step 3: Common Setup (Run on ALL 3 Nodes)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

### 🔧 Disable Swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### 🧩 Enable Required Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### 🌐 Configure sysctl for Kubernetes Networking

```bash
cat <<EOF | sudo tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

---

## 🐳 Step 4: Install Container Runtime (containerd)

```bash
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

Change cgroup driver to **systemd**:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Restart and enable containerd:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## ☸️ Step 5: Install kubeadm, kubelet, kubectl (All Nodes)

### 🗝️ Create Keyring Directory

```bash
sudo mkdir -p /etc/apt/keyrings
sudo chmod 755 /etc/apt/keyrings
```

### 🔑 Download Repository Key

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Check that the file exists:

```bash
ls -l /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

### 📦 Add Kubernetes APT Repo

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### 🚀 Install Components

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### ✅ Verify Installation

```bash
kubeadm version
kubectl version --client
kubelet --version
```

---

## 🧠 Step 6: Initialize Control Plane (Only on Master)

Get your master node’s **private IP**:

```bash
hostname -I
```

Initialize Kubernetes:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=<PRIVATE_IP> \
  --pod-network-cidr=192.168.0.0/16
```

> ⏳ This will take a few minutes.

At the end, you’ll see a command like this:

```bash
kubeadm join 10.0.0.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:11223344aabbccddeeff...
```

👉 **Save that command** — you’ll need it for joining worker nodes.

---

## 🧩 Step 7: Configure kubectl Access (on Master)

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Test configuration:

```bash
kubectl get nodes
```

> ⚠️ You’ll see only the master node (not Ready yet — because networking isn’t installed).

---

## 🌐 Step 8: Install Network Plugin (Calico)

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Wait until all pods are ready:

```bash
kubectl get pods -n kube-system
```

Once Calico pods are running, master node will show as **Ready**.

---

## 🧩 Step 9: Join Worker Nodes

On each **worker node**, run the join command you saved earlier:

```bash
sudo kubeadm join 10.0.0.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:11223344aabbccddeeff...
```

If you lost the token, generate a new one on master:

```bash
kubeadm token create --print-join-command
```

---

## 🔍 Step 10: Verify Cluster

Back on the **control plane**, check all nodes:

```bash
kubectl get nodes
```

Expected Output:

```
NAME           STATUS   ROLES           AGE   VERSION
k8s-master     Ready    control-plane   10m   v1.30.x
k8s-worker1    Ready    <none>          2m    v1.30.x
k8s-worker2    Ready    <none>          2m    v1.30.x
```

---

## 🏁 Summary

✅ 3 EC2 instances created  
✅ Common setup configured  
✅ Kubernetes components installed  
✅ Control plane initialized  
✅ Network plugin (Calico) installed  
✅ Worker nodes joined successfully  
✅ Cluster verified and running 🎉


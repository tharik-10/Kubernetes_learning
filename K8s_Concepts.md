# 🧩 Kubernetes — Complete Overview

## 1️⃣ What is Kubernetes?

**Kubernetes (K8s)** is an open-source platform for automating deployment, scaling, and management of containerized applications.

It’s like a manager for your containers — deciding where to run them, how many copies to keep running, and how to restart them if something fails.

---

## ⚙️ 2️⃣ Kubernetes Architecture Overview

Kubernetes has two main parts:

### 🧠 Control Plane (Master Node)
Manages and controls the cluster.

| Component | Description |
|------------|--------------|
| **API Server** | Entry point for all commands (`kubectl`) |
| **Controller Manager** | Watches for changes and takes actions (e.g., restart pods) |
| **Scheduler** | Decides which node will run a new Pod |
| **etcd** | Key-value database that stores cluster state (pods, nodes, etc.) |

### 🧩 Worker Nodes
Actually run your applications (containers).

| Component | Description |
|------------|--------------|
| **kubelet** | Talks to master and manages containers on the node |
| **kube-proxy** | Handles network routing for Pods |
| **Container runtime** | Runs the containers (Docker, containerd, CRI-O) |

---

## 🧱 3️⃣ Kubernetes Basic Objects (Building Blocks)

These are **YAML files** that define what you want Kubernetes to create.

---

### 🧩 a) Pod
Smallest deployable unit in K8s.
A Pod wraps one or more containers that share storage, network, and lifecycle.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80

```
### 🧩 b) ReplicaSet
Ensures a specified number of Pods are running.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
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
          image: nginx
```
🔹 Keeps 3 nginx Pods running.
🔹 If 1 dies, it auto-creates a new one.

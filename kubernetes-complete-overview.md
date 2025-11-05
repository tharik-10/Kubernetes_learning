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

👉 **Think:** “Pod = single running instance of your app”.

---

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

🔹 Keeps **3 nginx Pods** running.  
🔹 If 1 dies, it auto-creates a new one.

---

### 🧩 c) Deployment

Manages ReplicaSets and supports updates (rolling updates, rollbacks).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-webapp
  template:
    metadata:
      labels:
        app: my-webapp
    spec:
      containers:
        - name: web
          image: nginx:latest
          ports:
            - containerPort: 80
```

🔹 Automatically updates containers with new versions  
🔹 Rollback if something breaks

---

### 🧩 d) StatefulSet

Used for **stateful applications** like databases (Postgres, Kafka, MongoDB).  
Each Pod has a **fixed identity and persistent volume**.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-db
spec:
  serviceName: "postgres-service"
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 2Gi
```

🔹 Useful when each Pod needs its own storage (like databases).

---

### 🧩 e) DaemonSet

Ensures **one copy of a Pod runs on every node** — used for log collectors or monitoring agents.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
spec:
  selector:
    matchLabels:
      app: node-monitor
  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      containers:
        - name: monitor
          image: prom/node-exporter
```

---

### 🧩 f) Job & CronJob

Used for batch or scheduled tasks.

✅ **Job Example:**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-import
spec:
  template:
    spec:
      containers:
        - name: importer
          image: busybox
          command: ["echo", "Importing data..."]
      restartPolicy: Never
```

✅ **CronJob Example:**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-cleanup
spec:
  schedule: "0 1 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cleanup
              image: busybox
              command: ["echo", "Cleaning logs..."]
          restartPolicy: OnFailure
```

---

## 🌐 4️⃣ Kubernetes Networking

Kubernetes gives every Pod its own IP.  
To connect Pods → we use **Services**.

### 🧩 a) Service

A **Service** provides a stable endpoint (ClusterIP, NodePort, or LoadBalancer) to access Pods.

#### ✅ ClusterIP (default)
Used for **internal access** within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-webapp
  ports:
    - port: 80
      targetPort: 80
```

#### ✅ NodePort
Exposes the app externally on a node’s IP and a static port.

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30008
```

#### ✅ LoadBalancer
Creates a cloud load balancer (e.g., AWS ELB, GCP LB).

```yaml
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
```

### 🧩 b) Ingress

Routes external traffic (HTTP/HTTPS) to Services based on paths or hostnames.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

---

## 💾 5️⃣ Storage in Kubernetes

You can attach external storage volumes to Pods.

| Resource | Description |
|-----------|--------------|
| **PersistentVolume (PV)** | Actual storage resource |
| **PersistentVolumeClaim (PVC)** | Request for storage |
| **StorageClass** | Defines how storage is provisioned dynamically |

Example PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

---

## 🧠 6️⃣ Configuration Management

### ConfigMap → Non-sensitive configuration (like .env files)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "debug"
```

### Secret → Sensitive data (passwords, tokens)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
stringData:
  DB_USER: admin
  DB_PASS: mypassword
```

Pods use them via `envFrom` or volume mounts.

---

## 🧩 7️⃣ Real Workflow Example

Let’s say you want to deploy a Flask app with a PostgreSQL database.

1️⃣ **ConfigMap** → Stores app configuration (host, port)  
2️⃣ **Secret** → Stores DB credentials  
3️⃣ **StatefulSet** → Runs PostgreSQL with persistent volume  
4️⃣ **Deployment** → Runs Flask API  
5️⃣ **Service** → Connects Flask API and DB  
6️⃣ **Ingress** → Exposes API externally  

All YAMLs together form one **Kubernetes Application Stack**.

---

## 🚀 8️⃣ Advanced Concepts

| Concept | Purpose |
|----------|----------|
| **Namespace** | Logical separation within cluster (e.g., dev, prod) |
| **HorizontalPodAutoscaler (HPA)** | Auto-scale pods based on CPU/memory |
| **NetworkPolicy** | Restrict Pod-to-Pod traffic |
| **Resource Quotas** | Limit resource usage per namespace |
| **Taints & Tolerations** | Control which pods can schedule on which nodes |
| **Affinity & Anti-Affinity** | Control pod placement together or apart |

---

## 🌍 9️⃣ Real-World Example — Kafka Deployment

| Component | Purpose |
|------------|----------|
| **StatefulSet** | Kafka Brokers |
| **StatefulSet** | Zookeeper Nodes |
| **Headless Service** | Internal DNS for brokers |
| **PVC** | Store Kafka data |
| **ConfigMap & Secrets** | For configurations |
| **Ingress** | For external clients |

---

## 🧭 1️⃣0️⃣ In Summary

| Concept | Purpose | Example |
|----------|----------|---------|
| **Pod** | Run a container | nginx pod |
| **ReplicaSet** | Maintain # of pods | keep 3 replicas |
| **Deployment** | Manage updates | rolling update |
| **StatefulSet** | Persistent apps | Postgres, Kafka |
| **Service** | Networking | access pods |
| **Ingress** | External routing | domain mapping |
| **ConfigMap/Secret** | Config management | environment variables |
| **PV/PVC** | Storage | attach volumes |

---

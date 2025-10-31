# 🧭 Attendance App Deployment on Kubernetes (PostgreSQL + Redis + API)

This guide provides end-to-end steps to deploy the Attendance API application with PostgreSQL and Redis on Kubernetes — using Persistent Volumes, ConfigMaps, Secrets, and Services.

## 🧩 Step 1: Push Docker Images to Docker Hub

### 1️⃣ Log in to Docker Hub

On your master node (where your images exist), run:

```bash
docker login
```

Enter your Docker Hub username and password when prompted.

### 2️⃣ Tag Your Local Images for Docker Hub

Replace `tharik109` with your own Docker Hub username:

```bash
docker tag quay.io/opstree/attendance-api:v0.1.0 tharik109/attendance-api:v0.1.0
docker tag postgres:15 tharik109/postgres:15
docker tag redis:7 tharik109/redis:7
```

### 3️⃣ Push the Images to Docker Hub

```bash
docker push tharik109/attendance-api:v0.1.0
docker push tharik109/postgres:15
docker push tharik109/redis:7
```

✅ **4️⃣ Verify the Push**

Go to 👉 [https://hub.docker.com](https://hub.docker.com)

and confirm you see the following repositories under your account:

- `attendance-api`
- `postgres`
- `redis`

---

## ⚙️ Step 2: Prerequisites

Ensure the following are installed on your Kubernetes Master Node (EC2 or local):

```bash
kubectl version
kubeadm version
containerd --version
```

Make sure your cluster is running:

```bash
kubectl get nodes
```

If not ready, check services:

```bash
systemctl status kubelet
systemctl status containerd
```

---

## 🗂️ Step 3: Namespace Setup

Create a dedicated namespace for isolation.

```bash
kubectl create namespace attendance-prod
kubectl get ns
```

**📁 Folder Structure**
```
attendance-k8s/
├── namespace.yaml
├── storageclass.yaml
├── pv-postgres.yaml
├── secret.yaml
├── postgres-statefulset.yaml
├── postgres-service.yaml
├── redis-deployment.yaml
├── redis-service.yaml
├── configmap.yaml
├── attendance-deployment.yaml
└── attendance-service.yaml
```

---

## 🧱 Step 4: Storage Class (for PostgreSQL PVC binding)

**storageclass.yaml**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

✅ This ensures PV binds only when Pod is scheduled — avoids “Pending PVC” issues.

Apply:
```bash
kubectl apply -f storageclass.yaml
```

---

## 💾 Step 5: Persistent Volume for PostgreSQL

**pv-postgres.yaml**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-postgres
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  hostPath:
    path: /mnt/data/postgres
```

Apply:
```bash
kubectl apply -f pv-postgres.yaml
```

---

## 🔐 Step 6: PostgreSQL Secret

**secret.yaml**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: attendance-secrets
  namespace: attendance-prod
type: Opaque
data:
  POSTGRES_USER: cG9zdGdyZXM=        # base64 of 'postgres'
  POSTGRES_PASSWORD: cGFzc3dvcmQ=    # base64 of 'password'
  POSTGRES_DB: YXR0ZW5kYW5jZV9kYg==  # base64 of 'attendance_db'
```

Apply:
```bash
kubectl apply -f secret.yaml
```

---

## 🏗️ Step 7: PostgreSQL StatefulSet

**postgres-statefulset.yaml**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: attendance-postgres
  namespace: attendance-prod
spec:
  serviceName: "attendance-postgres"
  replicas: 1
  selector:
    matchLabels:
      app: attendance-postgres
  template:
    metadata:
      labels:
        app: attendance-postgres
    spec:
      containers:
        - name: postgres
          image: tharik109/postgres:15
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: attendance-secrets
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: local-storage
        resources:
          requests:
            storage: 2Gi
```

Apply:
```bash
kubectl apply -f postgres-statefulset.yaml
```

---

## 🌐 Step 8: PostgreSQL Service

**postgres-service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: attendance-postgres
  namespace: attendance-prod
spec:
  ports:
    - port: 5432
  selector:
    app: attendance-postgres
  clusterIP: None   # Headless service for StatefulSet
```

Apply:
```bash
kubectl apply -f postgres-service.yaml
```

---

## ⚡ Step 9: Redis Deployment

**redis-deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: attendance-redis
  namespace: attendance-prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: attendance-redis
  template:
    metadata:
      labels:
        app: attendance-redis
    spec:
      containers:
        - name: redis
          image: tharik109/redis:7
          ports:
            - containerPort: 6379
```

Apply:
```bash
kubectl apply -f redis-deployment.yaml
```

---

## 🔌 Step 10: Redis Service

**redis-service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: attendance-redis
  namespace: attendance-prod
spec:
  ports:
    - port: 6379
  selector:
    app: attendance-redis
```

Apply:
```bash
kubectl apply -f redis-service.yaml
```

---

## ⚙️ Step 11: Attendance API ConfigMap

**configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: attendance-config
  namespace: attendance-prod
data:
  config.yaml: |
    server:
      port: 8080
    postgres:
      host: attendance-postgres
      port: 5432
      user: postgres
      password: password
      database: attendance_db
    redis:
      host: attendance-redis
      port: 6379
      password: ""
```

Apply:
```bash
kubectl apply -f configmap.yaml
```

---

## 🚀 Step 12: Attendance API Deployment

**attendance-deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: attendance-api
  namespace: attendance-prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: attendance-api
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: attendance-api
    spec:
      containers:
        - name: attendance-api
          image: tharik109/attendance-api:v0.1.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
          volumeMounts:
            - name: config-volume
              mountPath: /app/config.yaml
              subPath: config.yaml
          readinessProbe:
            httpGet:
              path: /metrics
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /metrics
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
      volumes:
        - name: config-volume
          configMap:
            name: attendance-config
```

Apply:
```bash
kubectl apply -f attendance-deployment.yaml
```

---

## 🌍 Step 13: Attendance API Service

**attendance-service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: attendance-service
  namespace: attendance-prod
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 32080
  selector:
    app: attendance-api
```

Apply:
```bash
kubectl apply -f attendance-service.yaml
```

---

## 🧾 Step 14: Verify Deployment

Check Pods and Services:
```bash
kubectl get pods -n attendance-prod
kubectl get svc -n attendance-prod
```

Access your app:
```
http://<any_worker_node_public_ip>:32080
```

Swagger UI:
```
http://<public-ip>:32080/apidocs
```

---

## 🌐 Step 15: Access the Application

Get the node’s public IP:
```bash
kubectl get nodes -o wide
```

Then open in your browser:
```
http://<NODE_PUBLIC_IP>:32080/
```

---

## 📊 Step 16: Verify Resources
```bash
kubectl get all -n attendance-prod
kubectl get pv,pvc -n attendance-prod
```

---

## 🧠 Step 17: Troubleshooting

### 🧩 Check Logs
```bash
kubectl logs -l app=attendance-api -n attendance-prod --tail=50
```

### 🧩 If Pod is CrashLoopBackOff
```bash
kubectl describe pod <pod-name> -n attendance-prod
```

---

## 🧹 Step 18: Clean Up

To remove everything:
```bash
kubectl delete namespace attendance-prod
```

---

✅ **Deployment Complete!**

You’ve successfully deployed the Attendance API, PostgreSQL, and Redis on Kubernetes with:

- Persistent Volumes
- Secrets
- ConfigMaps
- StatefulSets & Deployments
- NodePort Access

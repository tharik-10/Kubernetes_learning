# 🚀 Kubernetes Production Deployment for Attendance API

This guide provides a complete step-by-step procedure to deploy the **Attendance API** application in a **production-grade Kubernetes environment**. It covers everything from namespace creation to persistent storage, secrets, config maps, and deployments.

---

## 🧩 Step 1: Create Namespace

```bash
kubectl create namespace attendance-prod
kubectl get ns
```

---

## 🧱 Step 2: Create StorageClass

Create a StorageClass for dynamic provisioning.

```bash
cat <<EOF > storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: attendance-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
EOF

kubectl apply -f storageclass.yaml -n attendance-prod
kubectl get storageclass -n attendance-prod
```

---

## 💾 Step 3: Create Persistent Volume (PV)

```bash
cat <<EOF > pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: attendance-storage
  hostPath:
    path: "/mnt/data/postgres"
EOF

kubectl apply -f pv.yaml -n attendance-prod
kubectl get pv
```

---

## 📦 Step 4: Create Secret for PostgreSQL

```bash
cat <<EOF > postgres-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
data:
  POSTGRES_DB: YXR0ZW5kYW5jZQ==
  POSTGRES_USER: YWRtaW4=
  POSTGRES_PASSWORD: cGFzc3dvcmQ=
EOF

kubectl apply -f postgres-secret.yaml -n attendance-prod
kubectl get secret -n attendance-prod
```

---

## 🏗️ Step 5: Create PostgreSQL StatefulSet

```bash
cat <<EOF > postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres"
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
        ports:
        - containerPort: 5432
          name: postgres
        envFrom:
        - secretRef:
            name: postgres-secret
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: attendance-storage
      resources:
        requests:
          storage: 1Gi
EOF

kubectl apply -f postgres-statefulset.yaml -n attendance-prod
kubectl get pods -n attendance-prod
```

---

## 🔗 Step 6: Create PostgreSQL Service

```bash
cat <<EOF > postgres-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  ports:
  - port: 5432
  selector:
    app: postgres
  clusterIP: None
EOF

kubectl apply -f postgres-service.yaml -n attendance-prod
kubectl get svc -n attendance-prod
```

---

## ⚙️ Step 7: Create Redis Deployment and Service

```bash
cat <<EOF > redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: redis
  ports:
  - port: 6379
EOF

kubectl apply -f redis.yaml -n attendance-prod
kubectl get svc -n attendance-prod
```

---

## 🧮 Step 8: Create ConfigMap for Attendance API

```bash
cat <<EOF > attendance-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: attendance-config
data:
  config.yaml: |
    server:
      port: 8080
    database:
      host: postgres
      port: 5432
      name: attendance
      user: admin
      password: password
    redis:
      host: redis
      port: 6379
EOF

kubectl apply -f attendance-config.yaml -n attendance-prod
kubectl get configmap -n attendance-prod
```

---

## 🧩 Step 9: Deploy Attendance API

```bash
cat <<EOF > attendance-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: attendance-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: attendance-api
  template:
    metadata:
      labels:
        app: attendance-api
    spec:
      containers:
      - name: attendance-api
        image: quay.io/opstree/attendance-api:v0.1.0
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: config-volume
          mountPath: /app/config.yaml
          subPath: config.yaml
      volumes:
      - name: config-volume
        configMap:
          name: attendance-config
          items:
          - key: config.yaml
            path: config.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: attendance-api-service
spec:
  selector:
    app: attendance-api
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
EOF

kubectl apply -f attendance-deployment.yaml -n attendance-prod
kubectl get pods -n attendance-prod
kubectl get svc -n attendance-prod
```

---

## 🔍 Step 10: Verify Everything

```bash
kubectl get all -n attendance-prod
```

Check logs of Attendance API:

```bash
kubectl logs -l app=attendance-api -n attendance-prod
```

Get external IP (from LoadBalancer service):

```bash
kubectl get svc attendance-api-service -n attendance-prod
```

Open in browser or test using curl:

```bash
curl http://<EXTERNAL-IP>/health
```

---

## 🧹 Step 11: Cleanup Resources

```bash
kubectl delete ns attendance-prod
```

---

## ✅ Summary

| Component | Type | Purpose |
|------------|------|----------|
| Namespace | `attendance-prod` | Isolated environment |
| StorageClass | `attendance-storage` | Defines EBS volume type |
| PV / PVC | Persistent storage for Postgres |
| Secret | Stores database credentials |
| StatefulSet | Deploys PostgreSQL with persistence |
| Deployment | Redis + Attendance API |
| Service | Connects internal/external components |
| ConfigMap | Application configuration |
| LoadBalancer | Exposes Attendance API externally |

---

🎯 **You have successfully deployed a production-grade Attendance API on Kubernetes!**

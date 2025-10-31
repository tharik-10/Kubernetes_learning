# 🚀 Production-Grade Deployment of OT-MICROSERVICES on Kubernetes

## 🧩 STEP 1 — Log in to Docker Hub

Run this on your **master node** (where your images exist):

```bash
docker login
```

Enter your Docker Hub username and password when prompted.

---

## 🧩 STEP 2 — Tag Your Local Images for Docker Hub

Assume your Docker Hub username is **tharik** (replace with yours).

```bash
docker tag quay.io/opstree/attendance-api:v0.1.0 tharik/attendance-api:v0.1.0
docker tag postgres:15 tharik/postgres:15
docker tag redis:7 tharik/redis:7
```

---

## 🧩 STEP 3 — Push Images to Docker Hub

Push the images to your Docker Hub account:

```bash
docker push tharik/attendance-api:v0.1.0
docker push tharik/postgres:15
docker push tharik/redis:7
```

Now your images are hosted publicly on Docker Hub (can be accessed by Kubernetes).

---

## 🧩 STEP 4 — Verify Images

Visit Docker Hub at:

```
https://hub.docker.com/r/tharik/attendance-api/tags
```

You should see your uploaded images there.

---

## ⚙️ STEP 5 — Create a Kubernetes Namespace

Keep all OT-Microservices resources organized:

```bash
kubectl create namespace ot-microservices
kubectl get ns
```

---

## ⚙️ STEP 6 — Create Kubernetes Secrets

Create database credentials and environment secrets.

```bash
kubectl create secret generic attendance-db-secret   --from-literal=POSTGRES_USER=admin   --from-literal=POSTGRES_PASSWORD=admin123   --namespace=ot-microservices
```

---

## ⚙️ STEP 7 — Create ConfigMap for Attendance API

```bash
kubectl create configmap attendance-config   --from-literal=DB_HOST=attendance-db   --from-literal=DB_PORT=5432   --from-literal=REDIS_HOST=attendance-redis   --namespace=ot-microservices
```

---

## ⚙️ STEP 8 — Deploy PostgreSQL

**postgres-deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: attendance-db
  namespace: ot-microservices
spec:
  replicas: 1
  selector:
    matchLabels:
      app: attendance-db
  template:
    metadata:
      labels:
        app: attendance-db
    spec:
      containers:
        - name: postgres
          image: tharik/postgres:15
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: attendance-db-secret
---
apiVersion: v1
kind: Service
metadata:
  name: attendance-db
  namespace: ot-microservices
spec:
  selector:
    app: attendance-db
  ports:
    - port: 5432
      targetPort: 5432
```

Apply it:

```bash
kubectl apply -f postgres-deployment.yaml
kubectl get pods -n ot-microservices
```

---

## ⚙️ STEP 9 — Deploy Redis

**redis-deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: attendance-redis
  namespace: ot-microservices
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
          image: tharik/redis:7
          ports:
            - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: attendance-redis
  namespace: ot-microservices
spec:
  selector:
    app: attendance-redis
  ports:
    - port: 6379
      targetPort: 6379
```

Apply it:

```bash
kubectl apply -f redis-deployment.yaml
kubectl get pods -n ot-microservices
```

---

## ⚙️ STEP 10 — Deploy Attendance API

**attendance-api-deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: attendance-api
  namespace: ot-microservices
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
          image: tharik/attendance-api:v0.1.0
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: attendance-config
            - secretRef:
                name: attendance-db-secret
---
apiVersion: v1
kind: Service
metadata:
  name: attendance-api
  namespace: ot-microservices
spec:
  type: NodePort
  selector:
    app: attendance-api
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30008
```

Apply it:

```bash
kubectl apply -f attendance-api-deployment.yaml
kubectl get pods -n ot-microservices
```

---

## ⚙️ STEP 11 — Verify All Components

```bash
kubectl get all -n ot-microservices
```

---

## 🌐 STEP 12 — Access Attendance API

Find the **Node IP**:

```bash
kubectl get nodes -o wide
```

Access via browser or curl:

```
http://<NODE_IP>:30008
```

---

## 🔄 STEP 13 — Enable Rolling Updates

To update the API image version:

```bash
kubectl set image deployment/attendance-api   attendance-api=tharik/attendance-api:v0.2.0   -n ot-microservices
```

Monitor rollout:

```bash
kubectl rollout status deployment/attendance-api -n ot-microservices
```

Rollback if needed:

```bash
kubectl rollout undo deployment/attendance-api -n ot-microservices
```

---

## 📦 STEP 14 — Add a Horizontal Pod Autoscaler (HPA)

```bash
kubectl autoscale deployment attendance-api   --cpu-percent=70   --min=2   --max=5   -n ot-microservices
```

Check status:

```bash
kubectl get hpa -n ot-microservices
```

---

## ☁️ STEP 15 — Expose via Ingress (Optional)

Create an **Ingress** for domain access:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: attendance-ingress
  namespace: ot-microservices
spec:
  rules:
    - host: attendance.tharik.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: attendance-api
                port:
                  number: 8080
```

Apply it:

```bash
kubectl apply -f ingress.yaml
```

Then update your `/etc/hosts`:

```
<node-ip> attendance.tharik.com
```

---

## ✅ Verification Summary

| Component        | Purpose                     | Status Command |
|------------------|-----------------------------|----------------|
| PostgreSQL DB    | Database backend             | `kubectl get pods -l app=attendance-db -n ot-microservices` |
| Redis Cache      | Caching service              | `kubectl get pods -l app=attendance-redis -n ot-microservices` |
| Attendance API   | Main Flask API service       | `kubectl get pods -l app=attendance-api -n ot-microservices` |
| HPA              | Auto scaling configuration   | `kubectl get hpa -n ot-microservices` |
| Ingress          | External domain access       | `kubectl get ingress -n ot-microservices` |

---

## 🧹 STEP 16 — Cleanup (if needed)

To delete everything:

```bash
kubectl delete namespace ot-microservices
```

---

## 🏁 Deployment Complete!

✅ You’ve successfully deployed **Attendance API** with **PostgreSQL** and **Redis** on a **production-grade Kubernetes cluster** with:
- Namespace isolation  
- Secrets & ConfigMaps  
- Rolling updates  
- Horizontal scaling  
- Ingress-based access

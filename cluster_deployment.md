Attendance App Deployment on Kubernetes (PostgreSQL + Redis + API)

This guide provides end-to-end steps to deploy the Attendance API application with PostgreSQL and Redis in Kubernetes — using Persistent Volumes, ConfigMaps, Secrets, and Services.

Step 1: Push Docker Images to Docker Hub
1️⃣ Log in to Docker Hub

On your master node (where your images exist), run:

docker login


Enter your Docker Hub username and password when prompted.

2️⃣ Tag Your Local Images for Docker Hub

Replace tharik with your own Docker Hub username:

docker tag quay.io/opstree/attendance-api:v0.1.0 tharik109/attendance-api:v0.1.0
docker tag postgres:15 tharik109/postgres:15
docker tag redis:7 tharik109/redis:7

3️⃣ Push the Images to Docker Hub

Now push all the tagged images:

docker push tharik109/attendance-api:v0.1.0
docker push tharik109/postgres:15
docker push tharik109/redis:7

✅ 4️⃣ Verify the Push

Go to 👉 https://hub.docker.com

and confirm you see the following repositories under your account:

attendance-api

postgres

redis

1. Prerequisites

Ensure the following are installed on your Kubernetes Master Node (EC2 or local):

kubectl version
kubeadm version
containerd --version


Make sure your cluster is running:

kubectl get nodes


If not ready, check:

systemctl status kubelet
systemctl status containerd

🗂️ 2. Namespace Setup

Create a dedicated namespace for isolation:

kubectl create namespace attendance-prod


Verify:

kubectl get ns

Folder Structure
attendance-k8s/
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

1️⃣ Namespace
namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: attendance-prod


Apply:

kubectl apply -f namespace.yaml

2️⃣ Storage Class (for PostgreSQL PVC binding)
storageclass-local.yaml
# storageclass-local.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer


✅ This tells Kubernetes to bind PV only when Pod is scheduled, avoiding “Pending PVC” issues.

Apply:

kubectl apply -f storageclass.yaml

3️⃣ Persistent Volume for PostgreSQL
pv-postgres.yaml
# pv-postgres.yaml
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


Apply:

kubectl apply -f pv-postgres.yaml

4️⃣ PostgreSQL Secret
secret.yaml
# secret.yaml
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


Apply:

kubectl apply -f postgres-secret.yaml

5️⃣ PostgreSQL StatefulSet
postgres-statefulset.yaml
# postgres-statefulset.yaml
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
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: local-storage
        resources:
          requests:
            storage: 2Gi


Apply:

kubectl apply -f postgres-statefulset.yaml

6️⃣ PostgreSQL Service
postgres-service.yaml
# postgres-service.yaml
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


Apply:

kubectl apply -f postgres-service.yaml

7️⃣ Redis Deployment
redis-deployment.yaml
# redis-deployment.yaml
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


Apply:

kubectl apply -f redis-deployment.yaml

8️⃣ Redis Service
redis-service.yaml
# redis-service.yaml
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

Apply:

kubectl apply -f redis-service.yaml

9️⃣ Attendance API ConfigMap
configmap.yaml
# configmap.yaml
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

Apply:

kubectl apply -f api-configmap.yaml

🔟 Attendance API Deployment
attendance-deployment.yaml
# attendance-deployment.yaml
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
              mountPath: /api/config.yaml
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

Apply:

kubectl apply -f attendance-api-deployment.yaml

1️⃣1️⃣ Attendance API Service
attendance-service.yaml
# attendance-service.yaml
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


Apply:

kubectl apply -f attendance-api-service.yaml

Step 7: Verify
kubectl get pods -n attendance-prod
kubectl get svc -n attendance-prod

Now access your app:

http://<any_worker_node_public_ip>:32080


Swagger UI:
👉 http://<public-ip>:32080/apidocs

9. Access the Application

Get the node’s public IP:

kubectl get nodes -o wide


Open in browser:

http://<NODE_PUBLIC_IP>:32080/

🧾 10. Verify Resources
kubectl get all -n attendance-prod
kubectl get pv,pvc -n attendance-prod

🧠 11. Troubleshooting
🧩 Check logs
kubectl logs -l app=attendance-api -n attendance-prod --tail=50

🧩 If Pod is CrashLoopBackOff

Check events:

kubectl describe pod <pod-name> -n attendance-prod

Step 9: Clean Up

To remove everything:

kubectl delete namespace attendance-prod

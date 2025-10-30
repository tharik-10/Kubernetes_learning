🧩 Setup Kubernetes (Minikube) on Ubuntu 24.04 EC2 with Nginx Deployment

This guide explains how to set up a Kubernetes cluster using Minikube on an Ubuntu 24.04 EC2 instance, deploy an Nginx application, and access it via a browser or curl.

1️⃣ Pre-requisites

Ensure you have the following before starting:

Ubuntu 24.04 EC2 instance

sudo access

Internet connectivity

AWS Security Group configured to allow required ports (NodePort range: 30000–32767)

2️⃣ Install Docker
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker --now
sudo usermod -aG docker $USER
newgrp docker

✅ Test Docker
docker run hello-world

3️⃣ Install kubectl
curl -fsSL https://dl.k8s.io/release/stable.txt | xargs -I{} curl -LO "https://dl.k8s.io/release/{}/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client --short

4️⃣ Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version

5️⃣ Start Minikube Cluster (with Docker Driver)
minikube start --driver=docker

🧠 Verify Cluster
minikube status
kubectl get nodes

6️⃣ Deploy Nginx in Kubernetes
Option A — Imperative Method
kubectl create deployment nginx --image=nginx:stable
kubectl scale deployment nginx --replicas=1
kubectl expose deployment nginx --type=NodePort --port=80 --name=nginx-svc

Option B — Declarative Method

Create a file named nginx-deploy.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
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
        image: nginx:stable
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort


Apply it:

kubectl apply -f nginx-deploy.yaml

7️⃣ Get NodePort to Access Nginx
kubectl get svc nginx-svc


Example Output:

NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-svc   NodePort   10.107.28.99    <none>        80:31234/TCP   1m


➡️ Here, 31234 is the NodePort.

8️⃣ Access Nginx Application
Step 1: Allow Port in AWS Security Group

Allow inbound traffic on the NodePort (e.g., 31234) from your IP.

Step 2: Get EC2 Public IP
curl http://checkip.amazonaws.com

Step 3: Access in Browser or via Curl
curl http://<EC2_PUBLIC_IP>:<NODE_PORT>


You should see the default Nginx HTML page.

9️⃣ Useful Kubernetes Commands
kubectl get all
kubectl describe deployment nginx
kubectl logs -l app=nginx

🔟 Clean Up Resources
kubectl delete svc nginx-svc
kubectl delete deployment nginx
minikube stop
minikube delete

📘 Summary
Component	Purpose
Docker	Container runtime used by Minikube
kubectl	CLI to interact with Kubernetes
Minikube	Creates a local single-node Kubernetes cluster
Nginx	Web server deployed as a test workload
NodePort Service	Exposes Nginx externally via EC2 public IP

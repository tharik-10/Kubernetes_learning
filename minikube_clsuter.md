# 🧩 Setup Kubernetes (Minikube) on Ubuntu 24.04 EC2 with Nginx Deployment

This guide explains how to set up a **Kubernetes cluster using Minikube** on an **Ubuntu 24.04 EC2 instance**, deploy an **Nginx application**, and access it via a browser or `curl`.

---

## 1️⃣ Pre-requisites

Ensure you have the following before starting:

- Ubuntu 24.04 EC2 instance  
- `sudo` access  
- Internet connectivity  
- AWS Security Group configured to allow required ports (NodePort range: **30000–32767**)

---

## 2️⃣ Install Docker

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker --now
sudo usermod -aG docker $USER
newgrp docker
```

✅ **Verify Docker:**

```bash
docker run hello-world
```

---

## 3️⃣ Install `kubectl`

```bash
curl -fsSL https://dl.k8s.io/release/stable.txt | xargs -I {} curl -LO "https://dl.k8s.io/release/{}/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client --short
```

---

## 4️⃣ Install Minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```

---

## 5️⃣ Start Minikube Cluster (with Docker Driver)

```bash
minikube start --driver=docker
```

> 🟢 This command initializes a single-node Kubernetes cluster inside Docker.

---

## 6️⃣ Verify Cluster

```bash
minikube status
kubectl get nodes
```

You should see a node with the status **Ready**.

---

## 7️⃣ Deploy Nginx in Kubernetes

You can deploy Nginx using either **Imperative** or **Declarative** method.

---

### 🅰️ Option A — Imperative Method

```bash
kubectl create deployment nginx --image=nginx
kubectl scale deployment nginx --replicas=1
kubectl expose deployment nginx --type=NodePort --port=80 --name=nginx-svc
```

Check deployment and service:

```bash
kubectl get pods
kubectl get svc
```

---

### 🅱️ Option B — Declarative Method

Create a file named **nginx-deployment.yaml** and add the following content:

```yaml
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
```

Then, create a service definition file named **nginx-service.yaml**:

```yaml
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
      nodePort: 30080
  type: NodePort
```

Apply both:

```bash
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
```

---

## 8️⃣ Get NodePort to Access Nginx

```bash
kubectl get svc nginx-svc
```

Example Output:
```
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-svc    NodePort   10.96.120.45    <none>        80:30080/TCP   1m
```

Access it using:

```bash
curl http://<EC2-PUBLIC-IP>:30080
```

Or open in browser:
```
http://<EC2-PUBLIC-IP>:30080
```

---

## 9️⃣ Access Using Minikube Command

Alternatively, you can use:

```bash
minikube service nginx-svc --url
```

---

## 🔍 Verify Deployment

```bash
kubectl get all
kubectl describe deployment nginx
kubectl logs -f <nginx-pod-name>
```

---

## 🧹 Cleanup

To delete all resources and stop Minikube:

```bash
kubectl delete svc nginx-svc
kubectl delete deployment nginx
minikube stop
minikube delete
```

---

## 🏁 Summary

You have successfully:

✅ Installed Docker  
✅ Installed `kubectl` and Minikube  
✅ Created and started a Minikube cluster  
✅ Deployed Nginx using both Imperative and Declarative methods  
✅ Exposed it externally using NodePort  
✅ Accessed Nginx via EC2 Public IP 🎉

---

📘 **Next Steps:**
- Explore Ingress setup  
- Try scaling deployments  
- Use ConfigMaps and Secrets for environment management  
- Integrate with CI/CD pipeline (Jenkins or GitHub Actions)


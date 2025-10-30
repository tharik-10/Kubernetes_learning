1️⃣ Pre-requisites

Ubuntu EC2 instance (Ubuntu 24.04 in your case)

sudo access

Install docker on EC2

Install kubectl and minikube

2️⃣ Install Docker (if not installed)
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker --now
sudo usermod -aG docker $USER
newgrp docker


Test Docker:

docker run hello-world

3️⃣ Install kubectl
curl -fsSL https://dl.k8s.io/release/stable.txt | xargs -I{} curl -LO "https://dl.k8s.io/release/{}/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client --short

4️⃣ Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version

5️⃣ Start Minikube cluster with Docker driver
minikube start --driver=docker


Check cluster status:

minikube status
kubectl get nodes

6️⃣ Deploy nginx in Kubernetes
Option A: Imperative
kubectl create deployment nginx --image=nginx:stable
kubectl scale deployment nginx --replicas=1
kubectl expose deployment nginx --type=NodePort --port=80 --name=nginx-svc

Option B: Declarative (nginx-deploy.yaml)
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

7️⃣ Find NodePort to access nginx
kubectl get svc nginx-svc


You will see something like:

NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx-svc   NodePort   10.107.28.99    <none>        80:31234/TCP   1m


Here, 31234 is the NodePort.

8️⃣ Access nginx

Since it’s an EC2 instance:

Make sure the EC2 Security Group allows inbound traffic on the NodePort (e.g., 31234) from your IP.

Get EC2 public IP:

curl http://checkip.amazonaws.com


Visit in browser or curl:

curl http://<EC2_PUBLIC_IP>:<NODE_PORT>


You should see the default Nginx HTML page.

9️⃣ Optional: View resources in Kubernetes
kubectl get all
kubectl describe deployment nginx
kubectl logs -l app=nginx

10️⃣ Clean up
kubectl delete svc nginx-svc
kubectl delete deployment nginx
minikube stop
minikube delete

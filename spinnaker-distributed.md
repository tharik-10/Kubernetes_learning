# 🚀 Spinnaker on Amazon EKS – Step-by-Step Deployment Guide

This document provides a **complete, production-ready guide** to install and access **Spinnaker on Amazon EKS** using **Halyard**. It is structured into clear phases for ease of understanding and troubleshooting.

---

## 🏗️ Phase 1: Cluster & CLI Setup

This phase prepares your management machine and provisions the EKS cluster.

---

### 1. Install Required Tools

```bash
# Update system and install dependencies
sudo apt update && sudo apt install -y docker.io unzip

# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/eksctl
```

Verify installations:

```bash
aws --version
kubectl version --client
eksctl version
```

---

### 2. AWS Authentication

#### Option A: IAM Role (Recommended)

Attach an IAM Role with **AdministratorAccess** to your EC2 instance:

```
EC2 → Actions → Security → Modify IAM Role
```

#### Option B: Access Keys

```bash
aws configure
# Enter Access Key, Secret Key, and default region (e.g., us-east-1)
```

---

### 3. Create the EKS Cluster

```bash
eksctl create cluster \
  --name spin-cluster \
  --region us-east-1 \
  --nodegroup-name standard-nodes \
  --node-type t3.xlarge \
  --nodes 3 \
  --with-oidc \
  --managed
```

Validate cluster access:

```bash
kubectl get nodes
```

---

## 🛠️ Phase 2: Halyard & Spinnaker Deployment

Spinnaker is deployed and managed by the **Halyard daemon**.

---

### 1. Fix Permissions for Halyard

Halyard runs as the `spinnaker` user and needs access to Kubernetes and AWS credentials.

```bash
# Create directories
sudo mkdir -p /home/spinnaker/.kube /home/spinnaker/.aws

# Copy configs
sudo cp ~/.kube/config /home/spinnaker/.kube/config
sudo cp -r ~/.aws/* /home/spinnaker/.aws/

# Fix ownership
sudo chown -R spinnaker:spinnaker /home/spinnaker/.kube /home/spinnaker/.aws
sudo chmod 600 /home/spinnaker/.kube/config /home/spinnaker/.aws/credentials

# Restart Halyard
sudo service halyard restart && sleep 15
```

---

### 2. Configure Kubernetes Provider

```bash
hal config provider kubernetes enable

hal config provider kubernetes account add my-k8s-account \
  --context $(kubectl config current-context) \
  --kubeconfig-file /home/spinnaker/.kube/config

hal config deploy edit \
  --type distributed \
  --account-name my-k8s-account
```

---

### 3. Deploy Spinnaker

```bash
hal config version edit --version 2025.0.1
hal deploy apply
```

Monitor deployment:

```bash
kubectl get pods -n spinnaker
```

---

## 🌐 Phase 3: Accessing Spinnaker

Choose **ONE** access method.

---

## Option A: NodePort (Easiest / Free)

### 1. Change Service Types

```bash
kubectl patch svc spin-deck -n spinnaker -p '{"spec": {"type": "NodePort"}}'
kubectl patch svc spin-gate -n spinnaker -p '{"spec": {"type": "NodePort"}}'
```

### 2. Get NodePorts and Public IP

```bash
kubectl get svc -n spinnaker

aws ec2 describe-instances \
  --filters "Name=tag:kubernetes.io/cluster/spin-cluster,Values=owned" \
  --query "Reservations[*].Instances[*].PublicIpAddress" \
  --output text
```

### 3. Update AWS Security Group (CRITICAL)

Add inbound rules:

| Service | Port  | Source    |
| ------- | ----- | --------- |
| Deck    | 32410 | 0.0.0.0/0 |
| Gate    | 30541 | 0.0.0.0/0 |

---

### 4. Update Halyard URLs

```bash
export NODE_IP=<NODE_PUBLIC_IP>

hal config security ui edit \
  --override-base-url http://$NODE_IP:32410

hal config security api edit \
  --override-base-url http://$NODE_IP:30541

hal config security api edit \
  --cors-access-pattern http://$NODE_IP:32410

hal deploy apply
```

---

## Option B: ALB Ingress Controller (Production / Professional)

---

### 1. Install Cert-Manager & AWS Load Balancer Controller

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.13.3/cert-manager.yaml

# IAM Service Account
eksctl create iamserviceaccount \
  --cluster spin-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonEKSLoadBalancerControllerIAMPolicy \
  --approve
```

---

### 2. Apply Ingress Manifest

**spinnaker-ingress.yaml**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: spinnaker-ingress
  namespace: spinnaker
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: spin-deck
            port:
              number: 9000
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: spin-gate
            port:
              number: 8084
```

```bash
kubectl apply -f spinnaker-ingress.yaml
```

---

### 3. Update Halyard for ALB

```bash
export ALB_URL=$(kubectl get ingress spinnaker-ingress -n spinnaker \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

hal config security ui edit --override-base-url http://$ALB_URL
hal config security api edit --override-base-url http://$ALB_URL/api/v1

hal deploy apply
```
Option C: Classic/Network LoadBalancer (Balanced)
This creates a dedicated AWS Load Balancer for both the UI and the API. It is more stable than NodePort but simpler than an Ingress Controller.

1. Change Service Types to LoadBalancer

Bash

kubectl patch svc spin-deck -n spinnaker -p '{"spec": {"type": "LoadBalancer"}}'
kubectl patch svc spin-gate -n spinnaker -p '{"spec": {"type": "LoadBalancer"}}'
2. Retrieve Load Balancer DNS Names Wait a few minutes for AWS to provision the ELBs, then run:

Bash

export DECK_URL=$(kubectl get svc spin-deck -n spinnaker -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
export GATE_URL=$(kubectl get svc spin-gate -n spinnaker -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
3. Update Halyard

Bash

hal config security ui edit --override-base-url http://$DECK_URL:9000
hal config security api edit --override-base-url http://$GATE_URL:8084
hal config security api edit --cors-access-pattern http://$DECK_URL:9000

hal deploy apply
---

## 🔍 Phase 4: Troubleshooting Common Issues

| Issue                       | Cause                  | Fix                                         |
| --------------------------- | ---------------------- | ------------------------------------------- |
| Error fetching applications | CORS mismatch          | Ensure `cors-access-pattern` matches UI URL |
| 404 on API calls            | Ingress path mismatch  | Ensure `/api/v1` matches Halyard config     |
| UI spins forever            | Security Group blocked | Open Gate port in EC2 SG                    |
| Halyard cannot connect      | Permission issue       | Re-run `chown -R spinnaker:spinnaker`       |

---
1. Restarting the Halyard Daemon
If you encounter Failed to connect to localhost/127.0.0.1:8064 after a reboot or crash, restart the daemon.

If running as a system service:

Bash

sudo systemctl restart halyard
If running manually (or if the service fails):

Bash

# Kill any hung processes
pkill -f halyard

# Start in the background
hal &

## ✅ Final Notes

* NodePort is ideal for **learning and demos**
* ALB Ingress is recommended for **production environments**
* Always verify Security Groups and IAM permissions

---

🎯 **You now have a fully functional Spinnaker deployment on Amazon EKS!**


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
## 🛠️ Phase 2: Java & Halyard Installation (MANDATORY)

Spinnaker is deployed and managed by the **Halyard daemon**, which **requires Java 17**.

---

### 1. Install Java 17

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
```

Set Java 17 as default:

```bash
sudo update-alternatives --config java
```

Verify:

```bash
java -version
```

Expected output:

```
openjdk version "17"
```

---

### 2. Install Halyard

```bash
curl -fsSL https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh | sudo bash
```

This will:

* Create the `spinnaker` user
* Install the `hal` CLI
* Create and start the `halyard` systemd service

Verify installation:

```bash
which hal
sudo systemctl status halyard
id spinnaker
```

---
## 🛠️ Phase 3: Halyard & Spinnaker Configuration
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
### 🛠️ Fix: Halyard Cannot Write to `/home/spinnaker/.hal/config`

If you see errors like:

```
ERROR Failure writing your halconfig to path "/home/spinnaker/.hal/config"
Status: 409
```

Follow **these exact steps**:

```bash
# Stop halyard
sudo systemctl stop halyard

# Ensure .hal directory exists and fix ownership
sudo mkdir -p /home/spinnaker/.hal
sudo chown -R spinnaker:spinnaker /home/spinnaker
sudo chmod -R 755 /home/spinnaker

# Verify ownership
ls -ld /home/spinnaker/.hal
```

Expected owner:

```
spinnaker spinnaker
```

Restart Halyard:

```bash
sudo systemctl start halyard
sleep 10
sudo systemctl status halyard
```

Validation check:

```bash
sudo -u spinnaker touch /home/spinnaker/.hal/test
```

### 2. Configure Kubernetes Provider (Distributed Deployment)

```bash
hal config provider kubernetes enable

hal config provider kubernetes account add my-k8s-account \
  --context $(kubectl config current-context) \
  --kubeconfig-file /home/spinnaker/.kube/config

hal config deploy edit \
  --type distributed \
  --account-name my-k8s-account
```

> ⚠️ **Important**: `distributed` is mandatory for EKS/Kubernetes deployments. Do **not** use `localdebian` on EKS.

---
Step 1: Create an S3 Bucket for Spinnaker

Create a globally unique bucket:

aws s3 mb s3://spin-cluster-spinnaker-data --region us-east-1


Verify:

aws s3 ls | grep spin-cluster-spinnaker-data

Step 3: Configure S3 Storage in Halyard

Run as the ubuntu user (not root):

hal config storage s3 edit \
  --bucket spin-cluster-spinnaker-data \
  --region us-east-1


Enable S3 as the storage backend:

hal config storage edit --type s3


Verify configuration:

hal config storage s3 


Expected output should show:

bucket: spin-cluster-spinnaker-data
region: us-east-1


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

Phase 3: ALB Ingress Setup
Install cert-manager:

bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.13.3/cert-manager.yaml
Create IAM Policy (skip if already exists):

bash
curl -o iam_policy.json \
https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.6.2/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
Create IAM Role with Trust Policy  
Use your cluster’s OIDC provider ID (58BEEF23A3B700AA3C2D19DCB86F4D57):

trust-policy.json:

json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::574621078554:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/58BEEF23A3B700AA3C2D19DCB86F4D57"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/58BEEF23A3B700AA3C2D19DCB86F4D57:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
        }
      }
    }
  ]
}
Create role:

bash
aws iam create-role \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --assume-role-policy-document file://trust-policy.json
Attach policy:

bash
aws iam attach-role-policy \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --policy-arn arn:aws:iam::574621078554:policy/AWSLoadBalancerControllerIAMPolicy
Annotate Service Account:

bash
kubectl annotate serviceaccount aws-load-balancer-controller \
  -n kube-system eks.amazonaws.com/role-arn=arn:aws:iam::574621078554:role/AmazonEKSLoadBalancerControllerRole --overwrite
Restart ALB Controller:

bash
kubectl rollout restart deployment aws-load-balancer-controller -n kube-system
Phase 4: Ingress Resource
Create IngressClass:

yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: alb
spec:
  controller: ingress.k8s.aws/alb
bash
kubectl apply -f alb-ingressclass.yaml
Apply Spinnaker ingress:

yaml
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
bash
kubectl apply -f spinnaker-ingress.yaml
Verify ALB DNS:

bash
kubectl describe ingress spinnaker-ingress -n spinnaker
Phase 5: Update Spinnaker URLs
Export ALB URL:

bash
export ALB_URL=$(kubectl get ingress spinnaker-ingress -n spinnaker \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
Update Halyard:

bash
hal config security ui edit --override-base-url http://$ALB_URL
hal config security api edit --override-base-url http://$ALB_URL/api/v1
Redeploy:

bash
hal deploy apply
Phase 6: Access Spinnaker
Browser: http://<ALB-DNS-NAME>

API: http://<ALB-DNS-NAME>/api/v1
---

## Option C: Classic / Network LoadBalancer (Balanced)

Creates a dedicated AWS Load Balancer for both the UI and the API.

### 1. Change Service Types to LoadBalancer

```bash
kubectl patch svc spin-deck -n spinnaker -p '{"spec": {"type": "LoadBalancer"}}'
kubectl patch svc spin-gate -n spinnaker -p '{"spec": {"type": "LoadBalancer"}}'
```

### 2. Retrieve Load Balancer DNS Names

```bash
export DECK_URL=$(kubectl get svc spin-deck -n spinnaker -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
export GATE_URL=$(kubectl get svc spin-gate -n spinnaker -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```

### 3. Update Halyard

```bash
hal config security ui edit --override-base-url http://$DECK_URL:9000
hal config security api edit --override-base-url http://$GATE_URL:8084
hal config security api edit --cors-access-pattern http://$DECK_URL:9000

hal deploy apply
```

---

## 🔍 Phase 4: Troubleshooting Common Issues

| Issue                       | Cause                  | Fix                                         |
| --------------------------- | ---------------------- | ------------------------------------------- |
| Error fetching applications | CORS mismatch          | Ensure `cors-access-pattern` matches UI URL |
| 404 on API calls            | Ingress path mismatch  | Ensure `/api/v1` matches Halyard config     |
| UI spins forever            | Security Group blocked | Open Gate port in EC2 SG                    |
| Halyard cannot connect      | Permission issue       | Re-run `chown -R spinnaker:spinnaker`       |

---

### Restarting the Halyard Daemon

If you encounter `Failed to connect to localhost/127.0.0.1:8064`:

```bash
sudo systemctl restart halyard
```

If running manually:

```bash
pkill -f halyard
hal &
```

---

## ✅ Final Notes

* NodePort is ideal for **learning and demos**
* ALB Ingress is recommended for **production environments**
* Always verify **Security Groups** and **IAM permissions**

---

🎯 **You now have a fully functional Spinnaker deployment on Amazon EKS!**

# 🚀 Spinnaker on Amazon EKS – Step-by-Step Deployment Guide

This document provides a **complete, production-ready guide** to install and access **Spinnaker on Amazon EKS** using **Halyard**. It is structured into clear phases for ease of understanding and troubleshooting.

---

## 🏗️ Phase 1: Cluster & CLI Setup

This phase prepares your management machine and provisions the EKS cluster.

---

### 1. Install Required Tools

```bash
# Update system and install dependencies
sudo apt update && sudo apt install -y docker.io unzip curl gnupg lsb-release

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

## 🛠️ Phase 2: Java & Halyard Installation (MANDATORY)

Spinnaker is deployed and managed by the **Halyard daemon**, which **requires Java 17**.

---

### 1. Install Java 17

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
```

Set Java 17 as default:

```bash
sudo update-alternatives --config java
```

Verify:

```bash
java -version
```

Expected output:

```
openjdk version "17"
```

---

### 2. Install Halyard

```bash
curl -fsSL https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh | sudo bash
```

This will:

* Create the `spinnaker` user
* Install the `hal` CLI
* Create and start the `halyard` systemd service

Verify installation:

```bash
which hal
sudo systemctl status halyard
id spinnaker
```

---

## 🛠️ Phase 3: Halyard & Spinnaker Configuration

---

### 1. Fix Permissions for Halyard

Halyard runs as the `spinnaker` user and needs access to Kubernetes and AWS credentials.

```bash
sudo mkdir -p /home/spinnaker/.kube /home/spinnaker/.aws

sudo cp ~/.kube/config /home/spinnaker/.kube/config
sudo cp -r ~/.aws/* /home/spinnaker/.aws/

sudo chown -R spinnaker:spinnaker /home/spinnaker/.kube /home/spinnaker/.aws
sudo chmod 600 /home/spinnaker/.kube/config /home/spinnaker/.aws/credentials

sudo systemctl restart halyard && sleep 15
```

---

### 2. Configure Kubernetes Provider (Distributed Deployment)

```bash
hal config provider kubernetes enable

hal config provider kubernetes account add my-k8s-account \
  --context $(kubectl config current-context) \
  --kubeconfig-file /home/spinnaker/.kube/config

hal config deploy edit \
  --type distributed \
  --account-name my-k8s-account
```

> ⚠️ **Important**: `distributed` is mandatory for EKS/Kubernetes deployments. Do **not** use `localdebian`.

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

## 🌐 Phase 4: Accessing Spinnaker

(Access methods remain unchanged: NodePort, ALB Ingress, or LoadBalancer)

---

## 🔍 Phase 5: Troubleshooting Common Issues

| Issue                                                                      | Cause                          | Fix                                        |
| -------------------------------------------------------------------------- | ------------------------------ | ------------------------------------------ |
| `UnsupportedClassVersionError`                                             | Java 11 installed              | Install Java 17 and set it as default      |
| `hal: command not found`                                                   | Halyard not installed          | Install Halyard using the official script  |
| `chown: invalid user spinnaker`                                            | Halyard not installed          | Install Halyard (creates spinnaker user)   |
| `ERROR Failure writing your halconfig to path /home/spinnaker/.hal/config` | `.hal` directory owned by root | Fix ownership of `/home/spinnaker/.hal`    |
| HTTP 409 while enabling Kubernetes provider                                | Halyard cannot write config    | Fix `.hal` permissions and restart Halyard |
| UI spins forever                                                           | Security Group / CORS issue    | Open Gate port & fix CORS                  |

---

### 🛠️ Fix: Halyard Cannot Write to `/home/spinnaker/.hal/config`

If you see errors like:

```
ERROR Failure writing your halconfig to path "/home/spinnaker/.hal/config"
Status: 409
```

Follow **these exact steps**:

```bash
# Stop halyard
sudo systemctl stop halyard

# Ensure .hal directory exists and fix ownership
sudo mkdir -p /home/spinnaker/.hal
sudo chown -R spinnaker:spinnaker /home/spinnaker
sudo chmod -R 755 /home/spinnaker

# Verify ownership
ls -ld /home/spinnaker/.hal
```

Expected owner:

```
spinnaker spinnaker
```

Restart Halyard:

```bash
sudo systemctl start halyard
sleep 10
sudo systemctl status halyard
```

Validation check:

```bash
sudo -u spinnaker touch /home/spinnaker/.hal/test
```

If this succeeds, retry:

```bash
hal config provider kubernetes enable
```

---

🎯 **You now have a complete, correct, and production-ready Spinnaker on EKS guide.**



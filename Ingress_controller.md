# Integrating EKS with AWS Load Balancer Controller (ALB Ingress)

## Purpose of This Document

This guide explains how to integrate an Amazon EKS cluster with AWS Ingress (ALB) using the AWS Load Balancer Controller, including:

* What is happening at each step
* Why each component is required
* Core concepts like OIDC, IRSA, Ingress, and ALB
* Real-world and interview-oriented explanations

This setup reflects production-grade EKS architecture.

## High-Level Architecture

```
User
 ↓
AWS Application Load Balancer (ALB)
 ↓
Target Group
 ↓
EKS Worker Node (ENI)
 ↓
Kubernetes Service
 ↓
Pod
```

## What We Are Doing (High Level)

```
EKS Cluster
   ↓
Install AWS Load Balancer Controller
   ↓
Create Kubernetes Ingress
   ↓
AWS ALB auto-created
   ↓
Traffic → Service → Pods
```

## PART 1 Prerequisites

**Required Setup:**

* An EKS cluster running
* kubectl configured with the cluster
* eksctl installed
* IAM user/role with admin or EKS admin permissions
* Helm installed

Verify cluster access:

```bash
kubectl get nodes
```

If nodes are Ready, the cluster is reachable.

## PART 1 IAM Policy for AWS Load Balancer Controller

**Why is this needed?**

The AWS Load Balancer Controller talks directly to AWS APIs to:

* Create ALBs
* Create target groups
* Attach security groups
* Register/deregister targets

So it must have IAM permissions.

**Step 1: Download Controller IAM Policy**

```bash
curl -o iam_policy.json \
https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

**Step 2: Create IAM Policy**

```bash
aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://iam_policy.json
```

> Save the Policy ARN — required in the next step.

## PART 2 IRSA (IAM Roles for Service Accounts)

**What is IRSA?**

* IRSA allows a Kubernetes ServiceAccount to assume an IAM Role securely.
* No static credentials
* No node IAM abuse
* Least-privilege security

**What is OIDC?**

* EKS exposes an OIDC identity provider
* Kubernetes ServiceAccounts authenticate via JWT
* AWS trusts this identity using OIDC

> This enables secure pod-to-AWS access.

**Step 3: Associate OIDC Provider with Cluster**

```bash
eksctl utils associate-iam-oidc-provider \
--region ap-south-1 \
--cluster demo-eks \
--approve
```

**Step 4: Create IAM Service Account**

```bash
eksctl create iamserviceaccount \
--cluster demo-eks \
--namespace kube-system \
--name aws-load-balancer-controller \
--attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
--approve
```

> Creates: IAM Role, Kubernetes ServiceAccount, and Trust relationship using OIDC

## PART 3 Install AWS Load Balancer Controller

**Step 5: Add Helm Repository**

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

**Step 6: Install Controller via Helm**

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
-n kube-system \
--set clusterName=demo-eks \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller \
--set region=ap-south-1 \
--set v=2
```

> Service account creation is disabled because IRSA already created it.

**Step 7: Verify Controller**

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get pods -n kube-system | grep load-balancer
```

> Pods should be Running

## PART 4 Deploy Sample Application

**Step 8: Create NGINX Deployment**

```bash
kubectl create deployment nginx --image=nginx
```

**Step 9: Expose Deployment as ClusterIP Service**

```bash
kubectl expose deployment nginx \
--port=80 \
--target-port=80 \
--type=ClusterIP
```

> Ingress always routes to Services, not Pods.

## PART 5 Create Ingress Resource (ALB)

**Step 10: Ingress YAML**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```

Apply:

```bash
kubectl apply -f ingress.yaml
```

## PART 6 Verify ALB Creation

**Step 11: Check Ingress Status**

```bash
kubectl get ingress
```

> You will see: `ADDRESS: xxxx.ap-south-1.elb.amazonaws.com`

**Step 12: Verify in AWS Console**

* Go to EC2 → Load Balancers
* ALB is created automatically
* Target group registers Pod IPs

**Step 13: Test in Browser**

```text
http://<ALB-DNS-NAME>
```

> You should see NGINX Welcome Page

## PART 7 Full Traffic Flow (Interview Gold)

```
User
 ↓
AWS ALB
 ↓
Target Group
 ↓
Node ENI
 ↓
Pod IP
```

## PART 8 Common Mistakes

* Forgetting to associate OIDC
* Using Node IAM instead of IRSA
* Wrong ingress class (alb)
* Using NodePort service
* Controller not running

## PART 9 Interview Short Answer

> To integrate EKS with AWS Ingress, we install the AWS Load Balancer Controller using IRSA. The controller watches Kubernetes Ingress resources and automatically provisions AWS ALB to route external traffic to Kubernetes services and pods securely.

## Key Concepts Covered

* EKS Architecture
* Ingress vs Service
* AWS Load Balancer Controller
* OIDC
* IRSA
* ALB traffic flow
* Production best practices

## Conclusion

In this project, we integrated an Amazon EKS cluster with the AWS Load Balancer Controller to expose Kubernetes applications using an Application Load Balancer (ALB). By leveraging OIDC and IAM Roles for Service Accounts (IRSA), we enabled secure and least-privilege access between Kubernetes and AWS services.

This setup demonstrates a production-ready EKS ingress architecture, providing scalable, highly available, and secure traffic routing using native AWS services. It reflects real-world EKS networking practices and is highly relevant for DevOps learning and interviews.

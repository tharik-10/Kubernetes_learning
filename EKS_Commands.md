# EKS CLI & kubectl Cheatsheet

## 1. AWS CLI – EKS Cluster Management

**List EKS clusters**

```bash
aws eks list-clusters --region ap-south-1
```

**Describe an EKS cluster**

```bash
aws eks describe-cluster \
  --name demo-eks \
  --region ap-south-1
```

**Create kubeconfig for EKS**

```bash
aws eks update-kubeconfig \
  --name demo-eks \
  --region ap-south-1
```

**With profile:**

```bash
aws eks update-kubeconfig \
  --name demo-eks \
  --region ap-south-1 \
  --profile admin
```

**Delete EKS cluster**

```bash
aws eks delete-cluster \
  --name demo-eks \
  --region ap-south-1
```

## 2. kubectl – EKS Cluster Access

**Check cluster connection**

```bash
kubectl get nodes
```

**Check cluster info**

```bash
kubectl cluster-info
```

**Get all namespaces**

```bash
kubectl get ns
```

**Get pods (all namespaces)**

```bash
kubectl get pods -A
```

## 3. aws-auth ConfigMap (RBAC)

**View aws-auth**

```bash
kubectl get configmap aws-auth -n kube-system -o yaml
```

**Edit aws-auth**

```bash
kubectl edit configmap aws-auth -n kube-system
```

**Example aws-auth:**

```yaml
mapUsers: |
  - userarn: arn:aws:iam::ACCOUNT_ID:user/admin
    username: admin
    groups:
      - system:masters

  - userarn: arn:aws:iam::ACCOUNT_ID:user/dev
    username: dev
    groups:
      - dev-group
```

## 4. IAM Identity Check (Very Important)

**Who am I (AWS identity)?**

```bash
aws sts get-caller-identity
```

**Check current kubectl user**

```bash
kubectl config current-context
```

**List kubeconfig contexts**

```bash
kubectl config get-contexts
```

**Switch context**

```bash
kubectl config use-context <context-name>
```

## 5. Node Groups (Managed)

**List node groups**

```bash
aws eks list-nodegroups \
  --cluster-name demo-eks \
  --region ap-south-1
```

**Describe node group**

```bash
aws eks describe-nodegroup \
  --cluster-name demo-eks \
  --nodegroup-name demo-ng \
  --region ap-south-1
```

**Scale node group**

```bash
aws eks update-nodegroup-config \
  --cluster-name spin-cluster \
  --nodegroup-name standard-nodes \
  --scaling-config minSize=0,maxSize=3,desiredSize=0 \
  --region us-east-1
```

## 6. Kubernetes RBAC Testing (Admin vs Dev)

**Test permissions**

```bash
kubectl auth can-i get pods
kubectl auth can-i create deployments
kubectl auth can-i delete nodes
```

**With namespace:**

```bash
kubectl auth can-i create pods -n dev
```

## 7. Networking & CNI (EKS specific)

**Check AWS VPC CNI**

```bash
kubectl get pods -n kube-system | grep aws-node
```

**CNI logs**

```bash
kubectl logs -n kube-system -l k8s-app=aws-node
```

**Check ENIs attached to nodes**

```bash
kubectl describe node <node-name>
```

## 8. Cluster Add-ons

**List add-ons**

```bash
aws eks list-addons \
  --cluster-name demo-eks \
  --region ap-south-1
```

**Install add-on**

```bash
aws eks create-addon \
  --cluster-name demo-eks \
  --addon-name vpc-cni \
  --region ap-south-1
```

## 9. Troubleshooting Commands

**Check events**

```bash
kubectl get events -A
```

**Describe pod**

```bash
kubectl describe pod <pod-name>
```

**Check API server health**

```bash
kubectl get --raw='/healthz'
```

## 10. Interview Gold Commands 

```bash
aws eks update-kubeconfig
kubectl get nodes
kubectl get pods -A
kubectl auth can-i *
aws sts get-caller-identity
kubectl get configmap aws-auth -n kube-system
```

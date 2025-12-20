# EKS Cluster Setup with RBAC (Admin & Dev Roles)

This repository demonstrates how to:

✅ Create an Amazon EKS cluster using `eksctl`  
✅ Configure Kubernetes RBAC using IAM users  
✅ Set up **Admin** and **Dev** access  
✅ Validate permissions practically (Allowed ✅ / Forbidden ❌)

This setup matches **real DevOps / Kubernetes interview and assignment expectations**.

---

## 🧰 Prerequisites

- AWS Account
- IAM User with `AdministratorAccess`
- Installed CLI tools:
  ```bash
  aws --version
  kubectl version --client
  eksctl version
  ```

### Install eksctl (if not installed)
```bash
curl --silent --location https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

---

## PART 1: AWS CONFIGURATION

### Step 1: Configure AWS CLI
```bash
aws configure
```

Provide:
- AWS Access Key
- AWS Secret Key
- Region: `ap-south-1`
- Output format: `json`

---

## PART 2: CREATE EKS CLUSTER

### Step 2: Create Cluster using eksctl
```bash
eksctl create cluster --name demo-eks --region ap-south-1 --nodegroup-name demo-nodes --node-type t3.medium --nodes 2 --nodes-min 1 --nodes-max 3 --managed
```

⏳ Takes around **15 minutes**

---

### Step 3: Verify Cluster
```bash
aws eks update-kubeconfig --region ap-south-1 --name demo-eks
kubectl get nodes
```

✅ Nodes should be in `Ready` state.

---

## PART 3: CREATE IAM USERS

### Admin User
- Name: `eks-admin`
- Access: Programmatic
- Policy: `AdministratorAccess`

### Dev User
- Name: `eks-dev`
- Access: Programmatic
- Policy: `AmazonEKSClusterPolicy`

⚠️ Save **Access Key & Secret Key** for both users.

---

## PART 4: MAP IAM USERS TO EKS (aws-auth)

### Step 4: Edit aws-auth ConfigMap
```bash
kubectl edit configmap aws-auth -n kube-system
```

Add the following:

```yaml
mapUsers: |
  - userarn: arn:aws:iam::<ACCOUNT_ID>:user/eks-admin
    username: eks-admin
    groups:
      - system:masters

  - userarn: arn:aws:iam::<ACCOUNT_ID>:user/eks-dev
    username: eks-dev
    groups:
      - dev-group
```

### Explanation
- `system:masters` → Full admin access
- `dev-group` → Custom RBAC group

---

## PART 5: ADMIN RBAC (No Extra YAML Needed)

Admin user is mapped to `system:masters`, which is bound to the `cluster-admin` role.

Admin can:
- Manage nodes
- Create/delete namespaces
- Manage RBAC
- Access all namespaces

---

## PART 6: DEV RBAC SETUP

### Dev Requirements
- ✅ Create Pods
- ✅ List Pods & Services
- ❌ No namespace deletion
- ❌ No cluster-wide access

---

### Step 5: Create Namespace
```bash
kubectl create namespace dev
```

---

### Step 6: Create Role (Namespace Scoped)

`rbac/dev-role.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-role
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch", "create"]
```

Apply:
```bash
kubectl apply -f rbac/dev-role.yaml
```

---

### Step 7: Bind Role to Dev Group

`rbac/dev-rolebinding.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-rolebinding
  namespace: dev
subjects:
- kind: Group
  name: dev-group
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
```

Apply:
```bash
kubectl apply -f rbac/dev-rolebinding.yaml
```

---

## PART 7: TEST DEV ACCESS

### Configure Dev AWS Profile
```bash
aws configure --profile eks-dev
```

### Update kubeconfig
```bash
aws eks update-kubeconfig --name demo-eks --region ap-south-1 --profile eks-dev
```

### Test Dev Permissions
```bash
kubectl get pods -n dev                    # ✅ Allowed
kubectl create deployment nginx --image=nginx -n dev  # ✅ Allowed
kubectl get nodes                          # ❌ Forbidden
kubectl delete namespace dev               # ❌ Forbidden
```

---

## PART 8: TEST ADMIN ACCESS

### Configure Admin AWS Profile
```bash
aws configure --profile eks-admin
```

### Update kubeconfig
```bash
aws eks update-kubeconfig --name demo-eks --region ap-south-1 --profile eks-admin
```

### Test Admin Permissions
```bash
kubectl get nodes                          # ✅ Allowed
kubectl get namespaces                    # ✅ Allowed
kubectl create namespace test-admin       # ✅ Allowed
kubectl delete namespace test-admin       # ✅ Allowed
kubectl get pods -A                       # ✅ Allowed
kubectl get clusterroles                  # ✅ Allowed
```

---

## RBAC VERIFICATION SUMMARY

| Action | Admin | Dev |
|------|------|------|
| Get nodes | ✅ | ❌ |
| Create namespace | ✅ | ❌ |
| Delete namespace | ✅ | ❌ |
| Create pods in dev | ✅ | ✅ |
| Access kube-system | ✅ | ❌ |
| Manage RBAC | ✅ | ❌ |

---

## Repository Structure

```text
eks-rbac-admin-dev/
├── README.md
├── rbac/
│   ├── dev-role.yaml
│   └── dev-rolebinding.yaml
```

---

## Cleanup (Optional)
```bash
eksctl delete cluster --name demo-eks --region ap-south-1
```

---

## Interview Ready Explanation

> In EKS, IAM users are mapped to Kubernetes RBAC using the aws-auth ConfigMap. Admin access is granted via the system:masters group, while fine-grained access for developers is controlled using Roles and RoleBindings scoped to namespaces.

---

⭐ If this repo helped you, feel free to star it!

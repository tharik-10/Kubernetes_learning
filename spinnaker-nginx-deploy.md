# 🚀 Spinnaker NGINX Deployment: The Ultimate Guide

This guide provides **two complete methods** to deploy **NGINX on Kubernetes using Spinnaker**:

* **Method 1:** Simple text-based manifest (quick testing)
* **Method 2:** Production-grade GitOps flow using **Kustomize + Bake stage**

---

## 🛠️ Phase 1: Infrastructure Permissions (RBAC)

Run the following steps on your **Ubuntu server terminal** to avoid `Forbidden` errors in Spinnaker.

### 1. Create RBAC File

```bash
nano spinnaker-rbac.yaml
```

### 2. Paste the RBAC Configuration

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: spinnaker-admin
subjects:
  - kind: User
    name: "system:node:ip-192-168-48-156.ec2.internal"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### 3. Apply RBAC

```bash
kubectl apply -f spinnaker-rbac.yaml
```

---

## 📗 Method 1: Simple Deployment (Text Manifest)

**Best for:** Quick testing without GitHub

---

### 1️⃣ UI Steps (Method 1)

1. **Create Application**

   * Click **Actions → Create Application**
   * Name: `nginx-simple`

2. **Create Pipeline**

   * Name: `Deploy-Text`

3. **Add Stage → Deploy (Manifest)**

   * Account: `my-k8s-v2-account`
   * Manifest Source: `Text`
   * Paste YAML (below)

4. **Moniker**

   * App: `nginx-simple`

---

### 2️⃣ Text Manifest YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-simple-text
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
        image: nginx:latest
```

---

### 3️⃣ Pipeline JSON (Method 1)

Use **Pipeline Actions → Edit as JSON** if UI fails.

```json
{
  "stages": [
    {
      "account": "my-k8s-v2-account",
      "cloudProvider": "kubernetes",
      "manifestSource": "text",
      "manifests": [
        {
          "apiVersion": "apps/v1",
          "kind": "Deployment",
          "metadata": { "name": "nginx-simple-text" },
          "spec": {
            "replicas": 1,
            "selector": { "matchLabels": { "app": "nginx" } },
            "template": {
              "metadata": { "labels": { "app": "nginx" } },
              "spec": {
                "containers": [
                  { "name": "nginx", "image": "nginx:latest" }
                ]
              }
            }
          }
        }
      ],
      "moniker": { "app": "nginx-simple" },
      "name": "Deploy (Manifest)",
      "type": "deployManifest"
    }
  ]
}
```

---

## 📘 Method 2: Advanced Deployment (Kustomize + GitOps)

**Best for:** Production-grade deployments

---

## 1️⃣ GitHub Repository Structure

Repository: **tharik-10/Kubernetes_learning**

```
my-nginx-app/
└── base/
    ├── deployment.yaml
    ├── service.yaml
    └── kustomization.yaml
```

---

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-simple
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:latest
        name: nginx
```

---

### service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-lb
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
```

---

### kustomization.yaml

```yaml
resources:
  - deployment.yaml
  - service.yaml
```

---

## 2️⃣ Halyard Configuration

Ensure GitHub artifact account appears in Spinnaker.

```bash
hal config artifact gitrepo enable
hal config artifact gitrepo account add tharik-10 \
  --token-file /home/ubuntu/git-token.txt
hal deploy apply
```

---

## 3️⃣ UI Pipeline Configuration (Method 2)

### Stage 1: Bake (Manifest)

* Stage Type: `Bake (Manifest)`
* Render Engine: `KUSTOMIZE`
* Expected Artifact: `tharik-10`
* Repo URL: `https://github.com/tharik-10/Kubernetes_learning`
* Branch: `main`
* Kustomize Path: `my-nginx-app/base/kustomization.yaml`

**Produces Artifact:**

* Name: `baked-output`
* Type: `embedded/base64`

---

### Stage 2: Deploy (Manifest)

* Manifest Source: `Artifact`
* Manifest Artifact: `baked-output`
* Artifact Account: `embedded-artifact`
* Moniker App: `nginx-service`

---

## 4️⃣ Pipeline JSON (Method 2)

```json
{
  "stages": [
    {
      "name": "Bake (Manifest)",
      "type": "bakeManifest",
      "refId": "1",
      "templateRenderer": "KUSTOMIZE",
      "kustomizeFilePath": "my-nginx-app/base/kustomization.yaml",
      "inputArtifact": {
        "account": "tharik-10",
        "artifact": {
          "artifactAccount": "tharik-10",
          "name": "https://github.com/tharik-10/Kubernetes_learning",
          "reference": "https://github.com/tharik-10/Kubernetes_learning",
          "type": "git/repo",
          "version": "main"
        }
      },
      "expectedArtifacts": [
        {
          "displayName": "baked-output",
          "id": "fixed-id-123",
          "matchArtifact": {
            "name": "baked-output",
            "type": "embedded/base64"
          }
        }
      ]
    },
    {
      "name": "Deploy (Manifest)",
      "type": "deployManifest",
      "refId": "2",
      "requisiteStageRefIds": ["1"],
      "account": "my-k8s-v2-account",
      "cloudProvider": "kubernetes",
      "manifestArtifactAccount": "embedded-artifact",
      "manifestArtifactId": "fixed-id-123",
      "manifestSource": "artifact",
      "source": "artifact",
      "moniker": {
        "app": "nginx-service",
        "cluster": "nginx-kustomize"
      },
      "manifestArtifact": {
        "artifactAccount": "embedded-artifact",
        "name": "baked-output",
        "type": "embedded/base64"
      }
    }
  ]
}
```

---

## 🚀 Execution & Verification

1. **Save Changes**
2. Click **Start Manual Execution**
3. Navigate to **Infrastructure → Clusters**
4. Locate **nginx-service-lb**
5. Access NGINX using the **External DNS/IP**

---

✅ **You now have both testing and production-grade NGINX deployments using Spinnaker!**


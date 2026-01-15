# Centralized Spinnaker Pipeline — Manual Setup Guide

This guide explains how to manually configure a centralized Spinnaker pipeline that:

* Reads `parameter.yml` from each application repo
* Parses values into pipeline variables
* Bakes a manifest with placeholders replaced by pipeline variables
* Deploys the baked manifest to Kubernetes
* Optionally uses a Git trigger to start the pipeline automatically

---

## 1. Repository Setup

### 1.1 Application Repository Structure

Each **application repository** (for example: `nginx-app`) should follow this structure:

```text
nginx-app/
├── parameter.yml
├── README.md
└── src/
    └── (application source code)
```

### `parameter.yml`

Each application repository must contain a `parameter.yml` file:

```yaml
appName: nginx-app
namespace: default
replicas: 2
image: nginx:latest

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"

enableProbes: true
```

---

## 2. Central Pipeline Repository Structure

The **centralized Spinnaker pipeline repository** (for example: `spinnaker-central-pipeline`) should follow this structure:

```text
spinnaker-central-pipeline/
├── manifests/
│   ├── base/
│   │   ├── deployment.yaml
│   │   └── kustomization.yaml
│   └── patches/
│       └── probes.yaml
├── charts/                  # Optional (if using Helm)
│   └── nginx/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           └── deployment.yaml
├── README.md
└── pipeline/
    └── centralized-pipeline.json
```

### Base Manifest Template (with placeholders)

`manifests/base/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
spec:
  replicas: ${REPLICAS}
  selector:
    matchLabels:
      app: ${APP_NAME}
  template:
    metadata:
      labels:
        app: ${APP_NAME}
    spec:
      containers:
        - name: ${APP_NAME}
          image: ${IMAGE}
          resources:
            requests:
              cpu: ${CPU_REQUEST}
              memory: ${MEMORY_REQUEST}
            limits:
              cpu: ${CPU_LIMIT}
              memory: ${MEMORY_LIMIT}
```

### Optional Patch (conditional features)

`manifests/patches/probes.yaml`

````yaml
spec:
  template:
    spec:
      containers:
        - name: ${APP_NAME}
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 20
```yaml
spec:
  template:
    spec:
      containers:
        - name: ${APP_NAME}
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
````

---

### Service Manifest (Internet-Facing)

`manifests/base/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
spec:
  type: LoadBalancer
  selector:
    app: ${APP_NAME}
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

---

## 3. Correct Pipeline Workflow (High-Level)

1. **Git Trigger** starts the pipeline on commit
2. **Webhook Stage** fetches `parameter.yml`
3. **Evaluate Variables Stage** parses YAML into pipeline variables
4. **Bake Manifest Stage** substitutes variables into base manifests
5. **Deploy Manifest Stage** deploys to Kubernetes

---

## 4. Pipeline Stages (Manual Method)

### Stage 0: Git Trigger Configuration

To automate pipeline execution when changes are pushed:

* Open pipeline configuration → **Automated Triggers**
* Click **Add Trigger**
* Select **Type**: Git
* Repo Type: `github`
* Organization/User: `tharik-10`
* Project/Repo: `spinnaker-central-pipeline`
* Branch: `main`
* Trigger Enabled: ✅

This ensures the pipeline runs automatically on commits to `main`.

---

### Stage 1: Read Deployment Parameters (Webhook)

**Type**: Webhook

**Method**: GET

**URL**: Raw GitHub link to `parameter.yml`

**Wait for completion**: Enabled

```json
{
  "type": "webhook",
  "name": "Read Deployment Parameters",
  "method": "GET",
  "url": "https://raw.githubusercontent.com/<org>/<repo>/<branch>/parameter.yml",
  "waitForCompletion": true
}
```

---

### Stage 2: Evaluate Variables

**Type**: Evaluate Variables

**Goal**: Parse YAML string into pipeline variables

```text
yamlBody       = ${#stage('Read Deployment Parameters').context.webhook.body}
APP_NAME       = ${#readYaml(#stage('Read Deployment Parameters').context.webhook.body).appName}
NAMESPACE      = ${#readYaml(#stage('Read Deployment Parameters').context.webhook.body).namespace}
REPLICAS       = ${#readYaml(#stage('Read Deployment Parameters').context.webhook.body).replicas}
IMAGE          = ${#readYaml(#stage('Read Deployment Parameters').context.webhook.body).image}
CPU_REQUEST    = ${#readYaml(#stage('Read Deployment Parameters').context.webhook.body).resources.requests.cpu}
MEMORY_REQUEST = ${#readYaml(#stage('Read Deployment Parameters').context.webhook.body).resources.requests.memory}
CPU_LIMIT      = ${#readYaml(#stage('Read Deployment Parameters').context.webhook.body).resources.limits.cpu}
MEMORY_LIMIT   = ${#readYaml(#stage('Read Deployment Parameters').context.webhook.body).resources.limits.memory}
ENABLE_PROBES  = ${#readYaml(#stage('Read Deployment Parameters').context.webhook.body).enableProbes}
```

---

### Stage 3: Bake Manifest

#### Option A: Kustomize

```json
{
  "type": "bakeManifest",
  "name": "Bake (Manifest)",
  "templateRenderer": "KUSTOMIZE",
  "inputArtifacts": [
    {
      "account": "git-repo",
      "id": "base-manifest"
    }
  ],
  "kustomizeFilePath": "manifests/base/kustomization.yaml"
}
```

#### Option B: Helm

```json
{
  "type": "bakeManifest",
  "name": "Bake (Manifest)",
  "templateRenderer": "HELM3",
  "inputArtifacts": [
    {
      "account": "git-repo",
      "id": "nginx-chart"
    }
  ],
  "overrides": {
    "appName": "${APP_NAME}",
    "namespace": "${NAMESPACE}",
    "replicas": "${REPLICAS}",
    "image": "${IMAGE}",
    "resources.requests.cpu": "${CPU_REQUEST}",
    "resources.requests.memory": "${MEMORY_REQUEST}",
    "resources.limits.cpu": "${CPU_LIMIT}",
    "resources.limits.memory": "${MEMORY_LIMIT}",
    "enableProbes": "${ENABLE_PROBES}"
  }
}
```

---

### Stage 4: Deploy Manifest

```json
{
  "type": "deployManifest",
  "name": "Deploy (Manifest)",
  "account": "my-k8s-v2-account",
  "cloudProvider": "kubernetes",
  "source": "artifact",
  "manifestArtifactId": "<baked-manifest-artifact-id>"
}
```

---

## 5. How Variables Are Parsed and Used

* Webhook stage → fetches raw YAML string
* Evaluate Variables stage → parses YAML into pipeline variables
* Bake Manifest stage → replaces placeholders with pipeline variables
* Deploy Manifest stage → applies baked manifest to Kubernetes

---

## 6. Manual Method Notes

This is a manual method for study:

* You explicitly fetch YAML via Webhook
* You manually define variable mappings in Evaluate Variables
* You manually wire Bake and Deploy stages

Once validated, you can automate:

* Use Git triggers to start pipelines
* Standardize Helm/Kustomize charts
* Programmatically generate pipelines per app

---

## 7. Troubleshooting

**Null errors in Evaluate Variables**
Use `.context.webhook.body` instead of `.outputs.body`

**Getter not found errors**
Wrap string in `#readYaml(...)` before accessing fields

**Bake stage not substituting values**
Ensure placeholders match variable names and Evaluate Variables outputs are present

---

## 8. Corrected Pipeline Flow (Final)

1. **Git Trigger**
   Watches the repo (`spinnaker-central-pipeline`) and branch (`main`).

2. **Webhook Stage (Read Deployment Parameters)**
   Fetches the latest `parameter.yml` from the application repo.

3. **Evaluate Variables Stage**
   Parses the YAML string into pipeline variables using SpEL (`#readYaml(...)`).

4. **Bake Manifest Stage**
   Uses Helm or Kustomize to substitute placeholders in the base manifest with pipeline variables.

5. **Deploy Manifest Stage**
   Deploys the baked manifest to Kubernetes using the configured account.


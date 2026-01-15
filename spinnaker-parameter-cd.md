# Centralized Spinnaker Pipeline — Manual Setup Guide

This guide explains how to manually configure a centralized Spinnaker pipeline that:
- Reads `parameter.yml` from each application repo
- Parses values into pipeline variables
- Bakes a manifest with placeholders replaced by pipeline variables
- Deploys the baked manifest to Kubernetes
- Optionally uses a Git trigger to start the pipeline automatically

---

## 1. Repository Setup

### `parameter.yml`

Each application repository should contain a `parameter.yml` file:

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
Base Manifest Template (with placeholders)
yaml
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
Optional Patch (conditional features)
yaml
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
2. Pipeline Stages (Manual Method)
3. Git Trigger Configuration
To automate pipeline execution when changes are pushed:

Open pipeline configuration → Automated Triggers

Click Add Trigger

Select Type: Git

Repo Type: github

Organization/User: tharik-10

Project/Repo: spinnaker-central-pipeline

Branch: main

Trigger Enabled: ✅

Save pipeline

This ensures the pipeline runs automatically on commits to main.
Stage 1: Read Deployment Parameters (Webhook)
Type: Webhook

Method: GET

URL: Raw link to parameter.yml in GitHub

Wait for completion: Enabled

json
{
  "type": "webhook",
  "name": "Read Deployment Parameters",
  "method": "GET",
  "url": "https://raw.githubusercontent.com/<org>/<repo>/<branch>/parameter.yml",
  "waitForCompletion": true
}
Stage 2: Evaluate Variables
Type: Evaluate Variables

Goal: Parse YAML string into pipeline variables

text
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
Stage 3: Bake Manifest
Option A: Kustomize
json
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
Option B: Helm
json
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
Stage 4: Deploy Manifest
json
{
  "type": "deployManifest",
  "name": "Deploy (Manifest)",
  "account": "my-k8s-v2-account",
  "cloudProvider": "kubernetes",
  "source": "artifact",
  "manifestArtifactId": "<baked-manifest-artifact-id>"
}
4. How Variables Are Parsed and Used
Webhook stage → fetches raw YAML string

Evaluate Variables stage → parses YAML into pipeline variables

Bake Manifest stage → replaces placeholders with pipeline variables

Deploy Manifest stage → applies baked manifest to Kubernetes

5. Manual Method Notes
This is a manual method for study:

You explicitly fetch YAML via Webhook

You manually define variable mappings in Evaluate Variables

You manually wire Bake and Deploy stages

Once validated, you can automate:

Use Git triggers to start pipelines

Standardize Helm/Kustomize charts

Programmatically generate pipelines per app

6. Troubleshooting
Null errors in Evaluate Variables  
Use .context.webhook.body instead of .outputs.body

Getter not found errors  
Wrap string in #readYaml(...) before accessing fields

Bake stage not substituting values  
Ensure placeholders match variable names and Evaluate Variables outputs are present
Corrected Pipeline Flow (Git trigger first)
Git Trigger

Watches the repo (spinnaker-central-pipeline) and branch (main).

When a commit is pushed, the pipeline starts automatically.

This replaces the need to manually start the pipeline.

Webhook Stage (Read Deployment Parameters)

Fetches the latest parameter.yml from the repo.

Returns the YAML string into the pipeline context.

Evaluate Variables Stage

Parses the YAML string into pipeline variables using SpEL (#readYaml(...)).

Example: ${#readYaml(#stage('Read Deployment Parameters').context.webhook.body).appName}.

Bake Manifest Stage

Uses Helm or Kustomize to substitute placeholders in the base manifest with pipeline variables.

Optional patches (like liveness/readiness probes) can be merged if enabled.

Deploy Manifest Stage

Deploys the baked manifest to Kubernetes using the configured account.

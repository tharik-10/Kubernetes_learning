# 📘 Centralized Spinnaker Deployment using Jenkins + Kustomize (Learning Guide)

This document explains **end-to-end how a centralized deployment pipeline works** using:

* **Jenkins** (CI & trigger)
* **Spinnaker** (CD)
* **Kubernetes (EKS)**
* **Kustomize** (manifest templating)
* **Webhook-based automation**

It is written **purely for learning and reference**, based on your working setup and screenshots.

---

## 🧠 High-Level Architecture

```
Developer
   │
   │ git push / manual build
   ▼
Jenkins Pipeline
   │
   │ reads parameter.yaml
   │ converts to JSON
   ▼
Spinnaker Webhook Trigger
   │
   │ Evaluate Variables (SpEL)
   │ Bake Manifest (Kustomize)
   ▼
Kubernetes Cluster (EKS)
   │
   ├── Deployment
   └── Service (LoadBalancer)
```

---

## 📂 Repository Structure

```
spinnaker-central-pipeline/
├── Jenkinsfile
├── parameter.yaml
└── manifests/
    ├── base/
    │   ├── deployment-base.yaml
    │   ├── service-base.yaml
    │   └── kustomization.yaml
    └── overlays/
        └── default/
            └── kustomization.yaml
```

---

## 🧩 1. Base Kubernetes Manifests (Reusable Templates)

### 📄 deployment-base.yaml

This is a **generic Deployment template**. All values are injected dynamically by Spinnaker using **SpEL variables**.

**Purpose:** One deployment template for all apps.

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
        ports:
        - containerPort: ${SERVICE_PORT}
        resources:
          requests:
            cpu: ${REQ_CPU}
            memory: ${REQ_MEM}
          limits:
            cpu: ${LIM_CPU}
            memory: ${LIM_MEM}
        readinessProbe:
          httpGet:
            path: ${READINESS_PATH}
            port: ${SERVICE_PORT}
          initialDelaySeconds: ${READINESS_DELAY}
          periodSeconds: ${READINESS_PERIOD}
        livenessProbe:
          httpGet:
            path: ${LIVENESS_PATH}
            port: ${SERVICE_PORT}
          initialDelaySeconds: ${LIVENESS_DELAY}
          periodSeconds: ${LIVENESS_PERIOD}
```

✔ Supports probes
✔ Supports CPU/Memory tuning
✔ Fully parameterized

---

### 📄 service-base.yaml

Exposes the application externally using AWS LoadBalancer.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
  labels:
    app: ${APP_NAME}
spec:
  type: LoadBalancer
  selector:
    app: ${APP_NAME}
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
```

---

### 📄 manifests/base/kustomization.yaml

Defines base resources.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment-base.yaml
- service-base.yaml
```

---

## 🎯 2. Overlay Configuration (Environment Specific)

### 📄 manifests/overlays/default/kustomization.yaml

This file **binds real values** to placeholders.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
```

---

### 📄 parameter.yaml (Runtime Input)

This file is read by **Jenkins** and sent to **Spinnaker**.

```yaml
appName: nginx-app
namespace: default
replicas: 2
image:
  repository: nginx
  tag: "1.25"
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"
servicePort: 80
readinessPath: /
readinessDelay: 5
readinessPeriod: 10
livenessPath: /
livenessDelay: 15
livenessPeriod: 15
```

---

## ⚙️ 3. Jenkins Pipeline (Trigger Engine)

### 📄 Jenkinsfile

**Responsibilities:**

* Clone repo
* Read parameter.yaml
* Convert YAML → JSON
* Trigger Spinnaker webhook

```groovy
pipeline {
  agent any

  environment {
    SPINNAKER_WEBHOOK_URL = "http://<spinnaker-gate>:8084/webhooks/webhook/centralized-deploy"
  }

  stages {

    stage('Checkout Application Repo') {
      steps {
        git branch: 'main',
            url: 'https://github.com/tharik-10/spinnaker-central-pipeline.git'
      }
    }

    stage('Read parameter.yaml') {
      steps {
        script {
          def paramsYaml = readYaml file: 'parameter.yaml'
          env.PARAM_JSON = groovy.json.JsonOutput.toJson(paramsYaml)
        }
      }
    }

    stage('Trigger Spinnaker Pipeline') {
      steps {
        sh """
          curl -X POST \
            -H "Content-Type: application/json" \
            --data-binary '${env.PARAM_JSON}' \
            ${SPINNAKER_WEBHOOK_URL}
        """
      }
    }
  }
}
```

---

## 🚀 4. Spinnaker Pipeline Design

### 🔹 Automated Trigger

* **Type:** Webhook
* **Endpoint:** `/webhooks/webhook/centralized-deploy`
* **Payload:** Jenkins JSON

---

### 🔹 Stage 1: Evaluate Variables

Maps webhook payload → pipeline variables.

| Variable  | SpEL Expression                   |
| --------- | --------------------------------- |
| APP_NAME  | `${trigger.payload['appName']}`   |
| NAMESPACE | `${trigger.payload['namespace']}` |

✔ Used across bake & deploy stages

---

### 🔹 Stage 2: Bake Manifest (Kustomize)

* Renderer: **KUSTOMIZE**
* Repo: `spinnaker-central-pipeline`
* File: `manifests/base/kustomization.yaml`

📦 Output: **Fully rendered Kubernetes YAML**

---

### 🔹 Stage 3: Deploy to Kubernetes

* Account: `my-k8s-v2-account`
* Manifest Source: **Artifact**
* Artifact: **Baked Manifest**

✔ Deployment
✔ Service
✔ Probes
✔ LoadBalancer

---

## ✅ Final Outcome

* Jenkins build triggers Spinnaker
* Spinnaker bakes Kustomize manifests
* App deployed to EKS
* Service exposed via AWS ELB
* Fully reusable & scalable pipeline

---

## 🧠 Key Learning Takeaways

✔ Centralized deployment model
✔ Zero duplication of manifests
✔ One pipeline for multiple apps
✔ Clean separation of CI and CD
✔ Production-grade Kubernetes deployments

---

## 🏁 Next Enhancements (Optional)

* Canary deployments
* Prometheus metrics + analysis stage
* Manual judgment gates
* Multi-environment overlays (dev/stage/prod)

---

**Author:** Mohamed Tharik
**Purpose:** Learning & Reference Documentation

🚀 Spinnaker NGINX Deployment: The Ultimate Guide
🛠️ Phase 1: Infrastructure Permissions (RBAC)
Run this on your Ubuntu server terminal to prevent "Forbidden" errors.

Create the file: nano spinnaker-rbac.yaml

Paste this content:

YAML

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
Apply it: kubectl apply -f spinnaker-rbac.yaml

📗 Method 1: The Simple Way (Text Manifest)
Best for quick testing without using GitHub.

1. UI Steps (Method 1)
Create App: Click Actions > Create Application. Name: nginx-simple.

Create Pipeline: Name it Deploy-Text.

Add Stage: Click Add Stage and select Deploy (Manifest).

Account: Select my-k8s-v2-account.

Manifest Source: Select Text.

Text Manifest: Paste the YAML from the section below.

Moniker: Scroll to the bottom and enter nginx-simple in the App field.

2. YAML Source (Method 1)
YAML

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
3. Pipeline JSON (Method 1)
If the UI fails, go to Pipeline Actions > Edit as JSON and paste this:

JSON

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
              "spec": { "containers": [{ "name": "nginx", "image": "nginx:latest" }] }
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
📘 Method 2: The Advanced Way (Kustomize)
The production GitOps flow using GitHub and the Bake stage.
Step 1: Prepare Git Repository
Your GitHub repo (tharik-10/Kubernetes_learning) must have this exact structure:

Plaintext

my-nginx-app/
└── base/
    ├── deployment.yaml
    ├── service.yaml
    └── kustomization.yaml

1. GitHub File Structure
In your repo tharik-10/Kubernetes_learning, create the folder my-nginx-app/base/ with these files:

deployment.yaml

YAML

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
service.yaml

YAML

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
kustomization.yaml

YAML

resources:
  - deployment.yaml
  - service.yaml
2. Halyard Configuration (Terminal)
Run these to ensure the git/repo account tharik-10 appears in Spinnaker.

Bash

hal config artifact gitrepo enable
hal config artifact gitrepo account add tharik-10 --token-file /home/ubuntu/git-token.txt
hal deploy apply
3. UI Steps (Method 2)
Stage 1: Bake (Manifest)

Add Stage: Select Bake (Manifest).

Render Engine: Select KUSTOMIZE.

Expected Artifact: Select tharik-10 (Git Repo).

URL: https://github.com/tharik-10/Kubernetes_learning.

Branch: main.

Kustomize File Path: my-nginx-app/base/kustomization.yaml.

Produces Artifacts (Bottom): Add a new artifact. Name: baked-output, Type: embedded/base64.

Stage 2: Deploy (Manifest)

Add Stage: Select Deploy (Manifest).

Manifest Source: Select Artifact.

Manifest Artifact: Select baked-output.

Artifact Account: Select embedded-artifact.

Moniker: Set App to nginx-service.

4. Pipeline JSON (Method 2)
Use this JSON to fix all pathing and artifact "src is null" errors at once:

JSON

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
          "matchArtifact": { "name": "baked-output", "type": "embedded/base64" }
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
      "moniker": { "app": "nginx-service", "cluster": "nginx-kustomize" },
      "manifestArtifact": {
        "artifactAccount": "embedded-artifact",
        "name": "baked-output",
        "type": "embedded/base64"
      }
    }
  ]
}
🚀 Execution & Verification
Save Changes and click Start Manual Execution.

Go to the Infrastructure > Clusters tab.

Locate nginx-service-lb to find your External DNS/IP.

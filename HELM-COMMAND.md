Helm Commands – Complete Learning Guide
📌 What is Helm?

Helm is a package manager for Kubernetes.

It helps you:

Install applications in Kubernetes

Manage versions

Upgrade / rollback deployments

Share reusable templates (Helm Charts)

1️⃣ Helm Installation & Setup
🔹 Check Helm Version
helm version


👉 Shows installed Helm version.

🔹 Add Helm Repository
helm repo add <repo-name> <repo-url>


Example:

helm repo add bitnami https://charts.bitnami.com/bitnami


👉 Adds a Helm chart repository.

🔹 List Added Repositories
helm repo list


👉 Displays all configured repositories.

🔹 Update Repository
helm repo update


👉 Downloads the latest chart metadata.

2️⃣ Searching Charts
🔹 Search Chart in Repository
helm search repo <chart-name>


Example:

helm search repo nginx


👉 Searches available charts in added repositories.

🔹 Search in Artifact Hub
helm search hub <chart-name>


👉 Searches charts in Artifact Hub.

3️⃣ Installing Charts
🔹 Install Chart
helm install <release-name> <chart-name>


Example:

helm install my-nginx bitnami/nginx


👉 Installs the application into Kubernetes.

🔹 Install with Custom Namespace
helm install my-nginx bitnami/nginx --namespace dev --create-namespace


👉 Creates namespace and installs chart.

🔹 Install with Values File
helm install my-nginx bitnami/nginx -f values.yaml


👉 Overrides default values using values.yaml.

🔹 Install with --set Flag
helm install my-nginx bitnami/nginx --set service.type=NodePort


👉 Override specific values via CLI.

4️⃣ Managing Releases
🔹 List Releases
helm list


👉 Shows deployed releases in current namespace.

🔹 List in All Namespaces
helm list -A

🔹 Check Release Status
helm status <release-name>


Example:

helm status my-nginx

🔹 Get Release Values
helm get values <release-name>

🔹 Get All Release Details
helm get all <release-name>

5️⃣ Upgrade & Rollback
🔹 Upgrade Release
helm upgrade <release-name> <chart-name>


Example:

helm upgrade my-nginx bitnami/nginx -f values.yaml


👉 Updates application configuration.

🔹 View Release History
helm history <release-name>

🔹 Rollback Release
helm rollback <release-name> <revision-number>


Example:

helm rollback my-nginx 1


👉 Rolls back to previous working version.

6️⃣ Uninstall
🔹 Delete Release
helm uninstall <release-name>


Example:

helm uninstall my-nginx


👉 Removes application from cluster.

7️⃣ Working with Charts (Development)
🔹 Create New Chart
helm create <chart-name>


Example:

helm create myapp


👉 Generates chart skeleton.

🔹 Lint Chart
helm lint <chart-folder>


👉 Validates chart syntax.

🔹 Package Chart
helm package <chart-folder>


👉 Creates .tgz packaged chart.

🔹 Install Local Chart
helm install myapp ./myapp

🔹 Render Templates (Without Installing)
helm template myapp ./myapp


👉 Outputs Kubernetes YAML manifests.

🔹 Dry Run Installation
helm install myapp ./myapp --dry-run --debug


👉 Simulates installation for debugging.

8️⃣ Helm Dependencies
🔹 Add Dependency (Chart.yaml)
dependencies:
  - name: mysql
    version: 9.0.0
    repository: https://charts.bitnami.com/bitnami

🔹 Update Dependencies
helm dependency update


👉 Downloads required dependency charts.

9️⃣ Chart Information Commands
🔹 Show Default Values
helm show values bitnami/nginx

🔹 Show Chart Info
helm show chart bitnami/nginx

🔹 Show Readme
helm show readme bitnami/nginx

🔹 Verify Chart
helm verify <chart-package>


👉 Verifies signed chart.

🔟 Debugging Commands
🔹 Debug Install
helm install myapp ./myapp --debug

🔹 Debug Upgrade
helm upgrade myapp ./myapp --debug

🔹 Dry Run Upgrade
helm upgrade myapp ./myapp --dry-run

🚀 Real-Time Example

Deploy Nginx with Custom NodePort:

helm install web bitnami/nginx \
  --set service.type=NodePort \
  --set service.nodePorts.http=30007

📌 Common Helm Workflow

Add repository

Search chart

Install chart

Customize values.yaml

Upgrade release

Rollback if needed

Uninstall when not required

🎯 Why Helm is Used in Industry

Standardized Kubernetes deployments

Version-controlled releases

Easy upgrades & rollbacks

CI/CD integration

Reusable templating

Microservices management

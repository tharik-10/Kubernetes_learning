# 📦 Helm Commands -- Complete Learning Guide

## 📌 What is Helm?

Helm is a package manager for Kubernetes that helps you install,
upgrade, rollback, and manage Kubernetes applications using Helm Charts.

------------------------------------------------------------------------

# 1️⃣ Helm Installation & Setup

## Check Helm Version

``` bash
helm version
```

## Add Repository

``` bash
helm repo add <repo-name> <repo-url>
```

Example:

``` bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

## List Repositories

``` bash
helm repo list
```

## Update Repositories

``` bash
helm repo update
```

------------------------------------------------------------------------

# 2️⃣ Searching Charts

## Search in Repo

``` bash
helm search repo <chart-name>
```

## Search in Artifact Hub

``` bash
helm search hub <chart-name>
```

------------------------------------------------------------------------

# 3️⃣ Installing Charts

## Install Chart

``` bash
helm install <release-name> <chart-name>
```

Example:

``` bash
helm install my-nginx bitnami/nginx
```

## Install with Namespace

``` bash
helm install my-nginx bitnami/nginx --namespace dev --create-namespace
```

## Install with Values File

``` bash
helm install my-nginx bitnami/nginx -f values.yaml
```

## Install with --set

``` bash
helm install my-nginx bitnami/nginx --set service.type=NodePort
```

------------------------------------------------------------------------

# 4️⃣ Managing Releases

## List Releases

``` bash
helm list
```

## List All Namespaces

``` bash
helm list -A
```

## Release Status

``` bash
helm status <release-name>
```

## Get Values

``` bash
helm get values <release-name>
```

## Get All Details

``` bash
helm get all <release-name>
```

------------------------------------------------------------------------

# 5️⃣ Upgrade & Rollback

## Upgrade

``` bash
helm upgrade <release-name> <chart-name>
```

## History

``` bash
helm history <release-name>
```

## Rollback

``` bash
helm rollback <release-name> <revision-number>
```

------------------------------------------------------------------------

# 6️⃣ Uninstall

## Delete Release

``` bash
helm uninstall <release-name>
```

------------------------------------------------------------------------

# 7️⃣ Chart Development

## Create Chart

``` bash
helm create <chart-name>
```

## Lint Chart

``` bash
helm lint <chart-folder>
```

## Package Chart

``` bash
helm package <chart-folder>
```

## Install Local Chart

``` bash
helm install myapp ./myapp
```

## Render Templates

``` bash
helm template myapp ./myapp
```

## Dry Run

``` bash
helm install myapp ./myapp --dry-run --debug
```

------------------------------------------------------------------------

# 8️⃣ Dependencies

Add in Chart.yaml:

``` yaml
dependencies:
  - name: mysql
    version: 9.0.0
    repository: https://charts.bitnami.com/bitnami
```

Update dependencies:

``` bash
helm dependency update
```

------------------------------------------------------------------------

# 9️⃣ Chart Information

## Show Values

``` bash
helm show values bitnami/nginx
```

## Show Chart Info

``` bash
helm show chart bitnami/nginx
```

## Show Readme

``` bash
helm show readme bitnami/nginx
```

------------------------------------------------------------------------

# 🔟 Debugging

## Debug Install

``` bash
helm install myapp ./myapp --debug
```

## Debug Upgrade

``` bash
helm upgrade myapp ./myapp --debug
```

## Dry Run Upgrade

``` bash
helm upgrade myapp ./myapp --dry-run
```

------------------------------------------------------------------------

# 🚀 Why Helm is Used in Industry

-   Standardized Kubernetes deployments
-   Version control for releases
-   Easy rollback capability
-   CI/CD integration
-   Reusable templating
-   Microservices management

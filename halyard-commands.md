# Halyard Commands

> **Halyard** is Spinnaker’s official configuration management CLI.
> This repository provides a **complete, learning‑friendly A–Z reference of Halyard commands**, written for **DevOps engineers, SREs, and beginners**.

---

## Purpose of This Repository

* Learn **what Halyard is and why it is used**
* Understand **every major Halyard command with examples**
* Use this as a **ready reference while working with Spinnaker**
* Helpful for **interviews, self‑learning, and real projects**

---

## 📑 Table of Contents

1. [What is Halyard?](#what-is-halyard)
2. [Prerequisites](#prerequisites)
3. [Halyard Basics](#halyard-basics)
4. [Global Commands](#global-commands)
5. [Spinnaker Version Management](#spinnaker-version-management)
6. [Account Management](#account-management)
7. [Provider Management](#provider-management)
8. [Artifact Management](#artifact-management)
9. [Storage Configuration](#storage-configuration)
10. [Security Configuration](#security-configuration)
11. [CI & Notifications](#ci--notifications)
12. [Canary & Metrics](#canary--metrics)
13. [Deployment & Lifecycle](#deployment--lifecycle)
14. [Advanced & Utility Commands](#advanced--utility-commands)
15. [Common Examples](#common-examples)
16. [Best Practices](#best-practices)

---

## What is Halyard?

**Halyard** is a command‑line tool used to:

* Install Spinnaker
* Configure cloud providers (Kubernetes, AWS, GCP, etc.)
* Manage accounts, security, storage, and CI integrations
* Upgrade and deploy Spinnaker safely

> Halyard edits configuration stored in `~/.hal` and applies it to Spinnaker.

---

## Prerequisites

* Linux / macOS
* Java 11 or 17 (based on Spinnaker version)
* Docker (recommended)
* kubectl (for Kubernetes provider)
* Valid cloud credentials (AWS/GCP/etc.)

---

## Halyard Basics

### Check Halyard Version

```bash
hal --version
```

### Help

```bash
hal help
hal <command> --help
```

---

## Global Commands

### Validate Configuration

Checks whether the configuration is valid before deployment.

```bash
hal config deploy validate
```

### Apply Configuration

Deploys Spinnaker using the current Halyard configuration.

```bash
hal deploy apply
```

---

## Spinnaker Version Management

### List Available Spinnaker Versions

```bash
hal version list
```

### Set Spinnaker Version

```bash
hal config version edit --version 1.34.0
```

### Show Current Version

```bash
hal config version show
```

---

## Account Management

### List Accounts

```bash
hal config provider <provider-name> account <action>
```

### Delete an Account

```bash
hal config account delete <account-name>
```

---

## Provider Management

Providers allow Spinnaker to deploy applications to cloud platforms.

---

### Kubernetes Provider

#### Enable Kubernetes Provider

```bash
hal config provider kubernetes enable
```

#### Add Kubernetes Account

```bash
hal config provider kubernetes account add my-k8s \
  --context my-cluster \
  --provider-version v2
```

#### List Kubernetes Accounts

```bash
hal config provider kubernetes account list
```

#### Delete Kubernetes Account

```bash
hal config provider kubernetes account delete my-k8s
```

---

### AWS Provider

#### Enable AWS Provider

```bash
hal config provider aws enable
```

#### Add AWS Account

```bash
hal config provider aws account add my-aws \
  --account-id 123456789012 \
  --assume-role role/spinnakerManaged
```

---

## Artifact Management

Artifacts are external inputs to pipelines (Docker images, Git repos, Helm charts).

### Enable Artifacts Feature

```bash
hal config features edit --artifacts true
```

### Enable GitHub Artifacts

```bash
hal config artifact github enable
```

### Enable Docker Registry Artifacts

```bash
hal config artifact docker-registry enable
```

---

## Storage Configuration

Spinnaker requires persistent storage to store pipelines and metadata.

---

### S3 Storage

#### Enable S3

```bash
hal config storage s3 enable
```

#### Configure S3

```bash
hal config storage s3 edit \
  --bucket spinnaker-bucket \
  --region ap-south-1
```

---

### GCS Storage

```bash
hal config storage gcs enable
```

---

## Security Configuration

### Authentication

#### Enable OAuth2

```bash
hal config security authn oauth2 enable
```

### Authorization

#### Enable RBAC

```bash
hal config security authz enable
```

### SSL / HTTPS

```bash
hal config security api ssl enable
hal config security ui ssl enable
```

---

## CI & Notifications

### Jenkins CI Integration

```bash
hal config ci jenkins enable
```

```bash
hal config ci jenkins master add my-jenkins \
  --address http://jenkins:8080 \
  --username admin \
  --password password
```

### Slack Notifications

```bash
hal config notification slack enable
```

### Email Notifications

```bash
hal config notification email enable
```

---

## Canary & Metrics

### Enable Canary Analysis

```bash
hal config canary enable
```

### Prometheus Metrics

```bash
hal config metric-store prometheus enable
```

### Datadog Metrics

```bash
hal config metric-store datadog enable
```

---

## Deployment & Lifecycle

### Deploy Spinnaker

```bash
hal deploy apply
```

### Check Deployment Status

```bash
hal deploy status
```

### Clean Deployment

```bash
hal deploy clean
```

### Delete Spinnaker

```bash
hal deploy delete
```

---

## Advanced & Utility Commands

### Export Configuration

```bash
hal config export
```

### Backup Configuration

```bash
hal config export > hal-config-backup.yml
```

### Restore Configuration

```bash
hal config import --file hal-config-backup.yml
```

---

## Common Examples

### Kubernetes + S3 Deployment Example

```bash
hal config provider kubernetes enable
hal config provider kubernetes account add prod \
  --context prod-cluster

hal config storage s3 enable
hal config storage s3 edit \
  --bucket spinnaker-prod \
  --region ap-south-1

hal deploy apply
```

---

## Best Practices

- Always run validation before applying configuration
- Backup Halyard config before upgrades
- Prefer IAM roles over static credentials
- Keep Halyard and Spinnaker versions compatible
- Secure `~/.hal` directory properly

---

## References

* Official Docs: [https://spinnaker.io](https://spinnaker.io)
* Halyard GitHub: [https://github.com/spinnaker/halyard](https://github.com/spinnaker/halyard)

---

**Maintained for learning & DevOps community use**

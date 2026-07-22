# Lab 3 – Helm Charts: Package & Deploy on EKS

> [!NOTE]
> **JD Skills Covered**
>
> - Helm
> - Kustomize
> - EKS
> - DEV / SIT / UAT / PROD
> - values.yaml

---

# Overview

This lab demonstrates how to create a custom Helm chart, package your Kubernetes application, and deploy it to multiple environments on Amazon EKS using environment-specific configuration files.

Helm enables consistent, repeatable, and version-controlled Kubernetes deployments across development and production environments.

---

# Lab Objective

Create a custom Helm chart for your application and deploy it to different namespaces (**DEV**, **SIT/UAT**, and **PROD**) using environment-specific values files. This lab aligns with the JD requirement for **Helm/Kustomize-based environment releases** on AKS/EKS.

---

# Step 1 – Create a Helm Chart

## 1.1 Install Helm 3

Install Helm 3 on Linux.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify the installation.

```bash
helm version
```

> [!NOTE]
> Ensure Helm 3 is installed before proceeding with chart creation.

---

## 1.2 Create the Chart Scaffold

Generate the default Helm chart structure.

```bash
helm create my-app
```

This command creates the following directory structure:

```text
my-app/
├── Chart.yaml
├── values.yaml
├── charts/
└── templates/
```

---

## 1.3 Edit `Chart.yaml`

Update the chart metadata.

```yaml
apiVersion: v2
name: my-app
description: My DevOps Demo Application
type: application
version: 0.1.0
appVersion: "1.0.0"
```

---

# Step 2 – Configure Values Files per Environment

Create separate values files for each deployment environment.

---

## 2.1 Create `values-dev.yaml`

Configure the Development environment.

```yaml
replicaCount: 1

image:
  repository: 123456789.dkr.ecr.ap-south-1.amazonaws.com/my-app
  tag: latest
  pullPolicy: Always

resources:
  requests:
    memory: 128Mi
    cpu: 100m
  limits:
    memory: 256Mi
    cpu: 200m

service:
  type: ClusterIP
  port: 80

env: dev
```

> [!NOTE]
> The Development environment uses a single replica with minimal resource allocation.

---

## 2.2 Create `values-prod.yaml`

Configure the Production environment.

```yaml
replicaCount: 3

image:
  repository: 123456789.dkr.ecr.ap-south-1.amazonaws.com/my-app
  tag: v1.2.0
  pullPolicy: IfNotPresent

resources:
  requests:
    memory: 512Mi
    cpu: 250m
  limits:
    memory: 1Gi
    cpu: 500m

service:
  type: LoadBalancer
  port: 80

env: production
```

> [!IMPORTANT]
> Production deployments use multiple replicas, increased resource limits, and a `LoadBalancer` service for high availability.

---

# Step 3 – Deploy the Helm Chart to Amazon EKS

## 3.1 Create Kubernetes Namespaces

Create a namespace for each deployment environment.

```bash
kubectl create namespace dev
kubectl create namespace sit
kubectl create namespace prod
```

---

## 3.2 Deploy to the Development Environment

Install the Helm chart using the Development values file.

```bash
helm install my-app-dev ./my-app \
    -f my-app/values-dev.yaml \
    -n dev
```

Verify the deployment.

```bash
helm list -n dev
```

---

## 3.3 Deploy to the Production Environment

Install the Helm chart using the Production values file.

```bash
helm install my-app-prod ./my-app \
    -f my-app/values-prod.yaml \
    -n prod
```

Monitor the deployed Pods.

```bash
kubectl get pods -n prod -w
```

---

## 3.4 Upgrade a Helm Release

After making application or chart changes, upgrade the existing release.

```bash
helm upgrade my-app-dev ./my-app \
    -f my-app/values-dev.yaml \
    -n dev
```

View the release history.

```bash
helm history my-app-dev -n dev
```

---

## 3.5 Roll Back a Helm Release

Roll back to a previous release revision.

```bash
helm rollback my-app-dev 1 -n dev
```

> **Note:** `1` represents the Helm revision number.

Verify the release status.

```bash
helm status my-app-dev -n dev
```

---

# Validation

After completing this lab, verify the following:

- ✅ Helm 3 is installed successfully.
- ✅ Helm chart scaffold has been created.
- ✅ Environment-specific values files are configured.
- ✅ DEV and PROD deployments are running successfully.
- ✅ Helm upgrade completes successfully.
- ✅ Helm rollback restores the previous release revision.

---

# Best Practice Tips

> [!TIP]

- Run `helm lint ./my-app` before every deployment to identify syntax and template issues.
- Store Helm charts in a Git repository to maintain version history and support collaborative development.
- Use **Helmfile** to manage multi-chart releases and orchestrate deployments across multiple environments.
- Avoid using the `latest` image tag in production. Use immutable image tags such as Git commit SHAs or semantic version tags.
- Use the `--atomic` flag with `helm install` or `helm upgrade` to automatically roll back failed deployments.
- Preview rendered Kubernetes manifests before deployment using:

```bash
helm template ./my-app -f values-prod.yaml
```

---

# Lab Summary

In this lab, you:

- Installed Helm 3.
- Created a custom Helm chart.
- Configured environment-specific values files.
- Deployed applications to Amazon EKS.
- Performed Helm upgrades.
- Executed Helm rollbacks.
- Applied recommended Helm deployment best practices for multi-environment Kubernetes workloads.

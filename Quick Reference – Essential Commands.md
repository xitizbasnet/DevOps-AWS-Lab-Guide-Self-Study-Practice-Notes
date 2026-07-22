# Quick Reference – Essential Commands

This reference guide provides commonly used commands for managing:

- Amazon EKS clusters
- Kubernetes workloads
- Helm deployments
- Terraform infrastructure
- Amazon ECR security scanning
- AWS Secrets Manager
- Kubernetes troubleshooting and monitoring

---

# Amazon EKS and Kubernetes Commands

| Command | Description |
|---------|-------------|
| `eksctl create cluster -f cluster.yaml` | Create an Amazon EKS cluster from a configuration file |
| `aws eks update-kubeconfig --name CLUSTER --region REGION` | Update local kubeconfig to connect with an Amazon EKS cluster |
| `kubectl get nodes -o wide` | List all Kubernetes nodes with detailed information |
| `kubectl get pods -A` | List all Pods across all Kubernetes namespaces |
| `kubectl rollout status deployment/NAME -n NS` | Monitor rolling deployment progress |
| `kubectl rollout undo deployment/NAME -n NS` | Roll back a Kubernetes deployment to the previous version |
| `kubectl logs POD_NAME -n NS --previous` | View logs from a previously crashed Pod |
| `kubectl describe pod POD_NAME -n NS` | View Pod events, status, and troubleshooting details |
| `kubectl top nodes && kubectl top pods -A` | View Kubernetes resource usage (requires Metrics Server) |

---

# Helm Deployment Commands

| Command | Description |
|---------|-------------|
| `helm install NAME ./chart -f values.yaml -n NS` | Install a Helm chart into a Kubernetes namespace |
| `helm upgrade NAME ./chart -f values.yaml -n NS --atomic` | Upgrade a Helm release with automatic rollback on failure |
| `helm rollback NAME REVISION -n NS` | Roll back a Helm release to a specific revision |

---

# Terraform Infrastructure Commands

| Command | Description |
|---------|-------------|
| `terraform init && terraform plan -out=tfplan` | Initialize Terraform and create an execution plan |
| `terraform apply tfplan` | Apply the approved Terraform execution plan |

---

# Amazon ECR Security Commands

| Command | Description |
|---------|-------------|
| `aws ecr describe-image-scan-findings --repository-name REPO --image-id imageTag=TAG` | Retrieve Amazon ECR image vulnerability scan results |

Example:

```bash
aws ecr describe-image-scan-findings \
--repository-name my-app \
--image-id imageTag=v1.0.0
```

---

# AWS Secrets Manager Commands

| Command | Description |
|---------|-------------|
| `aws secretsmanager get-secret-value --secret-id SECRET_NAME` | Retrieve a secret value stored in AWS Secrets Manager |

Example:

```bash
aws secretsmanager get-secret-value \
--secret-id prod/my-app/database
```

> [!WARNING]
> Avoid displaying secret values directly in shared terminals, logs, or CI/CD output.

---

# Kubernetes Troubleshooting Commands

| Command | Description |
|---------|-------------|
| `kubectl describe pod POD_NAME -n NS` | Review Pod configuration, events, and scheduling issues |
| `kubectl logs POD_NAME -n NS --previous` | Check logs from the previous failed container instance |
| `kubectl get pods -A` | Verify Pod status across all namespaces |

---

# Kubernetes Monitoring Commands

## View Node and Pod Resource Usage

```bash
kubectl top nodes

kubectl top pods -A
```

> [!NOTE]
> These commands require the Kubernetes Metrics Server to be installed.

---

# Command Categories Summary

| Category | Primary Tools |
|----------|---------------|
| Cluster Management | eksctl, kubectl |
| Application Deployment | Helm, kubectl |
| Infrastructure Provisioning | Terraform |
| Container Security | Amazon ECR |
| Secret Management | AWS Secrets Manager |
| Troubleshooting | kubectl logs, kubectl describe |
| Monitoring | kubectl top, CloudWatch |

---

# Best Practice Tips

> [!TIP]

- Use configuration files (`cluster.yaml`, Helm values files, Terraform files) instead of manual commands for repeatable deployments.
- Always verify deployment status after applying changes.
- Use `--atomic` with Helm upgrades to automatically roll back failed releases.
- Review Terraform plans before applying infrastructure changes.
- Avoid exposing credentials or secrets in command history, scripts, or CI/CD logs.
- Use namespaces consistently to separate environments such as `dev`, `sit`, `uat`, and `prod`.

---

# Quick Troubleshooting Workflow

When a deployment fails:

```text
1. Check Pod status
        |
        ↓
2. Describe Pod events
        |
        ↓
3. Review application logs
        |
        ↓
4. Check resource usage
        |
        ↓
5. Validate configuration changes
```

Recommended commands:

```bash
kubectl get pods -n NS

kubectl describe pod POD_NAME -n NS

kubectl logs POD_NAME -n NS

kubectl top pods -n NS
```

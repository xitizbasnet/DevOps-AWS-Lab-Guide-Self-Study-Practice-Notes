# Lab 8 – Complete DevOps Pipeline: End-to-End CI,CD on AWS

> [!NOTE]
> **JD Skills Covered**
>
> - End-to-End DevOps Pipeline
> - CI/CD
> - Amazon EKS
> - Helm
> - Security Integration
> - Amazon CloudWatch

---

# Overview

This lab combines all previous labs into a complete enterprise-style DevOps pipeline.

The pipeline automates the complete application lifecycle:

```text
Code
 ↓
Build
 ↓
Test
 ↓
Security Scan
 ↓
Push Container Image
 ↓
Deploy to Kubernetes
 ↓
Monitor Application
```

The implementation demonstrates a production-style CI/CD workflow using GitHub Actions, Amazon ECR, Amazon EKS, Helm, security scanning tools, and CloudWatch monitoring.

---

# Architecture Overview

| Stage | Tool / AWS Service | Output |
|-------|--------------------|--------|
| Source | GitHub Repository | Code push triggers pipeline |
| Build | GitHub Actions + AWS CodeBuild | Docker image built |
| Test | npm test / pytest + SonarCloud | Test results and quality report |
| Scan | Trivy + ECR Enhanced Scanning | CVE report; fail on CRITICAL vulnerabilities |
| Push | Amazon ECR | Versioned container image stored |
| Deploy DEV | Helm + kubectl → Amazon EKS | Application running in development namespace |
| Deploy PROD | Helm + Blue-Green / Canary → Amazon EKS | Zero-downtime production release |
| Monitor | CloudWatch + Container Insights | Dashboards, metrics, and alarms |
| Security | GuardDuty + Secrets Manager | Runtime protection and secret management |

---

# End-to-End Pipeline Flow

```text
Developer Push
       |
       ↓
GitHub Repository
       |
       ↓
GitHub Actions Pipeline
       |
       ├── Unit Testing
       |
       ├── SonarCloud Quality Scan
       |
       ├── Docker Build
       |
       ├── Trivy Security Scan
       |
       ├── Push Image to Amazon ECR
       |
       ↓
Deploy to DEV Environment
       |
       ↓
Validation / Approval
       |
       ↓
Deploy to PROD Environment
       |
       ↓
CloudWatch Monitoring
```

---

# Complete GitHub Actions Workflow (E2E)

Create the workflow file:

```text
.github/workflows/devops-pipeline.yml
```

```yaml
name: Full DevOps Pipeline

on:

  push:
    branches:
      - main

env:

  AWS_REGION: ap-south-1
  ECR_REPOSITORY: my-app
  EKS_CLUSTER: devops-cluster


jobs:

# ==================================================
# CI Stage: Build, Test, Scan and Push
# ==================================================

  ci:

    name: Build, Test, Scan & Push

    runs-on: ubuntu-latest

    outputs:

      image: ${{ steps.build.outputs.image }}


    steps:

    - name: Checkout Code
      uses: actions/checkout@v4

      with:

        fetch-depth: 0


    - name: Run Unit Tests

      run: |

        npm install
        npm test


    - name: SonarCloud Scan

      uses: SonarSource/sonarcloud-github-action@master

      env:

        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}


    - name: Configure AWS Credentials

      uses: aws-actions/configure-aws-credentials@v4

      with:

        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}

        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

        aws-region: ${{ env.AWS_REGION }}


    - name: Login to Amazon ECR

      id: login-ecr

      uses: aws-actions/amazon-ecr-login@v2


    - name: Build and Push Docker Image

      id: build

      run: |

        IMAGE=${{ steps.login-ecr.outputs.registry }}/$ECR_REPOSITORY:${{ github.sha }}

        docker build -t $IMAGE .

        docker push $IMAGE

        echo image=$IMAGE >> $GITHUB_OUTPUT


    - name: Trivy Security Scan

      uses: aquasecurity/trivy-action@master

      with:

        image-ref: ${{ steps.build.outputs.image }}

        severity: CRITICAL,HIGH

        exit-code: 1


# ==================================================
# DEV Deployment Stage
# ==================================================

  deploy-dev:

    needs: ci

    runs-on: ubuntu-latest

    environment: dev


    steps:


    - name: Checkout Code

      uses: actions/checkout@v4


    - name: Configure AWS Credentials

      uses: aws-actions/configure-aws-credentials@v4

      with:

        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}

        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

        aws-region: ${{ env.AWS_REGION }}


    - name: Update kubeconfig

      run: |

        aws eks update-kubeconfig \
        --region $AWS_REGION \
        --name $EKS_CLUSTER


    - name: Deploy Application to DEV

      run: |

        helm upgrade --install my-app-dev ./helm/my-app \
        -f helm/my-app/values-dev.yaml \
        -n dev \
        --create-namespace \
        --set image.tag=${{ github.sha }} \
        --atomic \
        --wait


# ==================================================
# PROD Deployment Stage
# ==================================================

  deploy-prod:

    needs: deploy-dev

    runs-on: ubuntu-latest

    environment: prod


    steps:


    - name: Checkout Code

      uses: actions/checkout@v4


    - name: Configure AWS Credentials

      uses: aws-actions/configure-aws-credentials@v4

      with:

        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}

        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

        aws-region: ${{ env.AWS_REGION }}


    - name: Update kubeconfig

      run: |

        aws eks update-kubeconfig \
        --region $AWS_REGION \
        --name $EKS_CLUSTER


    - name: Helm Blue-Green Deployment to PROD

      run: |

        helm upgrade --install my-app-prod ./helm/my-app \
        -f helm/my-app/values-prod.yaml \
        -n prod \
        --create-namespace \
        --set image.tag=${{ github.sha }} \
        --atomic \
        --wait
```

---

# Pipeline Stage Explanation

## Source Stage

**Tool: GitHub Repository**

Purpose:

- Stores application source code.
- Triggers pipeline execution when changes are pushed to the main branch.

---

## Build and Test Stage

**Tools: GitHub Actions + npm/pytest**

Activities:

- Checkout source code.
- Install dependencies.
- Execute unit tests.
- Generate test reports.

---

## Code Quality Stage

**Tool: SonarCloud**

Activities:

- Static code analysis.
- Quality gate validation.
- Code maintainability checks.

---

## Security Scanning Stage

**Tools: Trivy + Amazon ECR Enhanced Scanning**

Activities:

- Scan Docker images.
- Identify vulnerable dependencies.
- Block deployment when Critical vulnerabilities are detected.

---

## Container Registry Stage

**Tool: Amazon ECR**

Activities:

- Store versioned container images.
- Maintain immutable image history.

Image tagging strategy:

```text
<repository>:<commit-sha>
```

Example:

```text
my-app:a8f93d21
```

---

## Deployment Stage

**Tools: Helm + Amazon EKS**

Deployment flow:

```text
GitHub Actions
        |
        ↓
Helm Chart
        |
        ↓
Amazon EKS
        |
        ↓
Kubernetes Namespace
```

DEV deployment:

```text
dev namespace
```

Production deployment:

```text
prod namespace
```

---

# Important Notes

> [!IMPORTANT]

- Configure **GitHub Environments** with required reviewers for the `prod` environment to add a manual approval gate before production deployment.
- Use **OIDC authentication** with `aws-actions/configure-aws-credentials` and `role-to-assume` instead of static AWS access keys for production pipelines.
- In the AWS Free Tier:
  - The Amazon EKS control plane has associated costs.
  - EC2 worker nodes are billed separately.
  - Use smaller instance types for learning environments.
  - Terminate resources after practice sessions to avoid unnecessary charges.

---

# Best Practice Tips

> [!TIP]

- Treat the pipeline as code. Store `.github/workflows` YAML files in version control and review changes through pull requests.
- Add a smoke test job after DEV deployment to validate application availability before promoting to production.
- Use GitHub Environment protection rules:
  - Required reviewers
  - Wait timers
  - Deployment branch restrictions

- Document every pipeline stage in an operational Runbook (for example, Confluence or Notion).
- Tag container images using:
  - Commit SHA for traceability
  - Branch name for environment tracking
  - Semantic version tags for production releases

Example:

```text
my-app:
    v1.0.0
    main-a82f91c
```

- Monitor pipeline execution time.

If CI execution exceeds 15 minutes, optimize by:

- Running jobs in parallel.
- Caching dependencies such as `node_modules`.
- Using multi-stage Docker builds.

---

# Validation Checklist

After completing this lab, verify:

- ✅ Code push triggers the GitHub Actions workflow.
- ✅ Unit tests execute successfully.
- ✅ SonarCloud quality analysis completes.
- ✅ Docker image is created and pushed to Amazon ECR.
- ✅ Trivy scan blocks vulnerable images.
- ✅ DEV deployment completes successfully on Amazon EKS.
- ✅ Production deployment requires approval.
- ✅ Helm deployment completes successfully.
- ✅ CloudWatch monitors deployed workloads.

---

# Lab Summary

In this lab, you:

- Combined CI/CD, security, deployment, and monitoring workflows into a complete DevOps pipeline.
- Automated application delivery using GitHub Actions.
- Built and secured container images.
- Deployed applications to Amazon EKS using Helm.
- Implemented production deployment controls.
- Applied enterprise DevOps practices for cloud-native application delivery.

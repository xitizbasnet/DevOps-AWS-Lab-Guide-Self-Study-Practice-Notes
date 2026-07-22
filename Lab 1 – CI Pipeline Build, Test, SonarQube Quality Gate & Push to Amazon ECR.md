# Lab 1 – CI Pipeline Build, Test, SonarQube Quality Gate & Push to Amazon ECR

> [!NOTE]
> **JD Skills Covered**
>
> - CI/CD
> - GitHub Actions
> - Amazon ECR
> - SonarQube
> - YAML

---

# Overview

This lab demonstrates how to configure a GitHub Actions workflow that automatically builds a Docker image, runs unit tests, performs a SonarQube/SonarCloud code-quality scan, and pushes the Docker image to Amazon ECR.

This implementation mirrors the JD requirement for:

- CI/CD setup
- GitHub Actions
- YAML pipeline creation
- Artifact packaging
- Container registry integration

---

# Lab Objective

Configure a GitHub Actions workflow that automatically:

- Builds a Docker image on every push.
- Runs unit tests.
- Performs a SonarQube/SonarCloud code-quality scan (equivalent to CodeQL/SAST).
- Publishes the Docker image to Amazon ECR.

---

# Prerequisites

Before starting this lab, ensure the following requirements are met.

| Requirement | Details |
|------------|---------|
| GitHub Repository | Any public or private repository with a sample Node.js / Python / Java application |
| AWS Account | IAM user with ECR Full Access + Secrets permissions |
| SonarCloud | Free account at sonarcloud.io linked to your GitHub repository |
| Amazon ECR | Create a private repository using AWS Console → ECR → Create Repository → Private |

> [!IMPORTANT]
> Verify that all prerequisite accounts and services are configured before proceeding.

---

# Step 1 – Create AWS Credentials as GitHub Secrets

## 1.1 Create an IAM User

Navigate to:

**IAM Console → Users → Create User**

User name:

```text
github-actions-user
```

Attach the following policy:

```text
AmazonEC2ContainerRegistryFullAccess
```

---

## 1.2 Generate Access Keys

Navigate to:

**IAM → User → Security credentials → Create access key → CLI**

Copy the following values:

- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY

---

## 1.3 Add GitHub Repository Secrets

Navigate to:

**GitHub Repository → Settings → Secrets and variables → Actions → New secret**

Create the following secrets:

| Secret | Value |
|---------|-------|
| AWS_ACCESS_KEY_ID | AWS Access Key |
| AWS_SECRET_ACCESS_KEY | AWS Secret Key |
| AWS_REGION | Example: ap-south-1 |
| ECR_REPOSITORY | Your ECR repository name |
| SONAR_TOKEN | SonarCloud Token |

> [!TIP]
> Store all credentials as encrypted GitHub Secrets. Never commit credentials into source control.

---

# Step 2 – Create the GitHub Actions Workflow

Create the following file:

```text
.github/workflows/ci-pipeline.yml
```

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, develop]

  pull_request:
    branches: [main]

env:
  AWS_REGION: ${{ secrets.AWS_REGION }}
  ECR_REPOSITORY: ${{ secrets.ECR_REPOSITORY }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:

      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run Unit Tests
        run: |
          npm install
          npm test -- --coverage

      - name: SonarCloud Scan (SAST / Code Quality)
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
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

      - name: Build, Tag & Push Docker Image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}

        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG >> $GITHUB_OUTPUT
```

---

# Step 3 – Create Amazon ECR Repository

## 3.1 Open Amazon ECR

Navigate to:

**AWS Console → Search "ECR" → Elastic Container Registry**

---

## 3.2 Create Repository

Select:

**Create repository → Private**

Repository Name:

```text
my-app
```

Enable:

- Image scan on push

Select:

**Create**

---

## 3.3 Repository URI

Copy the repository URI.

Example:

```text
123456789.dkr.ecr.ap-south-1.amazonaws.com/my-app
```

---

## 3.4 Verify Image Push

After the GitHub Actions workflow completes successfully, navigate to:

**Amazon ECR → my-app → Images**

Verify that the SHA-tagged Docker image appears.

> [!NOTE]
> Successful image creation confirms that the CI pipeline has authenticated with AWS, built the Docker image, and pushed it to Amazon ECR.

---

# Best Practice Tips

> [!TIP]

- Always use immutable image tags (commit SHA) instead of `latest` to enable rollback.
- Enable ECR image scanning on push to detect OS-level vulnerabilities automatically.
- Store AWS credentials as GitHub Encrypted Secrets—never hardcode them in YAML.
- Use GitHub Environments with required reviewers for production deployments.
- Add a Quality Gate in SonarCloud to fail the build if coverage is less than **80%** or Critical bugs are greater than **0**.
- Use OIDC (OpenID Connect) instead of long-lived access keys for GitHub Actions to AWS for improved security.

---

# Lab Summary

In this lab, you configured a GitHub Actions CI pipeline that:

- Built a Docker image.
- Executed unit tests.
- Performed a SonarCloud quality scan.
- Authenticated with AWS.
- Published a Docker image to Amazon ECR.
- Followed recommended CI/CD security and deployment best practices.

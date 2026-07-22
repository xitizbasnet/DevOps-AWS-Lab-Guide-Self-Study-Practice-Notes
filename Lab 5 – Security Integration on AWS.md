# Lab 5 – Security Integration on AWS

> [!NOTE]
> **JD Skills Covered**
>
> - DevSecOps
> - ECR Image Scanning
> - AWS Secrets Manager
> - Amazon GuardDuty
> - Software Composition Analysis (SCA)

---

# Overview

This lab demonstrates how to integrate security throughout the software delivery lifecycle by implementing container vulnerability scanning, centralized secret management, dependency scanning (SCA), and runtime threat detection.

These activities directly align with the JD requirement for implementing **DevSecOps** practices, including dependency scanning, container image scanning, secret management, and secure runtime monitoring.

---

# Lab Objective

Integrate security controls at every stage of the CI/CD pipeline by implementing:

- Amazon ECR Enhanced Image Scanning
- AWS Secrets Manager
- Amazon GuardDuty
- Software Composition Analysis (SCA)

---

# Security Architecture

This lab secures the application lifecycle through multiple layers.

| Security Layer | AWS Service | Purpose |
|---------------|-------------|---------|
| Container Security | Amazon ECR Enhanced Scanning (Amazon Inspector) | Detect vulnerabilities in container images |
| Secret Management | AWS Secrets Manager | Securely store application secrets |
| Runtime Threat Detection | Amazon GuardDuty | Monitor EKS runtime activity and threats |
| Dependency Scanning (SCA) | Trivy | Detect vulnerable application dependencies |

---

# Step 1 – Configure Amazon ECR Enhanced Image Scanning

> [!IMPORTANT]
> Amazon ECR Enhanced Scanning uses **Amazon Inspector** to continuously scan container images for known vulnerabilities.

---

## 1.1 Enable Enhanced Scanning

Navigate to:

**AWS Console → Amazon ECR → Private Registry → Scanning Configuration**

Configure the following settings:

| Setting | Value |
|---------|-------|
| Scan Type | Enhanced scanning |
| Scan Engine | Amazon Inspector |
| Scan Frequency | Continuous scanning |

Select **Save scanning settings**.

---

## 1.2 View Scan Results

Navigate to:

**Amazon ECR → Repositories → my-app → Images → Select Image → Vulnerabilities**

Review the vulnerability findings.

Typical information includes:

- CVE ID
- Severity
- Installed Version
- Fixed Version

> [!TIP]
> Prioritize remediation of **Critical** and **High** severity vulnerabilities before promoting images to production.

---

## 1.3 Fail the CI Pipeline on Critical Vulnerabilities

Add the following step to the GitHub Actions workflow **after** the Docker image is pushed to Amazon ECR.

```yaml
- name: Check ECR Scan Results
  run: |
    FINDINGS=$(aws ecr describe-image-scan-findings \
      --repository-name $ECR_REPOSITORY \
      --image-id imageTag=$IMAGE_TAG \
      --query 'imageScanFindings.findingSeverityCounts.CRITICAL' \
      --output text)

    if [ "$FINDINGS" != "None" ] && [ "$FINDINGS" -gt 0 ]; then
      echo "CRITICAL vulnerabilities found! Failing build."
      exit 1
    fi
```

> [!WARNING]
> Failing the pipeline on **Critical** vulnerabilities prevents insecure container images from progressing through the deployment pipeline.

---

# Step 2 – Configure AWS Secrets Manager

> [!NOTE]
> AWS Secrets Manager is the AWS equivalent of Azure Key Vault for securely storing and managing application secrets.

---

## 2.1 Create a Secret

Navigate to:

**AWS Console → AWS Secrets Manager → Store a New Secret**

Configure the secret as follows.

| Property | Value |
|----------|-------|
| Secret Type | Other type of secret |
| Key | DB_PASSWORD |
| Value | MySecurePassword123! |
| Key | API_KEY |
| Value | abc123xyz |
| Secret Name | prod/my-app/database |

Select:

**Next → Next → Store**

---

## 2.2 Install the External Secrets Operator

Install the External Secrets Operator in Amazon EKS.

```bash
helm repo add external-secrets https://charts.external-secrets.io

helm install external-secrets external-secrets/external-secrets \
    -n external-secrets \
    --create-namespace
```

---

## 2.3 Create SecretStore and ExternalSecret Resources

Create the following Kubernetes resources.

```yaml
# secret-store.yaml

apiVersion: external-secrets.io/v1beta1
kind: SecretStore

metadata:
  name: aws-secretsmanager

spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
```

```yaml
# external-secret.yaml

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret

metadata:
  name: my-app-db-secret

spec:
  refreshInterval: 1h

  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore

  target:
    name: my-app-secret

  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: prod/my-app/database
        property: DB_PASSWORD
```

Apply the resources to the Amazon EKS cluster.

```bash
kubectl apply -f secret-store.yaml

kubectl apply -f external-secret.yaml
```

---

# Step 3 – Enable Amazon GuardDuty

Amazon GuardDuty provides continuous threat detection for AWS workloads and Amazon EKS clusters.

---

## 3.1 Enable GuardDuty

Navigate to:

**AWS Console → Amazon GuardDuty → Get Started → Enable GuardDuty**

---

## 3.2 Enable Amazon EKS Protection

Navigate to:

**Amazon GuardDuty → Protection Plans → EKS Protection**

Enable:

- EKS Protection

Coverage includes:

- EKS Audit Logs
- EKS Runtime Monitoring (agent installed on worker nodes)

---

## 3.3 Review Security Findings

Navigate to:

**Amazon GuardDuty → Findings**

Review findings by severity.

Example finding:

```text
UnauthorizedAccess:EC2/MaliciousIPCaller
```

Each finding includes:

- Severity
- Affected Resource
- Recommended Remediation

---

## 3.4 Add Software Composition Analysis (SCA) to GitHub Actions

Use Trivy to scan application dependencies and Docker images for known vulnerabilities.

```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master

  with:
    image-ref: ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
    format: "sarif"
    output: "trivy-results.sarif"
    severity: "CRITICAL,HIGH"
    exit-code: "1"
```

> [!IMPORTANT]
> Setting `exit-code: "1"` causes the CI pipeline to fail when vulnerabilities matching the configured severity are detected.

---

# Validation

After completing this lab, verify the following:

- ✅ Amazon ECR Enhanced Image Scanning is enabled.
- ✅ Container image vulnerabilities are visible in Amazon ECR.
- ✅ GitHub Actions fails when Critical vulnerabilities are detected.
- ✅ Secrets are securely stored in AWS Secrets Manager.
- ✅ External Secrets Operator synchronizes secrets to Amazon EKS.
- ✅ Amazon GuardDuty is enabled for the AWS account.
- ✅ Amazon EKS Protection is enabled.
- ✅ Trivy successfully performs Software Composition Analysis (SCA).

---

# Best Practice Tips

> [!TIP]

- Never expose secrets as plain-text environment variables. Store sensitive information in **AWS Secrets Manager** or **AWS Systems Manager Parameter Store**.
- Enable automatic secret rotation using **AWS Secrets Manager** with AWS Lambda.
- Use **IAM Roles for Service Accounts (IRSA)** to allow Amazon EKS workloads to securely access AWS services without static credentials.
- Keep Amazon ECR repositories private and enforce repository policies to prevent unauthorized access.
- Integrate **Trivy** or **Snyk** into CI/CD pipelines to prevent merging pull requests containing Critical vulnerabilities.
- Enable **AWS Security Hub** to aggregate and centralize findings from Amazon GuardDuty, Amazon Inspector, AWS Config, and other AWS security services.

---

# Lab Summary

In this lab, you:

- Enabled Amazon ECR Enhanced Image Scanning.
- Integrated vulnerability scanning into the CI/CD pipeline.
- Stored application secrets securely using AWS Secrets Manager.
- Configured External Secrets Operator for Amazon EKS.
- Enabled Amazon GuardDuty for runtime threat detection.
- Added Software Composition Analysis (SCA) using Trivy.
- Applied DevSecOps best practices for securing containerized workloads on AWS.

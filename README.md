# AWS DevOps Skills Mapping

## Overview

This document maps common DevOps job description (JD) skill areas to their AWS service equivalents and the corresponding hands-on laboratory exercises.

> [!NOTE]
> This mapping is intended to help engineers identify the AWS technologies required for each competency and the associated lab used for practical implementation.

---

## Learning Objectives

After completing the associated labs, you will be able to:

- Implement CI/CD pipelines using AWS services.
- Build and manage container images.
- Deploy and manage Kubernetes workloads on Amazon EKS.
- Package applications using Helm.
- Implement deployment strategies.
- Apply DevSecOps security controls.
- Provision infrastructure using Terraform.
- Configure monitoring and centralized logging.
- Automate administrative tasks using Bash scripting.

---

## Skills Mapping Matrix

| Skill Area | AWS Technology | Laboratory |
|------------|----------------|------------|
| **CI/CD Pipelines** | CodePipeline, GitHub Actions, CodeBuild | **Lab 1, Lab 2** |
| **Container Build & Registry** | Docker, Amazon ECR | **Lab 2** |
| **Kubernetes Cluster Management** | Amazon EKS (eksctl / AWS Management Console) | **Lab 3** |
| **Helm / Kustomize** | Helm Charts on Amazon EKS | **Lab 4** |
| **Deployment Strategies** | Rolling, Blue-Green, Canary Deployments on Amazon EKS | **Lab 5** |
| **Security (DevSecOps)** | Amazon ECR Image Scanning, AWS Secrets Manager, Amazon GuardDuty | **Lab 6** |
| **Infrastructure as Code (IaC)** | Terraform on AWS | **Lab 7** |
| **Monitoring & Logging** | Amazon CloudWatch, CloudWatch Logs, Container Insights | **Lab 8** |
| **Scripting (Bash/Python)** | Bash scripts used throughout all laboratories | **All Labs** |

---

## Prerequisites

Complete the following prerequisites before beginning any laboratory.

> [!IMPORTANT]
> Verify that all required software is installed and configured successfully.

| Requirement | Purpose |
|------------|---------|
| AWS Free Tier Account | AWS resource provisioning |
| AWS CLI v2 | AWS command-line management |
| Docker Desktop | Container build and execution |
| kubectl | Kubernetes cluster management |
| eksctl | Amazon EKS cluster provisioning |
| Helm 3 | Kubernetes package management |
| Terraform | Infrastructure as Code (IaC) |
| Git | Source control management |

---

## Summary

The above mapping provides a structured learning path that aligns industry DevOps skills with AWS-native services and corresponding practical laboratories.


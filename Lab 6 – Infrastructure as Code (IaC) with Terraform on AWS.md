# Lab 6 – Infrastructure as Code (IaC) with Terraform on AWS

> [!NOTE]
> **JD Skills Covered**
>
> - Terraform
> - Infrastructure as Code (IaC)
> - Amazon VPC
> - Amazon EKS
> - Network Security Groups (AWS Security Groups)
> - Remote State Management

---

# Overview

This lab demonstrates how to provision a complete AWS infrastructure using **Terraform**. The deployment includes an Amazon VPC, private subnets, Security Groups, Amazon ECR, Amazon EKS, and a remote Terraform state backend using Amazon S3 and Amazon DynamoDB.

This implementation aligns with the JD requirement for using **Terraform/Bicep** to provision and manage cloud infrastructure, including AKS/EKS readiness, ACR/ECR, VNets/VPCs, NSGs/Security Groups, and secure state management.

---

# Lab Objective

Provision the following AWS infrastructure using Terraform:

- Amazon VPC
- Private Subnets
- Security Groups
- Amazon ECR Repository
- Amazon EKS Cluster
- Remote Terraform State Backend (Amazon S3 + DynamoDB)

---

# Architecture Components

| Component | Purpose |
|----------|---------|
| Amazon VPC | Isolated network for AWS resources |
| Private Subnets | Host Amazon EKS worker nodes |
| Security Groups | Control inbound and outbound network traffic |
| Amazon ECR | Store Docker container images |
| Amazon EKS | Managed Kubernetes cluster |
| Amazon S3 | Store Terraform remote state |
| Amazon DynamoDB | State locking to prevent concurrent Terraform operations |

---

# Step 1 – Create the Terraform Project Structure

## 1.1 Create the Project Layout

Create the following directory structure.

```text
terraform-aws/
│
├── main.tf                # Root module calls
├── variables.tf           # Input variables
├── outputs.tf             # Output values
├── backend.tf             # Remote state (Amazon S3 + DynamoDB)
├── provider.tf            # AWS provider configuration
│
└── modules/
    ├── vpc/               # VPC, Subnets, Internet Gateway, NAT Gateway
    ├── eks/               # Amazon EKS Cluster and Node Groups
    └── ecr/               # Amazon ECR Repositories
```

> [!TIP]
> Organizing Terraform into reusable modules improves maintainability, scalability, and code reuse across environments.

---

## 1.2 Configure the Remote State Backend

Create the following file.

```text
backend.tf
```

Configure Terraform to store the state file in Amazon S3 and use Amazon DynamoDB for state locking.

```hcl
terraform {

  backend "s3" {

    bucket         = "my-terraform-state-bucket"
    key            = "devops/eks/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"

  }

}
```

> [!IMPORTANT]
> Using a remote backend enables collaboration and prevents accidental loss or corruption of the Terraform state file.

---

# Step 2 – Create the Amazon VPC Module

## 2.1 Create the VPC, Private Subnets, Internet Gateway, and NAT Gateway

Create the following file.

```text
modules/vpc/main.tf
```

```hcl
resource "aws_vpc" "main" {

  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.cluster_name}-vpc"
  }

}

resource "aws_subnet" "private" {

  count = 2

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name                              = "private-${count.index}"
    "kubernetes.io/role/internal-elb" = "1"
  }

}
```

> [!NOTE]
> The private subnets are configured for internal load balancers and are intended to host Amazon EKS worker nodes.

---

# Step 3 – Create the Amazon EKS Module

## 3.1 Create the Amazon EKS Cluster

Create the following file.

```text
modules/eks/main.tf
```

```hcl
resource "aws_eks_cluster" "main" {

  name     = var.cluster_name
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.29"

  vpc_config {

    subnet_ids              = var.private_subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = true
    public_access_cidrs     = ["YOUR_IP/32"]

  }

  tags = {
    Environment = var.environment
  }

}

resource "aws_eks_node_group" "general" {

  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "general"
  node_role_arn   = aws_iam_role.eks_nodes.arn

  subnet_ids = var.private_subnet_ids

  instance_types = [
    "t3.medium"
  ]

  scaling_config {

    desired_size = 2
    max_size     = 5
    min_size     = 1

  }

}
```

> [!IMPORTANT]
> Replace **YOUR_IP/32** with your public IP address to restrict access to the Amazon EKS API server.

---

# Step 4 – Deploy the Infrastructure

## 4.1 Create the Remote State Resources

Run the following commands **once** before executing `terraform init`.

### Create the Amazon S3 Bucket

```bash
aws s3api create-bucket \
    --bucket my-terraform-state-bucket \
    --region ap-south-1 \
    --create-bucket-configuration LocationConstraint=ap-south-1
```

Enable bucket versioning.

```bash
aws s3api put-bucket-versioning \
    --bucket my-terraform-state-bucket \
    --versioning-configuration Status=Enabled
```

### Create the DynamoDB Table

```bash
aws dynamodb create-table \
    --table-name terraform-locks \
    --attribute-definitions AttributeName=LockID,AttributeType=S \
    --key-schema AttributeName=LockID,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST \
    --region ap-south-1
```

---

## 4.2 Initialize, Validate, Plan, and Apply

Execute the standard Terraform workflow.

```bash
terraform init
```

```bash
terraform validate
```

```bash
terraform plan -out=tfplan
```

```bash
terraform apply tfplan
```

---

## 4.3 Destroy the Infrastructure

When the lab is complete, remove the deployed resources to avoid unnecessary AWS charges.

```bash
terraform destroy -auto-approve
```

> [!WARNING]
> Running `terraform destroy` permanently removes all infrastructure managed by the current Terraform state.

---

# Validation

After completing this lab, verify the following:

- ✅ Amazon S3 remote state backend is configured.
- ✅ Amazon DynamoDB state locking is enabled.
- ✅ Amazon VPC and private subnets are created successfully.
- ✅ Amazon EKS cluster is provisioned.
- ✅ Amazon EKS managed node group is created.
- ✅ Terraform state is stored remotely.
- ✅ Infrastructure deployment completes without errors.

---

# Best Practice Tips

> [!TIP]

- Always use a **remote Terraform state**. Never store `terraform.tfstate` only on your local machine.
- Enable **Amazon DynamoDB state locking** to prevent concurrent `terraform apply` operations in collaborative environments.
- Use **Terraform Workspaces** or separate state files for **Development**, **Staging**, and **Production** environments.
- Pin provider versions using `required_providers` to prevent unexpected upgrades.
- Execute `terraform fmt` and `terraform validate` as part of the CI pipeline before applying infrastructure changes.
- Include the output of `terraform plan` in pull requests to allow infrastructure changes to be reviewed before deployment.

---

# Lab Summary

In this lab, you:

- Created a modular Terraform project structure.
- Configured a remote Terraform backend using Amazon S3 and Amazon DynamoDB.
- Provisioned an Amazon VPC and private networking components.
- Deployed an Amazon EKS cluster and managed node group.
- Applied Terraform best practices for remote state management, validation, and infrastructure lifecycle management.

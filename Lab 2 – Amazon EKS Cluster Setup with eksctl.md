# Lab 2 – Amazon EKS Cluster Setup with eksctl

> [!NOTE]
> **JD Skills Covered**
>
> - Kubernetes
> - EKS
> - eksctl
> - kubectl
> - Node Groups

---

# Overview

This lab demonstrates how to create and configure a production-ready Amazon EKS cluster using **eksctl**. You will install the required tools, provision an Amazon EKS cluster with managed node groups, configure **kubectl**, and verify cluster health using both the AWS Console and command-line tools.

---

# Lab Objective

Deploy and configure an Amazon EKS cluster using **eksctl (CLI)** and the **AWS Console**. Configure **kubectl**, set up **managed node groups**, and verify cluster health, aligning with the JD requirement for deploying and managing applications using Kubernetes.

---

# Step 1 – Install Required Tools (Local Machine / AWS Cloud9)

> [!IMPORTANT]
> Ensure all required tools are installed before creating the EKS cluster.

## 1.1 Install AWS CLI v2

Download and install AWS CLI v2.

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip
sudo ./aws/install
```

Configure the AWS CLI.

```bash
aws configure
```

Provide the following information when prompted:

- AWS Access Key ID
- AWS Secret Access Key
- Default Region: `ap-south-1`
- Output Format: `json`

---

## 1.2 Install eksctl

Install the Amazon EKS cluster management tool.

```bash
curl --silent --location \
"https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" \
| tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin
```

Verify the installation.

```bash
eksctl version
```

---

## 1.3 Install kubectl

Install the Kubernetes command-line interface.

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Verify the installation.

```bash
kubectl version --client
```

---

# Step 2 – Create an Amazon EKS Cluster Using eksctl

Create a configuration file named:

```text
cluster.yaml
```

Add the following configuration.

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: devops-cluster
  region: ap-south-1
  version: "1.29"

managedNodeGroups:
  - name: ng-general
    instanceType: t3.medium
    minSize: 1
    maxSize: 3
    desiredCapacity: 2
    volumeSize: 20

    ssh:
      allow: true

    labels:
      role: general

    tags:
      Environment: dev
      Team: devops

addons:
  - name: vpc-cni
  - name: coredns
  - name: kube-proxy
```

---

## 2.1 Create the Cluster

Run the following command.

> [!NOTE]
> Cluster creation typically takes **15–20 minutes**.

```bash
eksctl create cluster -f cluster.yaml
```

Monitor the deployment progress in the AWS CloudFormation Console.

---

## 2.2 Update kubeconfig

Configure **kubectl** to communicate with the new Amazon EKS cluster.

```bash
aws eks update-kubeconfig \
    --region ap-south-1 \
    --name devops-cluster
```

---

## 2.3 Verify the Cluster

Run the following commands to confirm that the cluster is healthy.

```bash
kubectl get nodes -o wide
```

```bash
kubectl get pods -A
```

```bash
kubectl cluster-info
```

> [!TIP]
> Verify that all nodes display a **Ready** status before proceeding to the next lab.

---

# Step 3 – Verify the Cluster in the AWS Console

## 3.1 Open the Amazon EKS Console

Navigate to:

**AWS Console → Elastic Kubernetes Service → Clusters → devops-cluster**

---

## 3.2 Verify the Managed Node Group

Navigate to:

**devops-cluster → Compute → Node groups → ng-general**

Verify:

- Status: **Active**

---

## 3.3 Verify Cluster Add-ons

Navigate to:

**devops-cluster → Add-ons**

Verify the following add-ons are installed and show an **Active** status:

- vpc-cni
- coredns
- kube-proxy

---

# Validation

After completing this lab, verify the following:

- ✅ Amazon EKS cluster created successfully.
- ✅ Managed node group is in **Active** status.
- ✅ kubectl is connected to the cluster.
- ✅ Worker nodes are in the **Ready** state.
- ✅ Core EKS add-ons are active.

---

# Best Practice Tips

> [!TIP]

- Use **managed node groups**—AWS automatically handles patching, updates, and node draining.
- Enable a **private API server endpoint** and restrict public access CIDRs for production clusters.
- Use **t3.medium** for development and testing; use **m5.large** or **c5.xlarge** for production workloads.
- Enable **CloudWatch Container Insights** during cluster creation (covered in **Lab 7**).
- Tag all AWS resources (for example, **Environment**, **Team**, and **CostCenter**) to improve billing visibility and governance.
- Use **eksctl** for repeatable, version-controlled cluster provisioning instead of manual configuration through the AWS Management Console.

---

# Lab Summary

In this lab, you:

- Installed AWS CLI, eksctl, and kubectl.
- Created an Amazon EKS cluster using eksctl.
- Configured kubectl connectivity.
- Verified cluster health.
- Validated managed node groups and cluster add-ons.
- Applied recommended AWS best practices for Amazon EKS deployments.

# Lab 7 – Monitoring with CloudWatch, Container Insights & Alerts

> [!NOTE]
> **JD Skills Covered**
>
> - CloudWatch
> - Container Insights
> - Log Analytics
> - Alarms
> - SNS

---

# Overview

This lab demonstrates how to configure complete observability for application pipelines and Kubernetes workloads using **Amazon CloudWatch**.

Amazon CloudWatch provides monitoring and logging capabilities equivalent to **Azure Monitor and Log Analytics** by collecting metrics, application logs, Kubernetes performance data, and operational events.

The lab covers:

- Amazon EKS Container Insights
- CloudWatch Logs
- Log Groups
- Metric Filters
- Dashboards
- CloudWatch Alarms
- Amazon SNS Notifications

---

# Lab Objective

Configure end-to-end observability for Kubernetes workloads running on Amazon EKS by:

- Enabling CloudWatch Container Insights.
- Collecting Kubernetes metrics and logs.
- Creating CloudWatch dashboards.
- Creating metric-based alarms.
- Sending notifications using Amazon SNS.

---

# Monitoring Architecture

| Component | Purpose |
|----------|---------|
| Amazon CloudWatch | Central monitoring and observability platform |
| Container Insights | Kubernetes cluster and workload metrics |
| CloudWatch Logs | Centralized log collection and analysis |
| Metric Filters | Convert log events into measurable metrics |
| CloudWatch Alarms | Detect operational issues |
| Amazon SNS | Send alert notifications |

---

# Step 1 – Enable CloudWatch Container Insights on Amazon EKS

## 1.1 Attach CloudWatch Policy to Node IAM Role

Find the Amazon EKS node group IAM role.

Navigate to:

**EC2 Console → IAM Roles**

Attach the following policy:

```text
CloudWatchAgentServerPolicy
```

Alternatively, use AWS CLI:

```bash
aws iam attach-role-policy \
--role-name <NODE_IAM_ROLE_NAME> \
--policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
```

> [!IMPORTANT]
> The CloudWatch agent requires IAM permissions to publish metrics and logs from Amazon EKS worker nodes.

---

## 1.2 Deploy CloudWatch Agent as DaemonSet

AWS provides a quick deployment manifest.

Set the cluster and region variables:

```bash
ClusterName=devops-cluster

RegionName=ap-south-1
```

Deploy the CloudWatch agent.

```bash
curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml \
| sed 's/{{cluster_name}}/'${ClusterName}'/;s/{{region_name}}/'${RegionName}'/g' \
| kubectl apply -f -
```

---

## 1.3 Verify Container Insights

Navigate to:

**AWS Console → CloudWatch → Container Insights → EKS Clusters → devops-cluster**

Review:

- CPU Utilization
- Memory Utilization
- Network I/O per Pod
- Network I/O per Node

> [!TIP]
> Container Insights provides Kubernetes-level visibility without requiring manual metric collection.

---

# Step 2 – Application Logs to CloudWatch Logs

## 2.1 Enable Amazon EKS Control Plane Logging

Navigate to:

**Amazon EKS → Cluster → Observability → Manage Logging**

Enable all log types:

- API Server
- Audit
- Authenticator
- Controller Manager
- Scheduler

---

## 2.2 View Logs in CloudWatch

Navigate to:

**CloudWatch → Log Groups**

Open:

```text
/aws/eks/devops-cluster/cluster
```

Example log filters:

Audit log query:

```text
{ $.sourceIPs = "*" }
```

Application error logs:

```text
error
```

---

## 2.3 Create a Log-Based Metric Filter

Navigate to:

**CloudWatch → Log Groups → Your Log Group → Metric Filters → Create**

Configure:

| Setting | Value |
|---------|-------|
| Filter Pattern | `[ERROR]` |
| Metric Name | `AppErrorCount` |
| Metric Value | `1` |
| Default Value | `0` |

> [!NOTE]
> Metric filters convert log patterns into CloudWatch metrics that can trigger alarms.

---

# Step 3 – Create CloudWatch Alarms and SNS Notifications

## 3.1 Create an SNS Topic for Alerts

Navigate to:

**AWS Console → SNS → Topics → Create Topic**

Configure:

| Setting | Value |
|---------|-------|
| Type | Standard |
| Name | devops-alerts |

Create a subscription:

| Setting | Value |
|---------|-------|
| Protocol | Email |
| Endpoint | your@email.com |

Confirm the subscription using the email confirmation message.

---

## 3.2 Create CPU Alarm for Amazon EKS Nodes

Navigate to:

**CloudWatch → Alarms → Create Alarm**

Select metric:

```text
EC2 → Per-Instance Metrics → CPUUtilization
```

Configure:

| Setting | Value |
|---------|-------|
| Threshold | Greater than 80% |
| Evaluation Periods | 2 consecutive periods |
| Action | Send notification to devops-alerts SNS topic |
| Alarm Name | EKS-Node-High-CPU |

---

## 3.3 Create Alarm Using AWS CLI

Create the alarm using AWS CLI.

```bash
aws cloudwatch put-metric-alarm \
--alarm-name "EKS-High-CPU" \
--alarm-description "EKS node CPU > 80%" \
--metric-name CPUUtilization \
--namespace AWS/EC2 \
--statistic Average \
--period 300 \
--threshold 80 \
--comparison-operator GreaterThanThreshold \
--evaluation-periods 2 \
--alarm-actions arn:aws:sns:ap-south-1:ACCT_ID:devops-alerts
```

---

# Step 4 – Create CloudWatch Dashboard

## 4.1 Create Dashboard Using AWS Console

Navigate to:

**CloudWatch → Dashboards → Create Dashboard**

Dashboard Name:

```text
DevOps-Dashboard
```

Add the following widgets.

| Widget Type | Metric |
|-------------|--------|
| Line | EKS Node CPUUtilization |
| Line | EKS Node MemoryUtilization |
| Number | AppErrorCount custom metric |
| Log Table | /aws/eks/devops-cluster/cluster |

Save the dashboard.

---

# Validation

After completing this lab, verify the following:

- ✅ CloudWatch agent is running as a Kubernetes DaemonSet.
- ✅ Container Insights metrics are visible.
- ✅ EKS control plane logs are available in CloudWatch Logs.
- ✅ Log-based metric filters are created successfully.
- ✅ SNS notifications are configured.
- ✅ CloudWatch alarms trigger based on defined thresholds.
- ✅ DevOps monitoring dashboard is available.

---

# Best Practice Tips

> [!TIP]

- Configure alarms for:
  - CPU utilization greater than **80%**
  - Memory utilization greater than **85%**
  - Pod restarts greater than **5/hour**
  - Error rate greater than **1%**

- Use **CloudWatch Synthetics (Canaries)** to continuously monitor application endpoints.
- Configure CloudWatch Log Group retention policies between **30–90 days** to control storage costs.
- Use CloudWatch Logs Insights for advanced queries.

Example:

```sql
fields @timestamp, @message
| filter @message like /ERROR/
```

- Create composite alarms combining CPU and Memory alarms to reduce alert noise.
- Export CloudWatch metrics to Grafana using the CloudWatch data source for enhanced visualization.

---

# Lab Summary

In this lab, you:

- Enabled Amazon CloudWatch Container Insights for Amazon EKS.
- Configured centralized Kubernetes logging.
- Created CloudWatch metric filters.
- Configured SNS-based alert notifications.
- Built CloudWatch dashboards.
- Implemented monitoring best practices for production Kubernetes workloads.

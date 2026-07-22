# Lab 4 – Deployment Strategies on Amazon EKS

> [!NOTE]
> **JD Skills Covered**
>
> - Rolling Update
> - Blue-Green Deployment
> - Canary Deployment
> - Amazon EKS
> - Kubernetes Services
> - kubectl

---

# Overview

This lab demonstrates how to implement three Kubernetes deployment strategies on Amazon EKS using native Kubernetes resources and **kubectl**.

The deployment strategies covered in this lab are:

- Rolling Update
- Blue-Green Deployment
- Canary Deployment

These deployment models directly align with the JD requirement for supporting **rolling**, **blue-green**, and **canary** deployment strategies.

---

# Lab Objective

Implement the three key deployment strategies (**Rolling Update**, **Blue-Green**, and **Canary**) on Amazon EKS using Kubernetes-native constructs.

---

# Deployment Strategy Comparison

| Strategy | Downtime | Rollback Speed | Traffic Control | Complexity |
|----------|----------|----------------|-----------------|------------|
| **Rolling Update** | Zero (gradual) | Slow (re-deploy) | None | Low |
| **Blue-Green** | Zero (instant cut) | Instant | All-or-nothing | Medium |
| **Canary** | Zero | Fast | Fine-grained (%) | Medium-High |

> [!TIP]
> Select the deployment strategy based on your application's availability requirements, rollback needs, and traffic management capabilities.

---

# Strategy A – Rolling Update (Default Kubernetes)

Rolling Updates replace Pods incrementally, ensuring application availability throughout the deployment.

---

## A.1 Create a Deployment with a Rolling Update Strategy

Create the following file:

```text
rolling-deploy.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: my-app
  namespace: dev

spec:
  replicas: 3

  strategy:
    type: RollingUpdate

    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  selector:
    matchLabels:
      app: my-app

  template:
    metadata:
      labels:
        app: my-app

    spec:
      containers:
        - name: my-app
          image: 123456789.dkr.ecr.ap-south-1.amazonaws.com/my-app:v1

          ports:
            - containerPort: 80
```

> [!NOTE]
>
> - **maxSurge: 1** allows one additional Pod during the deployment.
> - **maxUnavailable: 0** ensures no Pods become unavailable during the update.

---

## A.2 Apply and Monitor the Rollout

Deploy the application.

```bash
kubectl apply -f rolling-deploy.yaml
```

Monitor the rollout.

```bash
kubectl rollout status deployment/my-app -n dev
```

---

## A.3 Trigger a Rolling Update

Update the container image.

```bash
kubectl set image deployment/my-app \
my-app=123456789.dkr.ecr.ap-south-1.amazonaws.com/my-app:v2 \
-n dev
```

Monitor the rollout.

```bash
kubectl rollout status deployment/my-app -n dev
```

Roll back if required.

```bash
kubectl rollout undo deployment/my-app -n dev
```

---

# Strategy B – Blue-Green Deployment

Blue-Green Deployment runs two identical environments simultaneously, allowing instant traffic switching between application versions.

---

## B.1 Deploy Blue (v1) and Green (v2)

Create the Blue deployment.

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: my-app-blue
  namespace: dev

spec:
  replicas: 3

  selector:
    matchLabels:
      app: my-app
      version: blue

  template:
    metadata:
      labels:
        app: my-app
        version: blue

    spec:
      containers:
        - name: my-app
          image: my-app:v1
```

---

## B.2 Create the Service Pointing to Blue

The Kubernetes Service determines which deployment receives production traffic.

```yaml
apiVersion: v1
kind: Service

metadata:
  name: my-app-svc
  namespace: dev

spec:
  selector:
    app: my-app
    version: blue

  ports:
    - port: 80
      targetPort: 80

  type: LoadBalancer
```

> [!NOTE]
> The Service selector currently routes all traffic to the **Blue** deployment.

---

## B.3 Deploy Green (v2) and Switch Traffic

Deploy the Green version.

```bash
kubectl apply -f green-deploy.yaml
```

Verify that both deployments are running.

```bash
kubectl get pods -n dev
```

Switch production traffic to Green.

```bash
kubectl patch service my-app-svc \
-n dev \
-p '{"spec":{"selector":{"version":"green"}}}'
```

Remove the Blue deployment after validation.

```bash
kubectl delete deployment my-app-blue -n dev
```

> [!IMPORTANT]
> Validate application functionality before removing the Blue deployment.

---

# Strategy C – Canary Deployment

Canary Deployment gradually exposes a new application version to a subset of users before full rollout.

---

## C.1 Run Stable (90%) and Canary (10%) Deployments

Deploy two application versions simultaneously.

```text
stable-deploy.yaml
```

- spec.replicas: 9
- label version: stable

```text
canary-deploy.yaml
```

- spec.replicas: 1
- label version: canary
- image: v2

> [!NOTE]
> A 9:1 replica ratio approximates **90% Stable** and **10% Canary** traffic.

---

## C.2 Create a Single Service

The Service routes traffic to both deployments using the common application label.

```yaml
spec:
  selector:
    app: my-app
```

> [!NOTE]
> The selector targets the shared **app** label rather than the **version** label.

---

## C.3 Gradually Increase Canary Traffic

Increase Canary traffic after validating application health and monitoring metrics.

Increase Canary to approximately **30%**.

```bash
kubectl scale deployment my-app-canary --replicas=3 -n dev

kubectl scale deployment my-app-stable --replicas=7 -n dev
```

Complete the rollout.

```bash
kubectl scale deployment my-app-canary --replicas=10 -n dev

kubectl delete deployment my-app-stable -n dev
```

> [!IMPORTANT]
> Increase traffic gradually while monitoring application health, latency, and error rates.

---

# Validation

After completing this lab, verify the following:

- ✅ Rolling Update completes without downtime.
- ✅ Blue and Green deployments run simultaneously.
- ✅ Kubernetes Service successfully switches traffic between Blue and Green.
- ✅ Stable and Canary deployments coexist.
- ✅ Traffic successfully shifts from Stable to Canary.
- ✅ Rollback procedures function as expected.

---

# Best Practice Tips

> [!TIP]

- Configure **readinessProbe** and **livenessProbe** before implementing Rolling Updates or Canary deployments.
- Use **AWS App Mesh** or a service mesh such as **Istio** for advanced percentage-based traffic routing.
- Blue-Green deployments are well suited for stateless applications requiring instant rollback.
- Canary deployments are ideal for validating higher-risk changes with a subset of production traffic.
- Monitor error rates, latency, CPU utilization, and application metrics before increasing Canary traffic.
- Configure **PodDisruptionBudget** to maintain application availability during node maintenance.
- Apply consistent Kubernetes labels such as **app**, **version**, and **env** to simplify deployment management and traffic routing.

---

# Lab Summary

In this lab, you:

- Implemented Kubernetes Rolling Updates.
- Performed Blue-Green deployments using Kubernetes Services.
- Configured Canary deployments using replica-based traffic distribution.
- Executed deployment rollbacks.
- Applied Kubernetes deployment best practices for production workloads on Amazon EKS.

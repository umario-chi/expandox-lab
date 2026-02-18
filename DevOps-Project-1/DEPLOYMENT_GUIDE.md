# Payday Platform: GitOps Deployment Guide

This document provides a step-by-step guide to deploying the `mini-finance-app` using a declarative GitOps workflow powered by Argo CD. This setup mirrors the clean, simple, and declarative approach demonstrated in the `daascohort3` learning modules.

---

### **Core Principles**

*   **Git is the Source of Truth:** All configurations, including Kubernetes manifests and Argo CD applications, are stored in this Git repository.
*   **Declarative:** We define the *desired state* of our application in YAML files, and Argo CD makes the cluster match that state.
*   **Automated:** Changes are automatically synchronized to the cluster after they are pushed to the correct Git branch.

---

## **1. Prerequisites**

Before you begin, ensure you have the following tools installed and configured:

1.  **`kubectl`**: Connected to a running Kubernetes cluster.
    *   *This guide assumes you are using a local `kind` cluster named `mini-fin-cluster`.*
2.  **Docker**: Running locally to build and push the application image.
3.  **Git**: Configured with your user credentials.
4.  **Docker Hub Account**: You will need your username and a Personal Access Token to push a private image.

---

## **2. One-Time Cluster Setup**

These steps only need to be performed once to prepare your Kubernetes cluster.

### **Step 2.1: Install Argo CD**

First, create a dedicated namespace for Argo CD and install the latest stable version.

```bash
# Create the namespace
kubectl create namespace argocd

# Apply the stable Argo CD installation manifest
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### **Step 2.2: Add Your Git Repository to Argo CD**

Register your public Git repository with Argo CD by applying a declarative manifest. This allows Argo CD to access your configuration files.

```bash
kubectl apply -f DevOps-Project-1/gitops/v2-add-cluster-and-repo/add-public-repo.yaml
```

---

## **3. Application Deployment Workflow**

This is the main workflow for deploying the `mini-finance-app`.

### **Step 3.1: Build and Push the Docker Image**

The application is a static website served by Nginx. We need to build the container image and push it to a registry (Docker Hub).

```bash
# 1. Navigate to the application's source directory
cd DevOps-Project-1/mini-finance-app

# 2. Build the Docker image
# Replace 'lakunzy' with your Docker Hub username if different
docker build -t lakunzy/mini-finance-app:latest .

# 3. Log in to Docker Hub (use an Access Token for the password)
docker login

# 4. Push the image to the registry
docker push lakunzy/mini-finance-app:latest
```

### **Step 3.2: Create the Image Pull Secret**

Since the image is private, we must provide Kubernetes with the credentials to pull it from Docker Hub.

*   **Note:** You only need to do this once per namespace.

```bash
# Create the secret for the 'dev' environment
kubectl create secret docker-registry dockerhub-creds \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<your-docker-username> \
  --docker-password=<your-docker-hub-token> \
  --namespace=mini-finance-dev

# Create the secret for the 'production' environment
kubectl create secret docker-registry dockerhub-creds \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<your-docker-username> \
  --docker-password=<your-docker-hub-token> \
  --namespace=mini-finance-prod
```

### **Step 3.3: Define the Application Project**

Apply the `AppProject` manifest. This logically groups our finance applications within Argo CD and sets governance rules.

```bash
kubectl apply -f DevOps-Project-1/gitops/v5-app-project/app-project.yaml
```

### **Step 3.4: Deploy with the ApplicationSet**

This is the final step. Apply the `ApplicationSet` manifest. This single file tells Argo CD to create and manage the applications for **both** your `dev` and `production` environments automatically.

```bash
kubectl apply -f DevOps-Project-1/gitops/v6-application-sets/appset.yaml
```

Argo CD will now automatically:
1.  Create the `mini-finance-dev` and `mini-finance-prod` namespaces.
2.  Read the manifests from `DevOps-Project-1/k8s/dev` and `DevOps-Project-1/k8s/production`.
3.  Deploy the resources (Deployment, Service, etc.) into the respective namespaces.

---

## **4. Verifying the Deployment**

### **Check Argo CD**

Check the Argo CD UI or use the CLI to see the status of your newly created applications.

```bash
# List the applications created by the ApplicationSet
kubectl get applications -n argocd
```
You should see `mini-finance-dev` and `mini-finance-production`, and their status should eventually become `Healthy` and `Synced`.

### **Check the Pods**

Verify that the application pods are running in their respective namespaces.

```bash
# Check the dev environment
kubectl get pods -n mini-finance-dev

# Check the production environment
kubectl get pods -n mini-finance-prod
```
The status should be `Running`. If it shows `ImagePullBackOff`, it means the image pull secret is incorrect or was not created.

### **Access the Application**

Use `port-forward` to access the running application from your local machine.

```bash
# Access the dev environment on http://localhost:8080
kubectl port-forward svc/mini-finance-app-service -n mini-finance-dev 8080:80

# Access the production environment on http://localhost:8081
kubectl port-forward svc/mini-finance-app-service -n mini-finance-prod 8081:80
```

---

## **5. Making Changes (The GitOps Way)**

To make any change—whether it's updating the image tag, changing the number of replicas, or modifying a config map—**you must update the YAML files in Git.**

1.  **Edit** the relevant manifest file in the `DevOps-Project-1/k8s/` directory.
2.  **Commit** the change with a descriptive message: `git commit -m "feat: increase production replicas to 4"`.
3.  **Push** the change to the repository: `git push origin main`.

Argo CD will detect the change and automatically synchronize the cluster to match the new state defined in Git.

---

## **6. Future Integrations (Coming Soon)**

This guide provides the foundation for a robust GitOps deployment. The following components will be integrated in the future.

### **Observability (Planned)**

*   **Metrics:** Prometheus will be installed to scrape metrics from the application and cluster components.
*   **Dashboards:** Grafana will be used to create dashboards for visualizing application health, performance, and SLOs.
*   **Logging:** A centralized logging stack (like Loki or the ELK stack) will be configured to aggregate logs from all pods.

*This section will be updated with instructions on how to configure and view observability data.*

### **Security (Planned)**

*   **Vulnerability Scanning:** The CI/CD pipeline will be enhanced to scan container images for vulnerabilities before they are pushed to the registry.
*   **Policy Enforcement:** Tools like Kyverno or OPA Gatekeeper will be integrated to enforce security policies on all manifests before they are applied to the cluster.
*   **Network Policies:** Stricter `NetworkPolicy` manifests will be added to limit communication between pods, following the principle of least privilege.

*This section will be updated with details on security configurations and best practices.*
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

### **Step 3.1: Automated Build and Push with GitHub Actions (Recommended)**

The recommended approach is to use **GitHub Actions** to automatically build and push your Docker image whenever you push code to the repository. This eliminates manual image building and ensures consistency.

#### **One-Time Setup: Configure GitHub Secrets**

1. Go to your GitHub repository: `https://github.com/lakunzy7/expandox-lab`
2. Click **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret** and add:
   - **Name:** `DOCKER_USERNAME`
   - **Secret:** Your Docker Hub username (e.g., `lakunzy`)
4. Click **Add secret**
5. Click **New repository secret** again and add:
   - **Name:** `DOCKERHUB_TOKEN`
   - **Secret:** Your Docker Hub Personal Access Token
     - *To create a token: Go to Docker Hub → Settings → Security → New Access Token*

#### **How the Automation Works**

The workflow file at `.github/workflows/ci.yml` automatically:
- Triggers on every push to any branch
- Builds your Docker image from `DevOps-Project-1/mini-finance-app`
- Pushes to Docker Hub only from `main` and `develop` branches
- Updates your Kubernetes manifests with the new image SHA
- Commits and pushes the manifest changes back to the repository

**That's it!** Just push your code and the workflow handles the rest. You can monitor progress in the **Actions** tab on GitHub.

#### **Manual Build and Push (Alternative)**

If you prefer to build and push manually without GitHub Actions:

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

Since the Docker image is private, Kubernetes needs credentials to pull it from Docker Hub. Create the same secret in both namespaces.

*   **Note:** You only need to do this once per namespace.
*   **Important:** Use the same Docker Hub token you configured in GitHub Secrets (Step 3.1).

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

**Example with actual values:**
```bash
kubectl create secret docker-registry dockerhub-creds \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=lakunzy \
  --docker-password=dckr_pat_xxxxxxxxxxxxxxxx \
  --namespace=mini-finance-dev
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

The GitOps workflow involves two types of changes:

### **5.1: Application Code Changes (Automatic via GitHub Actions)**

When you update the application code and push to `main` or `develop`:

1. **Push** your code to the repository: `git push origin develop`
2. **GitHub Actions automatically:**
   - Builds a new Docker image
   - Pushes it to Docker Hub with the commit SHA and `latest` tag
   - Updates the Kubernetes manifest files with the new image SHA
   - Commits and pushes the manifest updates back to the repository
3. **Argo CD detects** the manifest change and synchronizes the cluster

**Example workflow:**
```bash
# 1. Make changes to your application code
# 2. Commit and push
git add .
git commit -m "feat: add new dashboard feature"
git push origin develop

# GitHub Actions now:
# - Builds: lakunzy/mini-finance-app:abc123def456
# - Pushes to Docker Hub
# - Updates DevOps-Project-1/k8s/dev/manifests.yaml with new image SHA
# - Argo CD auto-syncs the new image to your cluster
```

### **5.2: Infrastructure/Configuration Changes (Manual)**

To make infrastructure changes—such as updating replicas, changing resources, or modifying config maps—**you must update the YAML files in Git directly.**

1.  **Edit** the relevant manifest file in the `DevOps-Project-1/k8s/` directory.
2.  **Commit** the change with a descriptive message: `git commit -m "feat: increase production replicas to 4"`.
3.  **Push** the change to the repository: `git push origin main`.

Argo CD will detect the change and automatically synchronize the cluster to match the new state defined in Git.

**Example:**
```bash
# Edit the production manifest
vim DevOps-Project-1/k8s/production/manifests.yaml

# Update replicas: 1 → 3

# Commit and push
git add DevOps-Project-1/k8s/production/manifests.yaml
git commit -m "ops: increase production replicas from 1 to 3"
git push origin main

# Argo CD auto-syncs the new replica count
```

---

## **6. Monitoring the CI/CD Pipeline**

### **GitHub Actions**

Monitor the automated build and deploy process:
1. Go to your repository on GitHub
2. Click the **Actions** tab
3. Select **CI - Build, Push, and Update Manifests**
4. View the latest workflow run status
5. Click on a run to see detailed logs for each step

**What to look for:**
- ✅ Checkout code
- ✅ Build image (should complete in 30-60 seconds)
- ✅ Login to Docker Hub
- ✅ Push image to Docker Hub
- ✅ Update manifest and commit

### **Argo CD**

Monitor the cluster synchronization:
```bash
# View all applications
kubectl get applications -n argocd

# View detailed status of dev environment
kubectl describe application mini-finance-dev -n argocd

# View detailed status of production environment
kubectl describe application mini-finance-production -n argocd
```

Check the Argo CD UI or CLI to ensure:
- Status: `Healthy`
- Sync Status: `Synced`
- The image tag in the deployment matches the latest commit SHA

---

## **7. Troubleshooting**

### **GitHub Actions Workflow Fails**

**Error: "Username and password required"**
- Verify that both `DOCKER_USERNAME` and `DOCKERHUB_TOKEN` secrets are set in GitHub Settings
- Ensure the secret names in GitHub match the ones referenced in `.github/workflows/ci.yml`

**Error: "Permission to repository denied"**
- This is expected if the workflow tries to push manifests
- Verify that the workflow has `permissions: contents: write` configured
- Check that `GITHUB_TOKEN` is being used for git push operations

### **Image Pull Backoff in Kubernetes**

```bash
kubectl get pods -n mini-finance-dev
# Status: ImagePullBackOff
```

**Solutions:**
1. Verify the `dockerhub-creds` secret exists:
   ```bash
   kubectl get secret dockerhub-creds -n mini-finance-dev
   ```
2. Verify credentials are correct:
   ```bash
   kubectl get secret dockerhub-creds -n mini-finance-dev -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq
   ```
3. Check if the Docker username/token in Kubernetes matches GitHub secrets

### **Argo CD Not Syncing**

1. Verify the git repository secret is configured:
   ```bash
   kubectl get secret expandox-lab-repo -n argocd
   ```
2. Check Argo CD application status:
   ```bash
   kubectl get application mini-finance-dev -n argocd
   ```
3. View detailed sync status:
   ```bash
   kubectl describe application mini-finance-dev -n argocd
   ```

---

## **8. Future Integrations (Coming Soon)**

This guide provides the foundation for a robust GitOps deployment with automated CI/CD. The following components will be integrated in the future.

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
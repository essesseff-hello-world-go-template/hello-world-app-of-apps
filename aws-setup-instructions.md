# EKS Setup for essesseff Apps

## Overview

This guide assumes you have already created an essesseff app from a template, which automatically sets up the 6-repository structure:

- `{app-name}` - Source code repository
- `{app-name}-config-dev` - Development environment configuration (Helm chart + values.yaml)
- `{app-name}-config-qa` - QA environment configuration (Helm chart + values.yaml)
- `{app-name}-config-staging` - Staging environment configuration (Helm chart + values.yaml)
- `{app-name}-config-prod` - Production environment configuration (Helm chart + values.yaml)
- `{app-name}-app-of-apps` - Argo CD app-of-apps definitions (contains environment Application YAMLs)

**All repositories are already populated** with content from the essesseff template, including:
- Helm charts with generic templates
- Environment-specific values.yaml files
- Argo CD Application definitions
- All necessary configuration files

You only need to set up the EKS infrastructure and connect Argo CD to deploy your app.

### Architecture

- **Deployment Model**: Trunk-based development (single `main` branch)
- **Auto-Deploy**: DEV only (via essesseff webhooks)
- **Manual Deploy**: QA, STAGING, PROD (via essesseff UI with RBAC)
- **GitOps**: All deployments managed by Argo CD with automated sync

## Prerequisites

- ✅ essesseff app created from template (6 repositories exist and are populated)
- AWS EKS cluster created
- `kubectl` installed
- `aws` CLI configured
- GitHub Personal Access Token with `repo` and `read:packages` scopes

## Step 1: Configure EKS Endpoint (Choose One)

### Option A: Private Endpoint Only (Recommended for Production)

EKS API server accessible only from within VPC. Requires VPN/bastion to run kubectl/helm commands.

```bash
# Set cluster to private endpoint only
aws eks update-cluster-config \
  --name your-cluster-name \
  --region us-east-1 \
  --resources-vpc-config endpointPrivateAccess=true,endpointPublicAccess=false

# Wait for update (5-10 minutes)
aws eks wait cluster-active --name your-cluster-name --region us-east-1

# Connect to VPC (VPN/bastion), then:
aws eks update-kubeconfig --name your-cluster-name --region us-east-1
kubectl get nodes  # Verify access
```

### Option B: Public Endpoint (Simpler for Development)

EKS API server accessible from internet. Can run kubectl/helm locally without VPN.

```bash
# Keep public endpoint (default)
aws eks update-kubeconfig --name your-cluster-name --region us-east-1
kubectl get nodes  # Verify access
```

**Note:** You can run helm/kubectl locally with public endpoint. Private endpoint requires VPN/bastion because the EKS API server is only reachable from within the VPC.

## Step 2: Install AWS Load Balancer Controller

The AWS Load Balancer Controller is required for creating Application Load Balancers (ALBs) from Kubernetes Ingress resources.

```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=your-cluster-name \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller

# Verify installation
kubectl get deployment aws-load-balancer-controller -n kube-system
```

## Step 3: Install Argo CD

Argo CD is the GitOps tool that will sync your repositories to Kubernetes.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for Argo CD to be ready (2-3 minutes)
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Get initial admin password (save this securely)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

## Step 4: Configure Argo CD Repository Secrets

Argo CD needs access to your GitHub repositories. Since your essesseff app already has 6 repositories, you need to create secrets for all of them.

**Replace `{app-name}` and `{github-org}` with your actual app name and GitHub organization.**

### 4.1: Create Repository Secret File

Create a file `argocd-repository-secret.yaml` (DO NOT COMMIT THIS FILE):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {app-name}-app-of-apps-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/{github-org}/{app-name}-app-of-apps
  password: <your-github-token>  # Replace with actual token
  username: essesseff-admin
---
apiVersion: v1
kind: Secret
metadata:
  name: {app-name}-config-dev-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/{github-org}/{app-name}-config-dev
  password: <your-github-token>  # Replace with actual token
  username: essesseff-admin
---
apiVersion: v1
kind: Secret
metadata:
  name: {app-name}-config-qa-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/{github-org}/{app-name}-config-qa
  password: <your-github-token>  # Replace with actual token
  username: essesseff-admin
---
apiVersion: v1
kind: Secret
metadata:
  name: {app-name}-config-staging-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/{github-org}/{app-name}-config-staging
  password: <your-github-token>  # Replace with actual token
  username: essesseff-admin
---
apiVersion: v1
kind: Secret
metadata:
  name: {app-name}-config-prod-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/{github-org}/{app-name}-config-prod
  password: <your-github-token>  # Replace with actual token
  username: essesseff-admin
```

### 4.2: Apply Repository Secrets

```bash
# Replace <your-github-token> with your actual GitHub Personal Access Token
# Token needs: repo (Full control) and read:packages (Download packages) scopes
# Replace {app-name} and {github-org} placeholders in the YAML file

# Apply secrets
kubectl apply -f argocd-repository-secret.yaml

# Verify secrets were created
kubectl get secrets -n argocd | grep {app-name}

# IMPORTANT: Delete the file after applying to prevent accidental commits
rm argocd-repository-secret.yaml
```

### 4.3: Verify Repository Access

```bash
# Check if Argo CD can access repositories
kubectl get applications -n argocd

# Or use Argo CD CLI (if installed)
argocd repo list
```

## Step 5: Create App-of-Apps Root Application

The essesseff template already created the environment Application YAMLs in your `{app-name}-app-of-apps` repository. You just need to create the root Application that manages them.

**Replace `{app-name}` and `{github-org}` with your actual values.**

### 5.1: Create Root Application

Create `app-of-apps-root.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: {app-name}-app-of-apps
  namespace: argocd
  labels:
    app: {app-name}
    essesseff: "true"
spec:
  project: default
  source:
    repoURL: https://github.com/{github-org}/{app-name}-app-of-apps
    targetRevision: main
    path: argocd  # Path to environment Application YAMLs (created by essesseff template)
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 5.2: Apply Root Application

```bash
# Replace {app-name} and {github-org} in the YAML file, then:
kubectl apply -f app-of-apps-root.yaml

# Verify root Application is created
kubectl get application {app-name}-app-of-apps -n argocd

# Watch for sync status
kubectl get application {app-name}-app-of-apps -n argocd -w
```

### 5.3: Verify Environment Applications

The root Application will automatically discover and create environment Applications (dev, qa, staging, prod) from the YAMLs in your `{app-name}-app-of-apps/argocd/` directory:

```bash
# List all Applications
kubectl get applications -n argocd

# Check specific environment Applications (created automatically)
kubectl get application {app-name}-dev -n argocd
kubectl get application {app-name}-qa -n argocd
kubectl get application {app-name}-staging -n argocd
kubectl get application {app-name}-prod -n argocd

# View Application details
kubectl describe application {app-name}-dev -n argocd
```

## Step 6: App-of-Apps Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│ Root Application ({app-name}-app-of-apps)               │
│ Points to: {app-name}-app-of-apps repo                  │
│ Path: argocd/ (contains environment Application YAMLs)  │
└──────────────────┬──────────────────────────────────────┘
                   │
                   ├─── Discovers & Creates ───┐
                                               │
        ┌──────────┴──────────┐                │
        │                     │                │
┌───────▼────────┐  ┌─────────▼────┐  ┌──────────────┐
│ Dev Application│  │ QA Application│  │ ... (others) │
│ Points to:      │  │ Points to:     │  │              │
│ config-dev repo │  │ config-qa repo │  │              │
│ Deploys:        │  │ Deploys:       │  │              │
│ Helm chart      │  │ Helm chart     │  │              │
│ from values.yaml│  │ from values.yaml│  │              │
└─────────────────┘  └────────────────┘  └──────────────┘
```

**Flow:**
1. Root Application syncs from `{app-name}-app-of-apps/argocd/` directory (populated by essesseff template)
2. Root Application discovers environment Application YAMLs and creates them
3. Each environment Application syncs from its config repo (Helm chart + values.yaml from essesseff template)
4. Each environment Application deploys Helm chart with environment-specific values.yaml
5. Helm chart creates Kubernetes resources (Deployments, Services, Ingress, etc.)

## Step 7: Verify Ingress Configuration

The essesseff template already configured your `values.yaml` files with AWS ALB ingress settings. Verify the configuration in each config repository:

**In `{app-name}-config-{env}/values.yaml`:**

```yaml
ingress:
  enabled: true
  className: "alb"
  annotations:
    alb.ingress.kubernetes.io/scheme: internal  # Or 'internet-facing'
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/group.name: {app-name}
    alb.ingress.kubernetes.io/group.order: '1'  # Increment per env (1=dev, 2=qa, 3=staging, 4=prod)
  hosts:
    - host: example.com  # Update with your domain
      paths:
        - path: /{app-name}-{env}  # Path-based routing
          pathType: Prefix
```

**Update as needed:**
- Replace `example.com` with your actual domain
- Verify `group.order` values are correct (1=dev, 2=qa, 3=staging, 4=prod)
- Adjust `scheme` to `internet-facing` if needed

## Step 8: Port Forward to Services

After Argo CD deploys your apps (wait for Applications to sync and deployments to be ready):

```bash
# Check Application sync status
kubectl get applications -n argocd

# Check if deployments are ready
kubectl get deployments -n {app-name}-dev
kubectl get deployments -n {app-name}-qa

# Forward dev environment
kubectl port-forward service/{app-name}-dev 8080:80 -n {app-name}-dev

# Forward Argo CD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access:
# - App: http://localhost:8080
# - Argo CD: https://localhost:8080
```

**Note:** Service port is 80 (not 8080) because the service exposes port 80, which forwards to container port 8080.

## Step 9: Quick Port Forward Script

```bash
#!/bin/bash
# port-forward.sh - Port forward all environments
# Replace {app-name} with your actual app name

kubectl port-forward service/{app-name}-dev 8080:80 -n {app-name}-dev &
kubectl port-forward service/{app-name}-qa 8081:80 -n {app-name}-qa &
kubectl port-forward service/{app-name}-staging 8082:80 -n {app-name}-staging &
kubectl port-forward service/{app-name}-prod 8083:80 -n {app-name}-prod &

echo "Port forwarding active:"
echo "  - Dev:      http://localhost:8080"
echo "  - QA:       http://localhost:8081"
echo "  - Staging:  http://localhost:8082"
echo "  - Prod:     http://localhost:8083"
echo ""
echo "Press Ctrl+C to stop."
wait
```

## Step 10: Verification

```bash
# Test cluster access
kubectl get nodes

# Check Argo CD
kubectl get pods -n argocd

# Check Applications status
kubectl get applications -n argocd
kubectl get application {app-name}-app-of-apps -n argocd -o yaml

# Check environment Applications
kubectl get applications -n argocd -l app={app-name}

# Check deployments in each environment
kubectl get deployments -n {app-name}-dev
kubectl get deployments -n {app-name}-qa
kubectl get deployments -n {app-name}-staging
kubectl get deployments -n {app-name}-prod

# Check pods
kubectl get pods -n {app-name}-dev
kubectl get pods -n {app-name}-qa

# Check services
kubectl get services -n {app-name}-dev
kubectl get ingress -n {app-name}-dev

# Test port forward
kubectl port-forward service/{app-name}-dev 8080:80 -n {app-name}-dev
# In another terminal: curl http://localhost:8080/health

# Check Argo CD sync status via CLI (if installed)
argocd app list
argocd app get {app-name}-dev
```

## Troubleshooting

### Applications Not Syncing

```bash
# Check Application sync status
kubectl get application {app-name}-dev -n argocd -o yaml

# Check for sync errors
kubectl describe application {app-name}-dev -n argocd

# Check if repository secrets are correct
kubectl get secrets -n argocd | grep {app-name}

# Manually trigger sync (if needed)
kubectl patch application {app-name}-dev -n argocd --type merge -p '{"operation":{"sync":{"syncStrategy":{"hook":{}}}}}'
```

### Repository Access Issues

```bash
# Verify repository secrets exist
kubectl get secrets -n argocd | grep {app-name}

# Check repository connection
argocd repo list

# Test repository access
argocd repo get https://github.com/{github-org}/{app-name}-config-dev

# Verify GitHub token has correct scopes (repo, read:packages)
```

### Ingress Not Creating ALB

```bash
# Check ingress controller logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# Check ingress resource
kubectl get ingress -n {app-name}-dev -o yaml

# Verify AWS Load Balancer Controller is running
kubectl get pods -n kube-system | grep aws-load-balancer-controller

# Check for IAM permissions issues
kubectl logs -n kube-system deployment/aws-load-balancer-controller | grep -i error
```

### Helm Chart Issues

```bash
# Check what Helm is trying to deploy
kubectl get application {app-name}-dev -n argocd -o yaml | grep -A 20 "helm:"

# Verify values.yaml syntax
helm template {app-name}-dev ./path/to/chart -f ./path/to/values.yaml --debug

# Check for missing values
kubectl describe application {app-name}-dev -n argocd | grep -i error
```

## Step 11: Deployment Process

Once your infrastructure is set up, deployments are managed through essesseff with the following workflows:

### Automatic Deployment (DEV only)

DEV environment automatically deploys on code pushes:

1. **Push code** to `main` branch in `{app-name}` repository
2. **GitHub Actions** builds container image and pushes to GitHub Container Registry (GHCR)
3. **essesseff webhook** receives build completion event
4. **essesseff automatically** updates `{app-name}-config-dev/values.yaml` with new image tag
5. **Argo CD syncs** DEV Application (auto-sync enabled)
6. **Deployment completes** - new version running in DEV

**No manual intervention required** - fully automated from code push to deployment.

### Manual Deployment (QA, STAGING, PROD)

QA, STAGING, and PROD require manual approval through essesseff UI:

#### QA Deployment Process

1. **Developer** declares Release Candidate (RC) in essesseff UI
2. **QA Engineer** reviews and accepts RC in essesseff UI
3. **essesseff** updates `{app-name}-config-qa/values.yaml` with RC image tag
4. **Argo CD syncs** QA Application (auto-sync enabled)
5. **QA Engineer** tests the deployment
6. **QA Engineer** marks image as Stable when ready

#### STAGING Deployment Process

1. **Release Engineer** selects Stable image in essesseff UI
2. **essesseff** updates `{app-name}-config-staging/values.yaml` with Stable image tag
3. **Argo CD syncs** STAGING Application (auto-sync enabled)
4. **Release Engineer** validates staging deployment

#### PROD Deployment Process

1. **Release Engineer** selects Stable image for PROD in essesseff UI
2. **OTP verification required** - essesseff prompts for one-time password
3. **After OTP approval**, essesseff updates `{app-name}-config-prod/values.yaml` with Stable image tag
4. **Argo CD syncs** PROD Application (auto-sync enabled)
5. **Deployment completes** - new version running in PROD

### RBAC Enforcement

essesseff enforces role-based access control:

- **Developers**: Can declare Release Candidates, cannot deploy
- **QA Engineers**: Can accept RCs and deploy to QA, can mark images as Stable
- **Release Engineers**: Can deploy to STAGING and PROD (PROD requires OTP)
- **DevOps Engineers**: Full access to all environments

### Repository URLs

Your essesseff app repositories:

- **Source**: `https://github.com/{github-org}/{app-name}`
- **Config DEV**: `https://github.com/{github-org}/{app-name}-config-dev`
- **Config QA**: `https://github.com/{github-org}/{app-name}-config-qa`
- **Config STAGING**: `https://github.com/{github-org}/{app-name}-config-staging`
- **Config PROD**: `https://github.com/{github-org}/{app-name}-config-prod`
- **App-of-Apps**: `https://github.com/{github-org}/{app-name}-app-of-apps` (this repo)

## Step 12: essesseff Integration

This setup requires the essesseff platform for deployment orchestration:

### essesseff Features

- **Event-driven promotions**: Automatic DEV deployments on code push
- **RBAC enforcement**: Role-based access control for deployments
- **Approval workflows**: Manual approvals for QA/STAGING/PROD
- **Deployment policies**: Enforced promotion paths (RC → QA → Stable → STAGING → PROD)
- **Audit trail**: Complete history of all deployments and approvals
- **OTP protection**: One-time password required for PROD deployments

### How essesseff Works with Argo CD

1. **essesseff manages** image lifecycle and promotion decisions
2. **essesseff updates** `values.yaml` files in config repos with approved image tags
3. **Argo CD detects** changes via Git polling
4. **Argo CD syncs** Applications automatically (auto-sync enabled)
5. **Kubernetes resources** are updated with new image versions

### essesseff UI Workflow

- **Image Lifecycle**: View and manage container images through lifecycle states (BUILD → DEV → RC → QA_TESTING → STABLE → STAGING → PROD)
- **Deployment Actions**: Declare RCs, accept RCs, deploy to environments
- **Approval Gates**: OTP verification for PROD, role-based permissions
- **History & Audit**: Complete deployment history with user actions

## Summary

Your essesseff app is now deployed with:

1. ✅ EKS cluster with proper endpoint configuration
2. ✅ AWS Load Balancer Controller installed
3. ✅ Argo CD installed and configured
4. ✅ Repository secrets for all 6 GitHub repos (created by essesseff)
5. ✅ App-of-apps root Application created
6. ✅ Environment Applications automatically managed (from essesseff template)
7. ✅ Helm charts deploying with environment-specific values.yaml (from essesseff template)
8. ✅ Path-based routing configured for internal ALB (from essesseff template)
9. ✅ essesseff integration ready for automated and manual deployments

The app-of-apps pattern ensures that:
- All environment Applications are managed from a single root Application
- Changes to environment Application definitions are version-controlled in `{app-name}-app-of-apps` repo
- New environments can be added by adding Application YAMLs to the app-of-apps repo
- All deployments follow GitOps principles with automated sync
- Configuration changes are made via Git commits to config repos, not manual kubectl commands
- essesseff orchestrates deployments with RBAC, approvals, and audit trails

## Next Steps

- **Monitor deployments** via Argo CD UI or CLI
- **Use essesseff UI** for image lifecycle management and deployments
- **Configure IAM roles** for ServiceAccounts if your app needs AWS resource access
- **Set up monitoring** and logging as needed
- **Test automatic DEV deployment** by pushing code to `main` branch
- **Test manual deployment** by declaring an RC and deploying to QA through essesseff UI

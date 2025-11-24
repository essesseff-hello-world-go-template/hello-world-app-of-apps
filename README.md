# hello-world - Argo CD App-of-Apps

This repository contains Argo CD Application manifests for all environments of the hello-world essesseff app.

## Repository Structure

```
hello-world-app-of-apps/
├── argocd/
│   ├── hello-world-dev-application.yaml      # DEV environment Application
│   ├── hello-world-qa-application.yaml       # QA environment Application
│   ├── hello-world-staging-application.yaml  # STAGING environment Application
│   └── hello-world-prod-application.yaml     # PROD environment Application
├── app-of-apps.yaml                         # Root Application (applies all environment Applications)
└── README.md                                # This file
```

## Architecture

- **Deployment Model**: Trunk-based development (single `main` branch)
- **Auto-Deploy**: DEV only (via essesseff webhooks)
- **Manual Deploy**: QA, STAGING, PROD (via essesseff UI with RBAC)
- **GitOps**: All deployments managed by Argo CD with automated sync

## Quick Start

For more detailed setup instructions, also see: 
[GitHub Argo CD Machine User and PAT Guide](./argocd-machine-user-github-token.md) 
[EKS Setup Guide](./aws-setup-instructions.md)
[Argo CD External Exposure Guide](./exposing-argocd-externally.md)

### Minimal Setup (After EKS Infrastructure is Ready)

1. **Install Argo CD** (if not already installed):
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

2. **Configure Argo CD access to essesseff app config repositories**:
```bash
# Edit argocd-repository-secret.yaml with your GitHub Argo CD machine username(s) and token(s)
kubectl apply -f argocd-repository-secret.yaml
```

3. **Configure Argo CD access to essesseff app image registry i.e. GHCR**:
```bash
# Edit ghcr-credentials-secret.yaml with your GitHub Argo CD machine username and token, email address and base64 of username:token
kubectl apply -f ghcr-credentials-secret.yaml
```

4. **Deploy app-of-apps root Application**:
```bash
kubectl apply -f app-of-apps.yaml
```

5. **Access Argo CD UI**:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Get admin password (and be sure to securely save it):
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
# Access: https://localhost:8080
```

## Applications

The root Application (`app-of-apps.yaml`) automatically creates these environment Applications:

- **hello-world-dev** → DEV environment (auto-sync enabled)
- **hello-world-qa** → QA environment (auto-sync enabled)
- **hello-world-staging** → STAGING environment (auto-sync enabled)
- **hello-world-prod** → PROD environment (auto-sync enabled)

Each environment Application points to its corresponding config repository:
- `hello-world-config-dev`
- `hello-world-config-qa`
- `hello-world-config-staging`
- `hello-world-config-prod`

## Deployment Process

### Automatic (DEV only)

1. Push code to `main` branch in `hello-world` repository
2. GitHub Actions builds container image
3. essesseff webhook triggers
4. essesseff auto-updates `hello-world-config-dev/Chart.yaml` and `hello-world-config-dev/values.yaml` with new image tag
5. Argo CD syncs DEV Application automatically

### Manual (QA, STAGING, PROD)

1. **Developer** declares Release Candidate (RC) in essesseff UI
2. **QA Engineer** accepts RC → essesseff deploys to QA
3. **QA Engineer** marks image as Stable when ready
4. **Release Engineer** deploys to STAGING (optional)
5. **Release Engineer** deploys to PROD (requires OTP approval)

## Repository URLs

- **Source**: `https://github.com/essesseff-hello-world-go-template/hello-world`
- **Config DEV**: `https://github.com/essesseff-hello-world-go-template/hello-world-config-dev`
- **Config QA**: `https://github.com/essesseff-hello-world-go-template/hello-world-config-qa`
- **Config STAGING**: `https://github.com/essesseff-hello-world-go-template/hello-world-config-staging`
- **Config PROD**: `https://github.com/essesseff-hello-world-go-template/hello-world-config-prod`
- **App-of-Apps**: `https://github.com/essesseff-hello-world-go-template/hello-world-app-of-apps` (this repo)

## essesseff Integration

This setup requires the essesseff platform for deployment orchestration:

- **Event-driven promotions**: Automatic DEV deployments on code push
- **RBAC enforcement**: Role-based access control for deployments
- **Approval workflows**: Manual approvals for QA/STAGING/PROD
- **Deployment policies**: Enforced promotion paths (RC → QA → Stable → STAGING → PROD)
- **Audit trail**: Complete history of all deployments and approvals
- **OTP protection**: One-time password required for PROD deployments

## How It Works

1. **essesseff manages** image lifecycle and promotion decisions
2. **essesseff updates** `values.yaml` files in config repos (e.g., `hello-world-config-dev/values.yaml`) with approved image tags
3. **Argo CD detects** changes via Git polling
4. **Argo CD syncs** Applications automatically (auto-sync enabled)
5. **Kubernetes resources** are updated with new image versions

## See Also

- [GitHub Argo CD Machine User and PAT Guide](./argocd-machine-user-github-token.md)
- [EKS Setup Guide](./aws-setup-instructions.md)
- [Argo CD External Exposure Guide](./exposing-argocd-externally.md)
- [essesseff Documentation](https://essesseff.com/docs) - essesseff platform documentation


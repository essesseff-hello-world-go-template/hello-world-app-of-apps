# Hello World Apps - Argo CD App-of-Apps

This repository contains Argo CD Application manifests for all environments.

## Architecture

- **Deployment Model**: Trunk-based development
- **Auto-Deploy**: DEV only (via essesseff)
- **Manual Deploy**: QA, STAGING, PROD (via essesseff UI)

## Installation

1. Install Argo CD:
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

2. Configure repository access:
```bash
# Edit argocd/argocd-repository-secret.yaml with your GitHub token
kubectl apply -f argocd/argocd-repository-secret.yaml
```

3. Deploy app-of-apps:
```bash
kubectl apply -f app-of-apps.yaml
```

4. Access Argo CD UI:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Applications

- **hello-world-dev** → DEV environment (auto-sync)
- **hello-world-qa** → QA environment (auto-sync)
- **hello-world-staging** → STAGING environment (auto-sync)
- **hello-world-prod** → PROD environment (auto-sync)

## Deployment Process

### Automatic (DEV only)
1. Push code to `main` branch
2. GitHub Actions builds image
3. essesseff webhook triggers
4. essesseff auto-deploys to DEV
5. Argo CD syncs DEV

### Manual (QA, STAGING, PROD)
1. Developer declares Release Candidate in essesseff UI
2. QA accepts RC → essesseff deploys to QA
3. QA marks as Stable
4. Release Engineer deploys to STAGING (optional)
5. Release Engineer requests PROD deployment (requires 2 approvals)
6. After approval, essesseff deploys to PROD

## Repository URLs

- Source: https://github.com/essesseff-hello-world-go-template/hello-world-app
- Config DEV: https://github.com/essesseff-hello-world-go-template/hello-world-config-dev
- Config QA: https://github.com/essesseff-hello-world-go-template/hello-world-config-qa
- Config STAGING: https://github.com/essesseff-hello-world-go-template/hello-world-config-staging
- Config PROD: https://github.com/essesseff-hello-world-go-template/hello-world-config-prod
- Apps: https://github.com/essesseff-hello-world-go-template/hello-world-apps (this repo)

## essesseff Integration

This setup requires the essesseff platform for deployment orchestration:
- Event-driven promotions
- RBAC enforcement
- Approval workflows
- Deployment policies
- Audit trail

See main documentation for essesseff setup instructions.

# Repository Secrets Setup

## ⚠️ SECURITY WARNING

**NEVER commit `argocd-repository-secret.yaml` or `ghcr-credentials-secret.yaml` with actual GitHub tokens!**

The files `argocd-repository-secret.yaml` and `ghcr-credentials-secret.yaml` contain placeholder tokens (`<your-github-token>`) and should be:

1. **Created locally** with actual tokens before applying
2. **Excluded from Git** (see `.gitignore` below)
3. **Deleted after applying** to Argo CD

## Setup Instructions

1. Copy `argocd-repository-secret.yaml` and `ghcr-credentials-secret.yaml` to a secure location
2. Replace all instances of `<your-github-token>` with your actual GitHub personal access token
3. Apply the secrets:
   ```bash
   kubectl apply -f argocd-repository-secret.yaml
   kubectl apply -f ghcr-credentials-secret.yaml
   ```
4. Verify secrets were created:
   ```bash
   kubectl get secrets -n argocd | grep hello-world
   ```
5. **Delete the file** with your token after applying

## GitHub Token Requirements

Your GitHub personal access token needs:
- **repo** scope (Full control of private repositories)
- **read:packages** scope (Download packages from GitHub Container Registry)

## .gitignore Entry

Add this to your `.gitignore`:
```
# Argo CD secrets with tokens (DO NOT COMMIT)
argocd-repository-secret.yaml
*argocd-repository-secret*.yaml
ghcr-credentials-secret.yaml
*ghcr-credentials-secret.yaml*.yaml
```

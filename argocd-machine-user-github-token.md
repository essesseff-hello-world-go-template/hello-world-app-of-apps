# GitHub Organization Argo CD Machine User and PAT Setup Procedure

This document provides a complete procedure for creating GitHub organization Argo CD machine user(s) with a Personal Access Token (PAT) that has repository access and Container Registry permissions.

## Table of Contents

- [Step 1: Create the Argo CD Machine User Account(s)](#step-1-create-the-argo-cd-machine-user-account(s))
- [Step 2: Add Argo CD Machine User(s) to Organization](#step-2-add-argo-cd-machine-user(s)-to-organization)
- [Step 3: Grant Container Registry Access](#step-3-grant-container-registry-access)
- [Step 4: Create Personal Access Token](#step-4-create-personal-access-token)
- [Step 5: Store the Token Securely](#step-5-store-the-token-securely)
- [Step 6: Test the Token](#step-6-test-the-token)
- [Step 7: Document and Rotate](#step-7-document-and-rotate)
- [Security Best Practices](#security-best-practices)
- [SSO Configuration](#if-organization-requires-sso)

---

## Step 1: Create the Argo CD Machine User Account(s)

1. **Sign out** of your personal GitHub account (or use an incognito/private browser window)
2. Go to https://github.com/signup
3. Create a new account with:
   - **Username**: Choose something clearly indicating it's a machine user
     - Examples: `argocd-machine-user-prod`, `argocd-machine-user-qa`, `argocd-machine-user-dev`
   - **Email**: Use a monitored email address
     - Examples: `devops@yourcompany.com` or a distribution list
   - **Password**: Use a strong, randomly generated password stored in your password manager
4. Complete the signup process and **verify the email address**

> **Note**: GitHub's Terms of Service permit machine users for automation purposes, but **not** bot accounts created through automated means.

---

## Step 2: Add Argo CD Machine User(s) to Organization

Choose the option that best fits your access requirements:

### Option A: As an Outside Collaborator (Specific Repositories)

Best for: Limited access to specific repositories only

1. As an **organization owner**, go to your organization page
2. Click **Settings** → **People** → **Outside collaborators**
3. Click **Invite member**
4. Enter the machine user's username
5. Select the repositories it needs access to
6. Choose **Write** or **Admin** access level (required for `repo` scope)
7. Send invitation
8. Switch to the machine user account and accept the invitation

### Option B: As an Organization Member (All/Many Repositories)

Best for: Broader access across many repositories

1. As an **organization owner**, go to your organization page
2. Click **People** → **Invite member**
3. Enter the machine user's email or username
4. Select role:
   - **Member** (for general access)
   - Or add to specific **Teams** for controlled access
5. Send invitation
6. Switch to the machine user account and accept the invitation

### Option C: Add to a Team (Recommended for Scale)

Best for: Managed access with clear governance

1. Create a team (e.g., "Automation" or "Deploy Bots")
2. Go to **Settings** → **Teams** → Your team
3. Click **Members** → **Add a member**
4. Add the machine user
5. Configure **Team repositories** to grant access to needed repos
6. Set team permissions appropriately

---

## Step 3: Grant Container Registry Access

If your packages are organization-scoped:

1. Go to your **organization Settings** → **Packages**
2. Find the packages the machine user needs to access
3. Click the package → **Package settings** → **Manage Actions access**
4. Add the machine user's username with appropriate role:
   - **Read** (for `read:packages` scope)
   - **Write** (if it needs to publish packages)

---

## Step 4: Create Personal Access Token

1. **Log in as the machine user**
2. Click profile picture → **Settings**
3. Scroll down to **Developer settings** (bottom of left sidebar)
4. Click **Personal access tokens**

### Option A: Classic Token (Simpler, broader access)

1. Click **Tokens (classic)** → **Generate new token (classic)**
2. Authenticate if prompted
3. Fill in token details:
   - **Note**: `Argo CD Machine User PROD - Created YYYY-MM-DD`
   - **Expiration**: Choose based on your security policy
     - Recommendation: 90 days or 1 year (requires rotation)
     - Or use "No expiration" only if absolutely necessary
   - **Select scopes**:
     - ✅ **`repo`** (Full control of private repositories)
       - This includes: repo:status, repo_deployment, public_repo, repo:invite, security_events
     - ✅ **`read:packages`** (Download packages from GitHub Container Registry)
     - Optional: **`write:packages`** (if it needs to publish packages)
     - Optional: **`delete:packages`** (if it needs to delete packages)
4. Click **Generate token**
5. **IMMEDIATELY COPY THE TOKEN** - you won't see it again!

### Option B: Fine-grained Token (More secure, recommended)

1. Click **Fine-grained tokens** → **Generate new token**
2. Fill in:
   - **Token name**: `Argo CD Machine User DEV - Created YYYY-MM-DD`
   - **Expiration**: Set based on security policy
   - **Resource owner**: Select your organization
   - **Repository access**: 
     - **All repositories** (if needed for all repos)
     - Or **Only select repositories** (choose specific repos)
3. Set **Repository permissions**:
   - **Contents**: Read and write (gives repo access)
   - **Metadata**: Read (automatically included)
   - **Packages**: Read (for `read:packages`)
4. If organization requires approval:
   - Click **Generate token**
   - Token will be in "Pending" state
   - An organization owner must approve it
5. **IMMEDIATELY COPY THE TOKEN** once generated/approved

---

## Step 5: Store the Token Securely

Choose one of these methods based on your infrastructure:

### Option A: Secrets Manager (Recommended)

- **AWS Secrets Manager**
- **Azure Key Vault**
- **HashiCorp Vault**
- **1Password Secrets Automation**
- **GitHub Actions Secrets** (if used in workflows)

### Option B: Environment Variable

```bash
# On your server/CI system
export GITHUB_TOKEN="ghp_xxxxxxxxxxxx"

# Or in your deployment config
GITHUB_TOKEN=ghp_xxxxxxxxxxxx
```

### Option C: Git Credential Helper

```bash
# Store in git credential helper
git config --global credential.helper store
echo "https://${USERNAME}:${TOKEN}@github.com" > ~/.git-credentials
```

> **Warning**: Be extremely careful with credential storage. Ensure proper file permissions and never commit credentials to repositories.

---

## Step 6: Test the Token

### Test Repository Access

```bash
# Clone a private repo
git clone https://MACHINE-USERNAME:TOKEN@github.com/org/repo.git

# Or set it as remote
git remote set-url origin https://MACHINE-USERNAME:TOKEN@github.com/org/repo.git
```

### Test API Access

```bash
# Test API access
curl -H "Authorization: token YOUR_TOKEN" \
  https://api.github.com/user

# Should return machine user details

# List org repos
curl -H "Authorization: token YOUR_TOKEN" \
  https://api.github.com/orgs/YOUR_ORG/repos
```

### Test Container Registry Access

```bash
# Login to GitHub Container Registry
echo "YOUR_TOKEN" | docker login ghcr.io -u MACHINE-USERNAME --password-stdin

# Pull an image
docker pull ghcr.io/your-org/your-image:tag
```

Expected output for successful login:
```
Login Succeeded
```

---

## Step 7: Document and Rotate

### Documentation Checklist

Create documentation that includes:

- [ ] Where the token(s) are stored
- [ ] What the token(s) are used for
- [ ] When the token(s) were created
- [ ] When the token(s) expire
- [ ] Who has access to the token(s)
- [ ] Argo CD Machine User username(s)
- [ ] Organization and repositories the machine user(s) / token(s) have access to

### Set Up Rotation Reminders

1. **Calendar reminder**: Set for 2 weeks before expiration
2. **Monitoring**: Check token usage regularly
3. **Automation**: Consider automating rotation if your secrets manager supports it

### Monitor Token Usage

1. Go to GitHub Settings → Developer settings → Personal access tokens
2. Check the "Last used" column periodically
3. Revoke any tokens that haven't been used in a while

---

## Security Best Practices

### ✅ DO:

- Use fine-grained tokens when possible
- Set expiration dates on all tokens
- Use the minimum required scopes
- Store tokens in secrets managers
- Rotate tokens regularly (at least annually)
- Monitor token usage
- Revoke tokens immediately if compromised
- Use separate tokens for different services/purposes
- Enable two-factor authentication on the machine user account
- Use teams to manage machine user access

### ❌ DON'T:

- Commit tokens to repositories
- Share tokens between services
- Use tokens with broader permissions than needed
- Set "No expiration" unless absolutely necessary
- Reuse personal tokens for machine automation
- Store tokens in plain text files
- Email or message tokens
- Use the same password across multiple machine users

---

## If Organization Requires SSO

If your organization uses SAML Single Sign-On:

1. After creating the token, go back to **Personal access tokens**
2. Find your new token in the list
3. Click **Configure SSO** next to it
4. Click **Authorize** for your organization
5. Complete SSO authentication
6. Verify the token shows "Authorized" for your organization

---

## Troubleshooting

### Token doesn't work for private repositories
- Verify the `repo` scope is enabled
- Check that the machine user has access to the repository
- If using fine-grained token, verify the repository is in the allowed list

### Can't access Container Registry
- Verify `read:packages` scope is enabled
- Check package permissions in organization settings
- Ensure the package visibility allows the machine user access

### Token pending approval (fine-grained tokens)
- Contact an organization owner to approve the token
- Check organization settings for token approval policies

### SSO Authorization Required
- Follow the SSO configuration steps above
- Contact your organization admin if SSO authorization fails

---

## Example Use Cases

### In CI/CD Pipeline (GitHub Actions)

```yaml
name: Deploy
on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Pull Docker image
        run: |
          echo "${{ secrets.MACHINE_USER_PAT }}" | docker login ghcr.io -u machine-user --password-stdin
          docker pull ghcr.io/org/app:latest
```

### In Deployment Script

```bash
#!/bin/bash

# Load token from environment or secrets manager
GITHUB_TOKEN="${GITHUB_TOKEN}"

# Clone repository
git clone https://machine-user:${GITHUB_TOKEN}@github.com/org/repo.git

# Pull container image
echo "${GITHUB_TOKEN}" | docker login ghcr.io -u machine-user --password-stdin
docker pull ghcr.io/org/app:latest
```

---

## Additional Resources

- [GitHub: Managing your personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
- [GitHub: Managing deploy keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys)
- [GitHub: About permissions for GitHub Packages](https://docs.github.com/en/packages/learn-github-packages/about-permissions-for-github-packages)
- [GitHub: Working with the Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)

---

**Document Version**: 1.0  
**Last Updated**: November 21, 2024  
**Maintained By**: DevOps Team

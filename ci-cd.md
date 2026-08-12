# CI/CD Pipeline

This document describes the GitHub Actions pipeline for Expensy, how to configure it, and a known limitation.

## Overview

The pipeline lives at `.github/workflows/ci-cd.yaml` and runs on every push to `main`. It has three sequential stages:

1. **build** — installs dependencies and builds both services (matrix job: `expensy_backend`, `expensy_frontend`)
2. **test** — runs each service's test suite (matrix job, depends on `build`)
3. **docker-build-push** — authenticates to AWS via OIDC, builds both Docker images, and pushes them to ECR (depends on `test`)

There is currently no automated `deploy` stage — that will be added once the EKS cluster (Phase 4) is provisioned and the manifests (Phase 5) are finalized.

## Required GitHub Secrets

Configure under **Settings → Secrets and variables → Actions**:

| Secret name | Value | Purpose |
|---|---|---|
| `AWS_ROLE_ARN` | `arn:aws:iam::686699774218:role/penchal-expensy-github-actions-role` | Assumed via OIDC to authenticate to AWS for ECR push |

No long-lived AWS access keys are stored in GitHub — authentication uses OpenID Connect (OIDC) federation instead, so GitHub issues a short-lived token that AWS exchanges for temporary credentials scoped to this one role.

## AWS-side setup (for reference)

This was configured once, outside of GitHub, and doesn't need to be repeated unless the role or provider is deleted:

1. **OIDC Identity Provider** registered in IAM:
   - URL: `https://token.actions.githubusercontent.com`
   - Client ID: `sts.amazonaws.com`

2. **IAM Role** (`penchal-expensy-github-actions-role`) with a trust policy scoped to this exact repo:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::686699774218:oidc-provider/token.actions.githubusercontent.com"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringEquals": {
             "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
           },
           "StringLike": {
             "token.actions.githubusercontent.com:sub": "repo:penchalj/devops-project-expensy:*"
           }
         }
       }
     ]
   }
   ```

3. **Inline policy** attached to the role, granting only `ecr:GetAuthorizationToken` (account-wide, required by ECR auth) plus push permissions scoped to exactly two repositories:
   - `penchal-expensy-backend`
   - `penchal-expensy-frontend`

   No broader permissions (e.g. `AdministratorAccess`, `PowerUserAccess`) are attached.

## ECR Repositories

Images are pushed to:
- `686699774218.dkr.ecr.us-east-1.amazonaws.com/penchal-expensy-backend:<git-sha>`
- `686699774218.dkr.ecr.us-east-1.amazonaws.com/penchal-expensy-frontend:<git-sha>`

Each image is tagged with the triggering commit's SHA (`${{ github.sha }}`), not `latest`, so every pushed image is traceable back to an exact commit.

## Known Issue: `docker-build-push` fails at OIDC auth

**Status:** Unresolved, escalated to instructor.

The `Configure AWS credentials via OIDC` step fails with:
```
Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

This was investigated thoroughly and ruled out as a configuration error on the project side:

- ✅ IAM role trust policy verified correct (`aws iam get-role`)
- ✅ OIDC provider verified registered with correct thumbprint and client ID (`aws iam get-open-id-connect-provider`)
- ✅ Repository path in trust policy confirmed to exactly match `git remote -v` output (case-sensitive)
- ❌ CloudTrail shows **zero** `AssumeRoleWithWebIdentity` events matching this role or the `githubusercontent.com` identity provider, across multiple isolated attempts in tightly-scoped time windows — even for a run that failed moments before the CloudTrail query

The absence of any CloudTrail record — rather than a logged, explicit `Deny` — suggests the call may be blocked before IAM evaluation, which is consistent with an **account or AWS Organizations-level Service Control Policy (SCP)** restricting external OIDC federation. This AWS account is shared across the course cohort (confirmed via pre-existing IAM roles scoped to other students' repos), making an org-level policy a plausible explanation outside of student control.

**Impact:** `build` and `test` stages are unaffected and pass reliably. Only the final image-push step is blocked, meaning images must currently be built and pushed manually if needed for testing:

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 686699774218.dkr.ecr.us-east-1.amazonaws.com

docker build -t 686699774218.dkr.ecr.us-east-1.amazonaws.com/penchal-expensy-backend:manual \
  -f expensy_backend/Dockerfile expensy_backend
docker push 686699774218.dkr.ecr.us-east-1.amazonaws.com/penchal-expensy-backend:manual
```

**Next step:** Confirm with the course instructor whether an SCP restricts OIDC federation for this account, or whether an alternative (e.g. long-lived IAM user credentials stored as a GitHub secret) is expected instead.

## Local verification

Before relying on CI, both services' build and test steps can be verified locally:

```bash
cd expensy_backend && npm ci && npm run build && npm test
cd ../expensy_frontend && npm ci && npm run build && npm test
```

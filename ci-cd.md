# CI/CD Pipeline

This document describes the GitHub Actions pipeline for Expensy, how to configure it, and its current status.

## Overview

The pipeline lives at `.github/workflows/ci-cd.yaml` and runs on every push to `main`. It has three sequential stages:

1. **build** — installs dependencies and builds both services (matrix job: `expensy_backend`, `expensy_frontend`)
2. **test** — runs each service's test suite (matrix job, depends on `build`)
3. **docker-build-push** — authenticates to AWS via OIDC, builds both Docker images, and pushes them to ECR (depends on `test`)

**All three stages are fully working and green as of run #8.**

There is currently no automated `deploy` stage — Kubernetes manifests (`k8s/`) are applied manually via `kubectl apply -f k8s/` rather than as part of the pipeline. Automating this (e.g. a `deploy` job running `kubectl apply` against the EKS cluster) is a natural next step but was not implemented, since the frontend image also needs to be rebuilt with a fresh `NEXT_PUBLIC_API_URL` build arg whenever the backend's external address changes (see "Known limitation" in the main `README.md`) — automating that safely would need more thought than fits in this pipeline as-is.

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

Note: the frontend image actually running on the cluster (`penchal-expensy-frontend:with-api-url`) was built and pushed manually rather than through this pipeline, since it needed a specific `NEXT_PUBLIC_API_URL` build argument baked in — see the main `README.md` for details.

## Resolved Issue: OIDC authentication (previously blocked)

Earlier runs of `docker-build-push` failed with:
```
Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

This was investigated thoroughly at the time and ruled out as a configuration error on the project side:

- ✅ IAM role trust policy verified correct (`aws iam get-role`)
- ✅ OIDC provider verified registered with correct thumbprint and client ID (`aws iam get-open-id-connect-provider`)
- ✅ Repository path in trust policy confirmed to exactly match `git remote -v` output (case-sensitive)
- ❌ CloudTrail showed **zero** `AssumeRoleWithWebIdentity` events matching this role or the `githubusercontent.com` identity provider, across multiple isolated attempts in tightly-scoped time windows

The absence of any CloudTrail record — rather than a logged, explicit `Deny` — suggested the call was being blocked before IAM evaluation, consistent with an **account or AWS Organizations-level Service Control Policy (SCP)** restricting external OIDC federation in this shared cohort AWS account.

**As of run #8 (commit `ef200be`), `docker-build-push` completes successfully.** The underlying restriction appears to have been lifted or resolved on the account side — images now push to ECR automatically on every push to `main`. The root cause of the original block was never confirmed (it resolved before an instructor response came back), but the pipeline has been reliably green since.

## Local verification

Both services' build and test steps can be verified locally before pushing:

```bash
cd expensy_backend && npm ci && npm run build && npm test
cd ../expensy_frontend && npm ci && npm run build && npm test
```

## Manual image build/push (if ever needed)

Since the pipeline handles this automatically now, this is only needed for one-off builds (e.g. the `with-api-url` tag mentioned above):

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 686699774218.dkr.ecr.us-east-1.amazonaws.com

docker build -t 686699774218.dkr.ecr.us-east-1.amazonaws.com/penchal-expensy-backend:manual \
  -f expensy_backend/Dockerfile expensy_backend
docker push 686699774218.dkr.ecr.us-east-1.amazonaws.com/penchal-expensy-backend:manual
```

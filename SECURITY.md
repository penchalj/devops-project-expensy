# Security & Compliance

This document summarizes how credentials, secrets, IAM roles, and network access are handled for the Expensy DevOps project, and notes known gaps/pending items.

## 1. IAM & Access Control

### Principle of least privilege
- A dedicated, project-scoped IAM role (`penchal-expensy-github-actions-role`) was created for CI/CD, rather than reusing broader existing roles found in this shared AWS account.
- During setup, an existing role (`ironhack-project-1-github-actions-role`) belonging to another user was identified — it carried `IAMFullAccess` and `PowerUserAccess`, both far broader than needed. This role was **not reused or modified**; a new, narrowly-scoped role was created instead.
- The CI/CD role's inline policy grants only `ecr:*` push permissions, scoped to two specific repositories:
  - `penchal-expensy-backend`
  - `penchal-expensy-frontend`
  No wildcard resource access is granted.

### Naming convention
All project-owned resources (IAM roles, ECR repos, VPC, EKS cluster, IAM cluster/node roles) are prefixed `penchal-expensy-` or `penchal-`, to avoid collisions and ambiguity in a shared, multi-student AWS account.

### Cluster & node IAM roles
- EKS cluster role (`penchal-expensy-cluster-role`) — trusts only `eks.amazonaws.com`, attached policy: `AmazonEKSClusterPolicy`.
- EKS node role (`penchal-expensy-node-role`) — trusts only `ec2.amazonaws.com`, attached policies: `AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy`, `AmazonEC2ContainerRegistryReadOnly` (read-only — nodes pull images, they do not need push access).
- Neither role uses admin/root-equivalent permissions.

### Known issue: GitHub Actions OIDC federation
The CI/CD pipeline's `docker-build-push` job fails at `sts:AssumeRoleWithWebIdentity`, despite a verified-correct trust policy, OIDC provider registration, and matching repo path. The failed calls do not appear in CloudTrail at all, even across multiple isolated attempts in tightly-scoped time windows. This is consistent with an account/organization-level Service Control Policy (SCP) blocking external OIDC federation — plausible given this is a shared cohort AWS account. This has been raised with the course instructor; `build` and `test` stages of the pipeline are unaffected and pass successfully.

## 2. Secrets Management

- No secrets are committed to the repository. `.gitignore` explicitly excludes `.env`, `.env.*` (except `.env.example`), `*.pem`, `*.key`, `*credentials*`, Terraform state/plan files, and Kubernetes secret manifests generated locally (e.g. `k8s/01-secrets.yaml`).
- Local development secrets (`DATABASE_URI`, `REDIS_PASSWORD`) are supplied via `.env` files (gitignored) and Docker Compose `env_file` directives — never hardcoded into Dockerfiles or source.
- Kubernetes secrets (`expensy-secrets`) are created directly via `kubectl create secret generic --from-literal=...`, rather than committing a filled-in Secret manifest — this keeps plaintext values out of git history entirely, including in any prior commits.
- GitHub Actions secrets (`AWS_ROLE_ARN`) are stored in GitHub's encrypted repo secrets store (**Settings → Secrets and variables → Actions**), not in workflow YAML.
- Before any commit that touches tracked files, `git status` is checked to confirm no `.env`, credential JSON (e.g. IAM trust-policy documents), or Terraform state files are staged.

## 3. Network Security

**Current state:**
- Backend Kubernetes Service is `ClusterIP` (internal-only) — not directly reachable from outside the cluster.
- Frontend Kubernetes Service is `LoadBalancer` — the only externally-exposed entry point.
- EKS cluster endpoint is currently `endpoint_public_access = true`, `endpoint_private_access = false` (standard for coursework/demo access from a local machine; not intended for production use as-is).

**Pending / not yet implemented:**
- TLS/HTTPS termination (via Ingress + cert-manager, or an ACM-backed load balancer) is not yet configured. Traffic to the frontend LoadBalancer is currently plain HTTP.
- Security group / NSG lockdown to restrict inbound traffic to only necessary ports (443, cluster-internal) has not yet been applied — default EKS-managed security groups are in use.
- These are planned as follow-up work before any production-style demo.

## 4. Data Handling

- The application stores expense-tracking data (user-entered financial records) in MongoDB, and uses Redis for caching/session data.
- No PII beyond what a user voluntarily enters (e.g. expense descriptions, amounts) is collected.
- Data is not currently encrypted at rest (default MongoDB/EBS volume behavior on EKS). Enabling EBS volume encryption for the underlying PVCs is a planned improvement.
- This project is a coursework/demo application and does not currently implement GDPR/HIPAA-specific controls (e.g. data subject deletion requests, audit logging of data access). If this were a production system handling real user financial data, that would be a required next step.

## 5. Summary of Trade-offs (for transparency)

| Area | Current approach | Production-grade alternative |
|---|---|---|
| CI/CD auth | Blocked (SCP issue), pending instructor input | OIDC federation (intended design) |
| EKS endpoint | Public access enabled | Private endpoint + VPN/bastion |
| TLS | Not yet implemented | cert-manager + Let's Encrypt or ACM |
| Data at rest | Unencrypted | EBS encryption enabled |
| Secrets at rest (k8s) | Native k8s Secrets (base64, not encrypted by default) | AWS Secrets Manager / Sealed Secrets |


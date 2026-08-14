# Expensy — End-to-End DevOps Deployment

[![CI/CD](https://github.com/penchalj/devops-project-expensy/actions/workflows/ci-cd.yaml/badge.svg)](https://github.com/penchalj/devops-project-expensy/actions/workflows/ci-cd.yaml)

Expensy is a lightweight expense tracker app (Next.js frontend + Express/TypeScript backend, MongoDB + Redis) deployed with a full DevOps pipeline: containerized, tested and pushed via CI/CD, provisioned on AWS EKS with Terraform, and monitored with Prometheus/Grafana.

**Status: fully deployed and verified end-to-end on live AWS infrastructure.**

<!--
DEMO: replace this line with an embedded GIF or screenshot once captured — see
"Capturing the demo" section below for exact steps. Suggested path: docs/images/demo.gif
Example once ready:
![Expensy demo](docs/images/demo.gif)
-->

## Table of contents

- [Architecture](#architecture)
- [Local development](#local-development)
- [Docker / Docker Compose](#docker--docker-compose)
- [CI/CD](#cicd)
- [Deploying to EKS](#deploying-to-eks)
- [Monitoring](#monitoring)
- [Security](#security)
- [Problems & Fixes](#problems--fixes)
- [Known limitations](#known-limitations)
- [Repository structure](#repository-structure)

## Architecture

**Component & functional diagram** — how requests flow from the browser through to storage, and how CI/CD ships new images:

![Component and functional diagram](docs/images/component-diagram.png)

**AWS infrastructure diagram** — the underlying VPC, EKS cluster, and supporting AWS resources:

![AWS infrastructure diagram](docs/images/aws-infrastructure-diagram.png)

All four application services run as Kubernetes Deployments on an EKS cluster, alongside a Prometheus/Grafana monitoring stack.

## Local development

**Prerequisites:** Node.js 20+, npm, Docker.

**1. Start MongoDB and Redis:**
```bash
docker run --name mongo -d -p 27017:27017 \
  -e MONGO_INITDB_ROOT_USERNAME=root \
  -e MONGO_INITDB_ROOT_PASSWORD=example \
  mongo:latest

docker run --name redis -d -p 6379:6379 \
  redis:latest redis-server --requirepass someredispassword
```

**2. Backend:**
```bash
cd expensy_backend
cp .env.example .env   # fill in DATABASE_URI, REDIS_PASSWORD, etc.
npm install
npm run dev
```

**3. Frontend:**
```bash
cd expensy_frontend
# set NEXT_PUBLIC_API_URL in .env.local to point at the backend (e.g. http://localhost:8706)
npm install
npm run dev
```

Visit `http://localhost:3000` and confirm the app loads and can reach the backend.

## Docker / Docker Compose

Each service has its own multi-stage `Dockerfile` (non-root user, slim runtime image). To run the full stack locally in containers:

```bash
docker compose up --build
```

This starts `mongo`, `redis`, `backend` (port `8706`), and `frontend` (port `3001` on the host, mapped to `3000` in-container). See `docker-compose.yml` for the full service wiring and environment variables.

## CI/CD

GitHub Actions (`.github/workflows/ci-cd.yaml`) runs on every push to `main`:

1. **build** — installs deps and builds both services
2. **test** — runs each service's test suite
3. **docker-build-push** — authenticates to AWS via OIDC and pushes both images to ECR, tagged by commit SHA

Full details, required secrets, and setup steps: **[CI-CD.md](./CI-CD.md)**.

## Deploying to EKS

### 1. Provision infrastructure

```bash
cd infrastructure/terraform
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

This creates a VPC, an EKS cluster (`penchal-expensy`), a managed node group, and least-privilege IAM roles for the cluster and nodes. Takes roughly 10 minutes, most of it waiting on the EKS control plane.

### 2. Point kubectl at the cluster

```bash
aws eks update-kubeconfig --name penchal-expensy --region us-east-1
kubectl get nodes   # should show 2 nodes, Ready
```

### 3. Install the EBS CSI driver (required for persistent storage)

EKS clusters provisioned via raw Terraform don't include the EBS CSI driver by default — without it, Mongo/Redis's PersistentVolumeClaims will stay `Pending` indefinitely.

```bash
eksctl utils associate-iam-oidc-provider --cluster penchal-expensy --region us-east-1 --approve

eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa --namespace kube-system \
  --cluster penchal-expensy --region us-east-1 \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve --role-only --role-name penchal-expensy-ebs-csi-role

aws eks create-addon \
  --cluster-name penchal-expensy --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::686699774218:role/penchal-expensy-ebs-csi-role

kubectl apply -f k8s/gp3-storageclass.yaml
```

### 4. Deploy the application

```bash
kubectl apply -f k8s/00-namespace.yaml

kubectl create secret generic expensy-secrets \
  --namespace=expensy \
  --from-literal=DATABASE_URI='mongodb://root:example@mongo:27017' \
  --from-literal=REDIS_PASSWORD='someredispassword'

kubectl apply -f k8s/02-mongo.yaml
kubectl apply -f k8s/03-redis.yaml
kubectl apply -f k8s/04-backend.yaml
kubectl apply -f k8s/05-frontend.yaml

kubectl get pods -n expensy --watch
```

Wait for all 6 pods to reach `Running`.

### 5. Expose the app and connect frontend to backend

The frontend's `NEXT_PUBLIC_API_URL` is baked in at **build time**, so it must point at a real, reachable backend address before the frontend image is built. This is the trickiest part of the deploy — see [Known limitations](#known-limitations) for why.

```bash
# expose backend (NodePort avoids AWS LoadBalancer quota limits — see below)
kubectl patch svc backend -n expensy -p '{"spec": {"type": "NodePort"}}'
kubectl get svc -n expensy backend   # note the NodePort, e.g. 31411
kubectl get nodes -o wide            # note a node's EXTERNAL-IP

# allow inbound traffic on the NodePort range (only needed once per cluster)
aws ec2 authorize-security-group-ingress \
  --group-id <node-security-group-id> \
  --protocol tcp --port 30000-32767 --cidr 0.0.0.0/0

# rebuild frontend with the real backend address baked in
cd expensy_frontend
docker build --build-arg NEXT_PUBLIC_API_URL=http://<node-ip>:<nodeport> \
  -t 686699774218.dkr.ecr.us-east-1.amazonaws.com/penchal-expensy-frontend:with-api-url .
docker push 686699774218.dkr.ecr.us-east-1.amazonaws.com/penchal-expensy-frontend:with-api-url

kubectl set image deployment/frontend \
  frontend=686699774218.dkr.ecr.us-east-1.amazonaws.com/penchal-expensy-frontend:with-api-url \
  -n expensy
kubectl rollout status deployment/frontend -n expensy
```

Frontend is exposed via a `LoadBalancer` Service — get its address:
```bash
kubectl get svc -n expensy frontend
```

Open the `EXTERNAL-IP` hostname in a browser. **This deployment has been manually verified end-to-end** — signing up, navigating the app, and adding an expense all work correctly through the live UI.

## Monitoring

Prometheus + Grafana (`kube-prometheus-stack` via Helm) are deployed in the `monitoring` namespace, scraping both application pods and cluster-level metrics (CPU, memory, restarts). Full setup steps: **[monitoring/README.md](./monitoring/README.md)**.

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3001:80
# open http://localhost:3001, log in, check the "Expensy" dashboard folder
```

## Security

IAM least-privilege design, secrets handling, network posture, and known gaps (TLS not yet implemented) are documented in **[SECURITY.md](./SECURITY.md)**.

## Problems & Fixes

Everything below actually happened during this build — no step was skipped or simplified for the writeup. Each entry follows the same shape: what broke, how it was diagnosed, and what fixed it.

### 1. GitHub Actions OIDC authentication failed on every push

**Symptom:** The `docker-build-push` job failed consistently with:
```
Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

**Investigation:** Rather than guessing, each piece of the OIDC chain was verified independently:
- IAM role trust policy checked against `aws iam get-role` — correct
- OIDC identity provider checked against `aws iam get-open-id-connect-provider` — correct thumbprint, correct client ID
- Repository path in the trust policy (`repo:penchalj/devops-project-expensy:*`) confirmed to exactly match `git remote -v`, case-sensitive

With every documented piece confirmed correct, the next step was checking whether the call was even reaching AWS. Querying CloudTrail for `AssumeRoleWithWebIdentity` events — across several isolated attempts, including one just seconds after a live failed run — turned up **zero matching events**. Not a logged `Deny`, just... nothing. That absence was the actual signal: the call was being rejected before IAM ever evaluated it, which points to an account or AWS Organizations–level Service Control Policy rather than anything in the role/trust-policy configuration.

**Fix:** Since this pointed to infrastructure outside the project's control (a shared, cohort-wide AWS account), the finding was documented with the full evidence trail and escalated rather than "fixed" by guessing at workarounds. The pipeline began passing on its own days later — consistent with an account-level restriction being adjusted upstream. `build` and `test` were unaffected throughout; only the final image-push step was blocked.

**Why it mattered:** This is the difference between "it doesn't work, not sure why" and a documented, ruled-out root cause with evidence — the kind of trail that makes it fast for whoever owns the account to act on.

---

### 2. Kubernetes PersistentVolumeClaims stuck in `Pending`

**Symptom:** After deploying MongoDB and Redis, their PVCs sat in `Pending` indefinitely. `kubectl describe pvc` showed:
```
FailedBinding: no persistent volumes available for this claim and no storage class is set
```

**Investigation:** A `StorageClass` named `gp2` did exist on the cluster — but its provisioner (`kubernetes.io/aws-ebs`) is part of Kubernetes' legacy in-tree AWS EBS support, which was removed in Kubernetes 1.23+. Since this cluster runs 1.31, that StorageClass was inert. The actual gap: a cluster provisioned via raw Terraform (rather than `eksctl`) doesn't install the **EBS CSI driver** by default, and without it there's no way to dynamically provision the underlying EBS volumes the claims need.

**Fix:**
1. Associated an IAM OIDC provider with the *cluster itself* (a separate OIDC setup from the GitHub Actions one — cluster-scoped, for IRSA)
2. Created a scoped IAM role for the EBS CSI driver via `eksctl create iamserviceaccount`
3. Installed the driver as an EKS add-on, pointed at that role
4. Defined a working `gp3` StorageClass and set it as cluster default

Both PVCs bound within seconds of the StorageClass being created.

**Why it mattered:** This wasn't a typo or a missing YAML field — it required understanding *why* a resource that looked correctly configured (`gp2` existed, PVCs referenced valid claims) still failed, which meant reasoning about Kubernetes version history and how EKS clusters differ depending on how they're provisioned.

---

### 3. LoadBalancer stuck in `<pending>` for 18+ minutes

**Symptom:** The frontend's `Service` (`type: LoadBalancer`) never received an external IP. `kubectl describe svc` revealed the real error, repeated on every reconcile attempt:
```
TooManyLoadBalancers: Exceeded quota of account 686699774218
```

**Investigation:** This AWS account is shared across an entire course cohort. Querying existing load balancers directly (`aws elbv2 describe-load-balancers`) showed 5 already in use — clearly named after other students' projects, not anything created by this one. The account-wide ELB quota was simply exhausted by other people's active work.

**Fix:** Rather than blocking on a quota increase that wasn't in this project's control, `NodePort` was used as a working interim exposure method — no ELB required, reachable via a node's public IP plus a corresponding security-group rule opened for the NodePort range. When the quota later cleared on its own, the frontend was switched back to `LoadBalancer`.

**Why it mattered:** Recognizing a shared-resource contention problem versus a configuration bug — and having a working fallback ready rather than just waiting — kept the project moving without depending on someone else's timeline.

---

### 4. Prometheus discovered the app's Services but never scraped them

**Symptom:** `ServiceMonitor` objects for the backend and frontend existed, matched no errors in the Prometheus Operator logs, and RBAC was fully open (`serviceMonitorSelector: {}`, `serviceMonitorNamespaceSelector: {}`). Yet `/targets` in the Prometheus UI showed nothing from the app — only the built-in cluster jobs (`node_exporter`, `kube-state-metrics`, etc.).

**Investigation:** Working backward from Prometheus's actual scrape config (`/api/v1/status/config`) confirmed the `expensy-backend` and `expensy-frontend` jobs were registered — so the ServiceMonitors *were* being read. The next layer down, service discovery's dropped-targets list, showed the backend and frontend endpoints were being **discovered and then dropped** during relabeling. Comparing the ServiceMonitor's `selector.matchLabels: app: backend` against the actual Kubernetes `Service` objects revealed the gap: the Services had a `spec.selector` (which picks the *pods* to route to) but no `metadata.labels` on the Service object itself — and it's the latter that a ServiceMonitor's selector matches against.

**Fix:**
```bash
kubectl patch svc backend -n expensy -p '{"metadata":{"labels":{"app":"backend"}}}'
kubectl patch svc frontend -n expensy -p '{"metadata":{"labels":{"app":"frontend"}}}'
```
Confirmed via a live re-check of `/targets` — both services immediately went active (returning `404` on `/metrics`, expected, since that endpoint isn't implemented yet — but proof the network path and discovery pipeline were fully correct).

**Why it mattered:** Every layer *looked* fine in isolation — no errors, correct RBAC, matching-looking selectors. Finding the actual gap meant understanding the difference between a Service's `spec.selector` (pod routing) and its `metadata.labels` (what other objects match against) — a subtle but common Kubernetes distinction.

## Known limitations

Being upfront about trade-offs made for a coursework/demo timeline:

- **Backend's address isn't stable.** It's exposed via `NodePort` + a specific EC2 node's public IP, rather than a proper LoadBalancer or Ingress with a stable DNS name. If that node is ever replaced (scaling, AZ failure, node group update), the address changes and the frontend image must be rebuilt with a new `NEXT_PUBLIC_API_URL`. A proper fix would use an Ingress controller (e.g. AWS Load Balancer Controller) with a single stable hostname for both services.
- **NodePort was used instead of a second LoadBalancer** because this AWS account's Elastic Load Balancer quota is shared across the course cohort and was intermittently exhausted by other students' resources during development.
- **TLS/HTTPS is not implemented.** Traffic to both the frontend LoadBalancer and backend NodePort is plain HTTP. See `SECURITY.md` for the planned fix (cert-manager + Let's Encrypt, or an ACM-backed ALB).
- **No automated `deploy` stage in CI/CD.** Manifests are applied manually via `kubectl apply`, and the frontend image with the correct API URL is built manually — automating this safely needs an Ingress-based stable-hostname setup first (see above).
- **Backend doesn't yet expose `/metrics`.** Infrastructure-level monitoring (pod CPU/memory/restarts) works fully; application-level metrics (HTTP request rates) are stubbed out pending a small `prom-client` addition — see `monitoring/README.md`.

## Repository structure

```
devops-project-expensy/
├── expensy_backend/       # Express/TypeScript API
├── expensy_frontend/      # Next.js app
├── infrastructure/
│   └── terraform/         # VPC, EKS cluster, IAM roles
├── k8s/                   # Kubernetes manifests
├── monitoring/            # Prometheus/Grafana Helm values, dashboards
├── docs/images/           # README diagrams and demo media
├── .github/workflows/     # CI/CD pipeline
├── docker-compose.yml     # Local multi-service orchestration
├── CI-CD.md
├── SECURITY.md
└── README.md              # this file
```

# Expensy — End-to-End DevOps Deployment

Expensy is a lightweight expense tracker app (Next.js frontend + Express/TypeScript backend, MongoDB + Redis) deployed with a full DevOps pipeline: containerized, tested and pushed via CI/CD, provisioned on AWS EKS with Terraform, and monitored with Prometheus/Grafana.

**Status: fully deployed and verified end-to-end on live AWS infrastructure** — see [Live deployment](#deploying-to-eks) below.

## Table of contents

- [Architecture](#architecture)
- [Local development](#local-development)
- [Docker / Docker Compose](#docker--docker-compose)
- [CI/CD](#cicd)
- [Deploying to EKS](#deploying-to-eks)
- [Monitoring](#monitoring)
- [Security](#security)
- [Known limitations](#known-limitations)
- [Repository structure](#repository-structure)

## Architecture

```
                    ┌─────────────┐
   Browser  ───────▶│  frontend   │  (Next.js, 2 replicas)
                    └──────┬──────┘
                           │ NEXT_PUBLIC_API_URL
                           ▼
                    ┌─────────────┐
                    │   backend   │  (Express/TS, 2 replicas)
                    └──────┬──────┘
                    ┌──────┴──────┐
                    ▼             ▼
              ┌──────────┐  ┌──────────┐
              │  MongoDB │  │  Redis   │
              └──────────┘  └──────────┘
```

All four services run as Kubernetes Deployments on an EKS cluster, alongside a Prometheus/Grafana monitoring stack.

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
├── .github/workflows/     # CI/CD pipeline
├── docker-compose.yml     # Local multi-service orchestration
├── CI-CD.md
├── SECURITY.md
└── README.md              # this file

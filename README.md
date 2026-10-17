# Bank App
Learning project: take a real multi-service banking app (frontend, backends, PostgreSQL) and run it on **AWS** with a full DevOps platform — containers, Terraform, Ansible, CI, Helm, Argo CD, Istio, Liquibase, and observability.
Application source is based on [Google’s Bank of Anthos](https://github.com/GoogleCloudPlatform/bank-of-anthos) sample. Google Cloud / GKE deploy paths were **removed on purpose** so this repo stays the **app** layer only. Platform work lives in sibling repos.
## What this repo is (and is not)
| This repo (`bank-app`) | Not in this repo |
| ---------------------- | ---------------- |
| Service source under `src/` | VPC, EKS, IAM, RDS (→ `bank-infra`) |
| Dockerfiles / Compose (you own these) | Helm charts + Argo CD apps (→ `bank-gitops`) |
| Liquibase changeSets (as you add them) | Bastion / collector config (→ `bank-ansible`) |
| CI that builds images and bumps gitops | In-cluster Google manifests / Anthos extras |
**Rule:** do not `kubectl apply` this app to production by hand. Images go to ECR; desired state lives in `bank-gitops`; Argo CD deploys.
## Related repositories
| Repository | Purpose |
| ---------- | ------- |
| `bank-app` (this) | Application code, container build, app CI |
| `bank-infra` | Terraform: VPC, subnets, IAM/IRSA, EKS, ECR, Route53, Secrets Manager, RDS |
| `bank-gitops` | Helm values, Argo CD Applications, Istio policies, observability |
| `bank-ansible` | Bastion / OS / optional LogicMonitor collector configuration |
CI choice (pick **one**): **GitHub Actions** or **Jenkins**. Terraform and Ansible are used **together** (not either/or).
## Architecture (application)
| Service | Language | Description |
| ------- | -------- | ----------- |
| [frontend](src/frontend) | Python | HTTP UI: login, signup, home |
| [ledger-writer](src/ledger/ledgerwriter) | Java | Validates transactions, writes to the ledger |
| [balance-reader](src/ledger/balancereader) | Java | Cached balances from `ledger-db` |
| [transaction-history](src/ledger/transactionhistory) | Java | Cached history from `ledger-db` |
| [ledger-db](src/ledger/ledger-db) | PostgreSQL | Transaction ledger (local/dev; RDS in AWS) |
| [user-service](src/accounts/userservice) | Python | Accounts + auth; issues JWTs |
| [contacts](src/accounts/contacts) | Python | Saved payees for transfers |
| [accounts-db](src/accounts/accounts-db) | PostgreSQL | User accounts (local/dev; RDS in AWS) |
| [loadgenerator](src/loadgenerator) | Python/Locust | Synthetic traffic for demos and SLOs |
## Target platform (AWS)
Spine we are building toward:
1. **Terraform** — VPC (public / private-app / private-data), IAM/IRSA, EKS, ECR, Route53, Secrets Manager, RDS  
2. **Ansible** — bastion and optional agents (after Terraform creates hosts)  
3. **Ingress** — ALB Controller, ExternalDNS, cert-manager, External Secrets  
4. **Istio** — mTLS + AuthorizationPolicies between services (core for this multi-service app)  
5. **CI** — test → build → push digest to ECR → update `bank-gitops`  
6. **Helm + Argo CD** — only path that deploys the bank to the cluster  
7. **Liquibase** — schema migrations gated with the app release against RDS  
8. **Observability** — OpenTelemetry → Prometheus/Grafana; Fluent Bit → ELK; LogicMonitor optional  
## Current status
- [x] App source retained under `src/`  
- [x] Google platform shortcuts removed (`iac/`, `kubernetes-manifests/`, `extras/`, Skaffold, upstream CI)  
- [ ] Own Dockerfiles + Docker Compose (local)  
- [ ] `bank-infra` Terraform apply (EKS + ECR + RDS)  
- [ ] `bank-ansible` bastion playbooks  
- [ ] Cluster add-ons + Istio  
- [ ] CI → ECR → gitops  
- [ ] Helm + Argo CD sync  
- [ ] Liquibase on RDS  
- [ ] Prometheus/Grafana + ELK  
## Getting started (local — Week 1)
> Goal: run the bank on your laptop with **your** images. No AWS required yet.
1. Clone this repo and ensure Docker Desktop (or Engine + Compose) is installed.  
2. Add a `Dockerfile` (and `.dockerignore`) for each service you will run.  
3. Add a root `docker-compose.yml` that starts frontend, supporting services, and Postgres for accounts + ledger.  
4. Provide a JWT secret and DB URLs via Compose env files (never commit real secrets).  
5. `docker compose up --build` and open the UI; sign in with the demo users once DBs are seeded.
Detailed Compose wiring will live in this README (or `docs/local.md`) as Week 1 is completed.
## AWS deploy path (high level)
1. Apply `bank-infra` (network → EKS → ECR → RDS).  
2. Configure bastion with `bank-ansible` if needed.  
3. Install cluster add-ons + Istio via `bank-gitops` / bootstrap.  
4. CI on this repo pushes images to ECR and commits digests into `bank-gitops`.  
5. Argo CD syncs Helm releases; Liquibase Job/PreSync migrates RDS; UI is served through ALB + mesh.
Step-by-step commands belong in `bank-infra` and `bank-gitops` READMEs as those repos are filled in.
## Demo credentials
Upstream Bank of Anthos ships demo users when databases are initialized with sample data (exact usernames/passwords depend on how you seed Postgres locally or via Liquibase). Document the credentials you use for local Compose here once seed is working:
| User | Password | Notes |
| ---- | -------- | ----- |
| _TBD after local seed_ | _TBD_ | Do not use in any shared/prod account |
## License
Application code retains the upstream [Apache 2.0](LICENSE) license from Bank of Anthos where applicable. Platform repos you create (`bank-infra`, `bank-gitops`, `bank-ansible`) can use whatever license you choose for your learning work.
## Credits
- Original sample: [GoogleCloudPlatform/bank-of-anthos](https://github.com/GoogleCloudPlatform/bank-of-anthos)  
- This fork: learning vehicle for an AWS-centered DevOps platform (EKS, GitOps, mesh, observability) — not an official Google product.
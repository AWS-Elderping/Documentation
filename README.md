# ElderPing Documentation Repository

This repository contains the presentations, architectural diagrams, and documentation resources for the **ElderPing Platform**.

## Included Assets in this Repository
- **Presentation**: [ElderPing_Premium.pptx](file:///d:/third-review/ElderPinq/Documentation/ElderPing_Premium.pptx)
- **Architecture PDF**: [ARCHITECTURE_DOCUMENTATION.pdf](file:///d:/third-review/ElderPinq/Documentation/ARCHITECTURE_DOCUMENTATION.pdf)
- **Application Flow Diagram**: ![Application Flow Diagram](file:///d:/third-review/ElderPinq/Documentation/application.png)
- **Infrastructure / System Architecture Diagram**: ![Infrastructure Diagram](file:///d:/third-review/ElderPinq/Documentation/final%20inne%20ela.png)

## Main Project Documentation Files
- **Complete Technical Documentation**: [ELDERPING_DOCUMENTATION.md](file:///d:/third-review/ElderPinq/ELDERPING_DOCUMENTATION.md)
- **Architecture & Technical Details**: [ARCHITECTURE_DOCUMENTATION.md](file:///d:/third-review/ElderPinq/ARCHITECTURE_DOCUMENTATION.md)
- **Repository Reorganization Migration Report**: [MIGRATION_REPORT.md](file:///d:/third-review/ElderPinq/MIGRATION_REPORT.md)
- **Project Root Overview & Quick Start**: [README.md](file:///d:/third-review/ElderPinq/README.md)

---

# ElderPing Platform — Complete Technical Documentation

> **Platform:** AWS EKS + ArgoCD GitOps | **Stack:** Node.js microservices, React frontend, PostgreSQL, Terraform IaC

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Multi-Repo Structure](#2-multi-repo-structure)
3. [Infrastructure Layer — Terraform](#3-infrastructure-layer--terraform)
4. [Application Layer — Microservices](#4-application-layer--microservices)
5. [Request Flow — End to End](#5-request-flow--end-to-end)
6. [CI Pipeline — Line by Line](#6-ci-pipeline--line-by-line)
7. [CD Pipeline — Line by Line](#7-cd-pipeline--line-by-line)
8. [GitOps with ArgoCD](#8-gitops-with-argocd)
9. [Helm Charts](#9-helm-charts)
10. [Terraform CI Pipeline — Line by Line](#10-terraform-ci-pipeline--line-by-line)
11. [Monitoring & Observability](#11-monitoring--observability)
12. [Security Architecture](#12-security-architecture)
13. [Why These Tools?](#13-why-these-tools)

---

## 1. Architecture Overview

ElderPing is an elderly care platform that allows families to monitor the health of their elderly relatives. The platform is built on a microservices architecture deployed on AWS EKS, managed through GitOps using ArgoCD.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         USER DEVICES                                │
│              Browser / Mobile (elderping.online)                    │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ HTTPS
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   AWS APPLICATION LOAD BALANCER                     │
│              (WAF protected, SSL terminated, multi-AZ)              │
│         Routes by path prefix: /api/auth, /api/health, etc.         │
└──────┬────────────────────────────────────────────────┬─────────────┘
       │                                                │
       ▼                                                ▼
┌─────────────┐                             ┌──────────────────────┐
│  Frontend   │                             │   Backend Services   │
│  (React)    │                             │  (12 microservices)  │
│  on EKS     │                             │  on EKS private      │
└─────────────┘                             └──────────────────────┘
                                                       │
                                                       ▼
                                            ┌──────────────────────┐
                                            │  PostgreSQL on RDS   │
                                            │  (private subnets)   │
                                            └──────────────────────┘
```

**Core design principles:**
- All backend services run in **private subnets** — never directly internet-facing
- All inter-service communication stays within the VPC
- Secrets never appear in code — stored in AWS Secrets Manager, injected via External Secrets Operator
- Every deployment is a Git commit — no manual `kubectl apply` ever runs in production

---

## 2. Multi-Repo Structure

The project is split across multiple repositories. Each has a single responsibility.

| Repository | Purpose |
|------------|---------|
| `elderping-frontend` | React UI application |
| `elderping-auth-service` | User auth, JWT issuance, family linking |
| `elderping-health-service` | Health check-ins, vitals, timelines |
| `elderping-reminder-service` | Medication reminders and schedules |
| `elderping-notes-service` | Clinical and family notes |
| `elderping-alert-service` | Emergency alerts and notifications |
| `elderping-ai-service` | AWS Bedrock AI integration |
| `elderping-audit-service` | Security audit log ledger |
| `elderping-finops-service` | Cloud cost analytics |
| `elderping-notification-service` | Push/email/SMS notifications |
| `elderping-appointment-service` | Care appointments scheduling |
| `elderping-report-service` | Health report generation |
| `elderping-gitops` | ArgoCD GitOps config — Helm values, ArgoCD apps |
| `elderping-infra` | Terraform infrastructure-as-code |

**Why separate repos?**
Each service has its own CI/CD pipeline, its own release cycle, and its own team ownership. A bug in the notes-service does not require rebuilding the auth-service. Changes to one service cannot accidentally break another service's pipeline.

---

## 3. Infrastructure Layer — Terraform

All AWS infrastructure is defined as code in `elderping-infra`. The state is stored remotely in S3 with DynamoDB locking to prevent concurrent modifications.

### State Backend

```hcl
backend "s3" {
  bucket         = "elderpinq-462355914183-tfstate"
  key            = "dev/terraform.tfstate"
  region         = "us-east-1"
  dynamodb_table = "elderpinq-tfstate-lock"
  encrypt        = true
}
```

- **S3 bucket** stores the `terraform.tfstate` file — the source of truth for what Terraform has created
- **DynamoDB table** provides state locking — prevents two people or two pipeline runs from running `terraform apply` simultaneously, which would corrupt the state
- **`encrypt = true`** — state file is encrypted at rest using AWS-managed keys

### Module Breakdown

#### Module 1: VPC (Virtual Private Cloud)

**What it does:** Creates the entire network foundation — the private network inside AWS where everything runs.

**Resources created:**
- 1 VPC with DNS enabled
- 2 public subnets across 2 Availability Zones
- 2 private subnets across 2 Availability Zones
- 1 Internet Gateway (allows public subnets to reach internet)
- 2 NAT Gateways (one per AZ — allows private subnets to reach internet for outbound traffic only)
- 2 Elastic IPs for NAT Gateways
- Route tables for public and private subnets
- VPC Flow Logs → CloudWatch (all network traffic logged, KMS encrypted)

**Why 2 AZs?** High availability — if one AWS data center goes down, services in the other AZ continue running.

**Why NAT Gateways in private subnets?** Private subnet resources (EKS nodes, RDS) need to pull from the internet (Docker images, npm packages) but should never be reachable from the internet directly. NAT allows outbound-only traffic.

**Why VPC Flow Logs?** Every network packet in/out of the VPC is recorded — essential for security incident investigation and compliance audits.

**Alternatives considered:**
- AWS Transit Gateway — for multi-VPC routing (not needed at this scale)
- VPC Peering — for connecting multiple VPCs (single VPC is sufficient here)

---

#### Module 2: EKS (Elastic Kubernetes Service)

**What it does:** Creates the managed Kubernetes cluster where all microservices run.

**Why Kubernetes?**
- Run 12+ microservices without managing individual EC2 instances
- Automatic healing — if a pod crashes, Kubernetes restarts it
- Horizontal scaling — add more pod replicas under load
- Declarative configuration — define desired state, Kubernetes enforces it

**Why EKS instead of self-managed Kubernetes?**
AWS manages the Kubernetes control plane (API server, etcd, scheduler). You only manage the worker nodes. This eliminates the highest-risk part of Kubernetes operations.

**Alternatives:**
- ECS (Elastic Container Service) — AWS-native, simpler, but less portable and no Helm ecosystem
- EC2 with Docker Compose — not scalable, no self-healing
- App Runner — too limited for complex multi-service networking

---

#### Module 3: RDS (Relational Database Service)

**What it does:** Creates managed PostgreSQL databases for the services.

**Key configuration:**
```hcl
engine         = "postgres"
engine_version = "15.18"
instance_class = "db.t3.micro"
multi_az       = var.environment == "prod" ? true : false
storage_encrypted = true
kms_key_id     = aws_kms_key.rds.arn
```

- **PostgreSQL 15** — proven, feature-rich relational database
- **Multi-AZ in production** — automatic failover if the primary DB instance fails (zero-downtime)
- **KMS encryption** — all data encrypted at rest
- **Security group** — only allows connections from within the VPC (port 5432), never from internet
- **Private subnets** — RDS instances are in private subnets, unreachable directly from internet

**Why separate databases per service?**
Each microservice owns its own database schema. The reminder-service cannot accidentally read or corrupt the health-service's data. This is the database-per-service microservices pattern.

**Alternatives:**
- Aurora Serverless — scales to zero, cost-efficient for variable workloads (upgrade path)
- DynamoDB — NoSQL, good for high-throughput key-value, but this app needs relational joins
- Self-managed PostgreSQL on EC2 — no managed backups, patching, or failover

---

#### Module 4: ALB (Application Load Balancer)

**What it does:** The single entry point for all HTTP/HTTPS traffic. Routes requests to the correct Kubernetes service based on the URL path.

**Path routing rules:**
```
/api/auth/*      → auth-service
/api/health/*    → health-service
/api/reminder/*  → reminder-service
/api/notes/*     → notes-service
/api/finops/*    → finops-service
/*               → frontend
```

**Why ALB over NLB (Network Load Balancer)?**
ALB operates at Layer 7 (HTTP) — it understands URLs, paths, and headers. NLB operates at Layer 4 (TCP) — it only sees IP and port. Path-based routing requires Layer 7.

**Why one ALB for all services?**
Cost efficiency — one ALB costs ~$16/month. One per service would be ~$200/month for 12 services. The AWS Load Balancer Controller (running in EKS) manages routing rules for all Kubernetes Ingress resources under a single ALB using an `IngressGroup`.

---

#### Module 5: ACM (AWS Certificate Manager)

**What it does:** Issues and auto-renews the SSL/TLS certificate for `elderping.online` and `*.elderping.online`.

**Why ACM?** Free, auto-renews, integrates natively with ALB. No manual certificate management.

**Alternatives:** Let's Encrypt with cert-manager in Kubernetes — works but requires more operational overhead.

---

#### Module 6: Route 53

**What it does:** DNS management for `elderping.online`. Maps domain names to the ALB.

**Records created:**
- `elderping.online` → ALB
- `www.elderping.online` → ALB
- `api.elderping.online` → ALB
- `argocd.elderping.online` → ALB

**Why Route 53?** Native AWS DNS, integrates with ACM for DNS validation, low latency global DNS resolution.

---

#### Module 7: ECR (Elastic Container Registry)

**What it does:** Private Docker image registry in AWS. Each service has its own repository.

**Example repositories:**
- `elderpinq-dev-auth-service`
- `elderpinq-dev-health-service`
- `elderpinq-dev-frontend`

**Why ECR?** Native IAM authentication — no username/password for Docker login. EKS nodes can pull images using their IAM role. Images are private and encrypted.

**Alternatives:** Docker Hub (public, rate-limited), GitHub Container Registry (good alternative, but adds external dependency for AWS deployments).

---

#### Module 8: Secrets Manager

**What it does:** Stores sensitive values (database passwords, JWT secrets, API keys) encrypted in AWS.

**Why not environment variables in the code?** Environment variables in code or git files are a security risk. Secrets Manager provides encryption at rest, access logging, automatic rotation, and fine-grained IAM access control.

**How secrets reach the pods:**
```
AWS Secrets Manager
        ↓ (External Secrets Operator polls every 1 hour)
Kubernetes Secret
        ↓ (mounted as env var in Deployment)
Pod environment variable
```

---

#### Module 9: WAF (Web Application Firewall)

**What it does:** Sits in front of the ALB and inspects all incoming HTTP requests. Blocks common web attacks before they reach any service.

**Protects against:**
- SQL Injection
- Cross-site scripting (XSS)
- Known bad IP addresses
- Rate limiting (prevents DDoS)

**Why WAF?** Healthcare data requires protection against OWASP Top 10 vulnerabilities. WAF provides a managed ruleset that updates automatically as new attack patterns emerge.

---

#### Module 10: CloudFront

**What it does:** CDN (Content Delivery Network) for the frontend static assets (React build files stored in S3).

**Flow:** User browser → CloudFront edge location (nearest to user) → S3 bucket

**Why CloudFront?** Static React files (HTML, JS, CSS) are cached at 400+ edge locations worldwide. Users in Europe get files from a European edge location instead of US-East-1, reducing latency significantly.

---

#### Module 11: CloudTrail

**What it does:** Records every AWS API call made in the account — who did what, when, from where.

**Configuration:**
- Multi-region trail — captures activity in all AWS regions
- Log file validation — detects if log files are tampered with
- KMS encryption — logs encrypted at rest
- S3 delivery — logs stored in a dedicated S3 bucket

**Why CloudTrail?** Compliance requirement and forensic capability. If a security incident occurs, CloudTrail shows exactly which IAM user or role made which API calls.

---

#### Module 12: GuardDuty

**What it does:** Machine learning-based threat detection. Continuously analyzes CloudTrail logs, VPC Flow Logs, and DNS logs to detect anomalies.

**Enabled data sources:**
- S3 access logs — detects unusual data access patterns
- Kubernetes audit logs — detects pod privilege escalation, unusual API calls

**Example threats it detects:**
- Cryptocurrency mining on EKS nodes
- Compromised IAM credentials making API calls from unusual locations
- Unusual database access patterns

**Alternatives:** Third-party SIEM tools (Splunk, Datadog Security) — more feature-rich but significantly more expensive.

---

#### Module 13: Security Hub

**What it does:** Aggregates security findings from GuardDuty, Inspector, and AWS Config into a single dashboard. Provides a security score against CIS AWS Foundations Benchmark.

**Why Security Hub?** Without it, security findings are scattered across 3+ different AWS consoles. Security Hub normalizes and centralizes them with security scoring.

---

#### Module 14: AWS Inspector

**What it does:** Vulnerability scanning for ECR container images and EC2 instances.

**Scans for:**
- Known CVEs (Common Vulnerabilities and Exposures) in container base images
- Software packages with known vulnerabilities
- OS-level vulnerabilities on EC2

**Why Inspector?** Containers are built on base images (Node.js, Alpine, etc.) that may have known vulnerabilities. Inspector automatically scans every new image pushed to ECR.

---

#### Module 15: AWS Config

**What it does:** Continuously monitors and records AWS resource configurations. Evaluates resources against compliance rules.

**Example rules:**
- RDS instances must be encrypted
- S3 buckets must not be public
- Security groups must not allow unrestricted SSH

**Why AWS Config?** Config provides a configuration history — you can see what an S3 bucket's policy looked like 3 months ago. Essential for compliance audits (HIPAA, SOC 2).

---

#### Module 16: AWS Budgets

**What it does:** Monthly cost alerts.

**Two alerts configured:**
- 80% of monthly budget reached → email notification
- 100% of monthly budget forecasted → email notification

**Why Budgets?** Prevents surprise bills. The 100% forecasted alert triggers before you actually overspend.

---

#### Module 17: Cost Explorer Anomaly Detection

**What it does:** Uses ML to detect unusual cost spikes per AWS service.

**Configuration:** Daily email alert when any service shows a cost anomaly greater than $50.

**Why?** Without anomaly detection, a runaway Lambda function or accidentally public S3 bucket could generate thousands in unexpected costs before anyone notices.

---

#### Module 18: SNS, SQS, EventBridge, Lambda

- **SNS (Simple Notification Service)** — pub/sub messaging for alert notifications
- **SQS (Simple Queue Service)** — message queue for async processing between services
- **EventBridge** — event bus that triggers Lambda on scheduled events
- **Lambda** — serverless function for report generation on schedule

**Request flow for reports:**
```
EventBridge (scheduled) → Lambda (trigger) → Report Service → S3 (store PDF)
```

---

## 4. Application Layer — Microservices

### How Authentication Works

ElderPing uses local JWT (JSON Web Token) authentication with HS256 signing. No external identity provider.

**Login flow:**
```
1. User submits username + password to POST /api/auth/login
2. auth-service validates password against bcrypt hash in PostgreSQL
3. auth-service queries family_links table for FAMILY users
4. auth-service signs a JWT containing:
   {
     userId, email, role,
     linkedElders: ["elder-uuid-1", "elder-uuid-2"]  ← embedded at login
   }
5. JWT returned to browser, stored in localStorage
6. Every subsequent request sends: Authorization: Bearer <token>
7. Each service validates the JWT signature locally (no auth-service round-trip)
```

**Why embed linkedElders in the JWT?**
Each service can verify the family-elder relationship without making an HTTP call to auth-service. This keeps services independent and eliminates a network hop on every request.

**Important:** If a new elder is linked in the same session, the JWT does not automatically update. The user must log out and log back in to get a new JWT containing the newly linked elder.

---

### Service-by-Service Breakdown

#### auth-service
**What it does:** User registration, login, JWT issuance, family-elder linking, role management.

**Key endpoints:**
- `POST /register` — create account (ELDER or FAMILY role)
- `POST /login` — authenticate, return JWT
- `POST /link` — link a FAMILY user to an ELDER via invite code
- `GET /me` — return current user profile from JWT
- `PATCH /admin/users/:id/role` — ADMIN only: change a user's role

**Database:** `users_db_dev` (PostgreSQL) — tables: `users`, `family_links`

**Why it exists:** Centralizes all identity and access concerns. Other services trust the JWT it issues without needing to know about users directly.

---

#### health-service
**What it does:** Records and retrieves health check-ins for elders (blood pressure, heart rate, blood oxygen, weight, mood, steps).

**Key endpoints:**
- `POST /health/checkin` — submit a health check-in
- `GET /health/:userId` — get health history for an elder
- `GET /health/:userId/timeline` — get timeline of health events

**Authorization:** Checks `linkedElders` in JWT — a FAMILY user can only view elders in their linked list.

**Database:** `health_db_dev`

---

#### reminder-service
**What it does:** Creates and manages medication reminders for elders.

**Key endpoints:**
- `POST /reminders` — create a reminder (`{ userId, medicationName, dosage, frequency, scheduledTime }`)
- `GET /reminders/:userId` — list reminders for an elder
- `PUT /reminders/:id/take` — mark a medication as taken

**Frequency options:** `DAILY`, `TWICE_DAILY`, `WEEKLY`, `AS_NEEDED`

**Authorization:** Uses JWT `linkedElders` claim — no database fallback. Must re-login after linking a new elder.

**Database:** `reminder_db_dev`

---

#### notes-service
**What it does:** Creates and retrieves clinical notes about elders. Notes can be written by family, caregivers, doctors, or AI.

**Note categories:** `PATIENT`, `FAMILY`, `CAREGIVER`, `DOCTOR`, `AI`

**Key endpoints:**
- `POST /notes` — create a note
- `GET /notes/:elderId` — get all notes for an elder
- `POST /notes/ai` — generate an AI-powered clinical summary using ai-service

**Database:** `users_db_dev` (same as auth-service — allows JOIN with `users` table for author names)

**Important implementation detail:** `author_id` is stored as `TEXT` while `users.id` is `UUID`. All JOIN conditions use `ON n.author_id = u.id::text` to cast UUID to TEXT for comparison. Without this cast, PostgreSQL throws `operator does not exist: text = uuid` at query compilation.

---

#### alert-service
**What it does:** Manages emergency alerts. When an elder triggers an SOS or health threshold is breached, this service notifies linked family members.

**Key endpoints:**
- `POST /alerts` — create an emergency alert
- `GET /alerts/:userId` — get alerts for an elder

**Integration:** Calls notification-service to send push/email/SMS notifications to family members.

---

#### ai-service
**What it does:** Integrates with AWS Bedrock (Claude 3 Haiku) for AI-powered health analysis.

**Capabilities:**
- `symptom_check` — analyze reported symptoms
- `risk_analysis` — assess health risk from vitals
- `clinical_summary` — generate clinical notes summary

**Why AWS Bedrock?** Managed AI inference — no model hosting infrastructure. Pay per token. Claude 3 Haiku is fast and cost-effective for clinical summaries.

**Alternatives:** OpenAI API — works but adds external vendor dependency outside AWS ecosystem. Self-hosted models on EC2 — too expensive and operationally complex.

---

#### finops-service
**What it does:** Cloud cost analytics dashboard for SUPER_ADMIN users.

**Key endpoints:**
- `GET /finops/dashboard` — total cost, forecast, top cost drivers, recommendation summary
- `GET /finops/recommendations` — AI-generated cost optimization recommendations
- `POST /finops/recommendations/:id/apply` — mark a recommendation as applied
- `POST /finops/recommendations/:id/dismiss` — dismiss a recommendation

**Provider pattern:** Can use either `mock` (default) or `aws` (real Cost Explorer API) via `FINOPS_PROVIDER` env var.

**Authorization:** Requires `FINOPS_READ` or `FINOPS_MANAGE` permission. Only `SUPER_ADMIN` role has both.

**In-memory cache:** Cost data cached for 15 minutes to avoid excessive AWS Cost Explorer API calls.

---

#### audit-service
**What it does:** Records all significant user actions for compliance and security auditing.

**Every service calls audit-service** after significant operations:
```js
await fetch(`${auditServiceUrl}/audit`, {
  method: 'POST',
  body: JSON.stringify({ actionType, resource, resourceId, status, message })
});
```

**Audit actions recorded include:** `CREATE_NOTE`, `DELETE_NOTE`, `VIEW_DASHBOARD`, `APPLY_RECOMMENDATION`, `ROLE_ELEVATION_ATTEMPT`

**Why a separate service?** Audit concerns are cross-cutting. A dedicated service ensures consistent audit records across all services and provides a tamper-evident log.

---

#### notification-service
**What it does:** Sends push notifications, emails (via SES), and SMS (via SNS) to users.

**Called by:** alert-service, reminder-service, appointment-service

---

#### appointment-service
**What it does:** Schedules and manages care appointments between elders and caregivers/doctors.

---

#### report-service
**What it does:** Generates PDF health reports for elders. Triggered by EventBridge on a schedule, stores results in S3.

---

### Role-Based Access Control

| Role | Permissions |
|------|-------------|
| `ELDER` | View own health data, create notes about self |
| `FAMILY` | View linked elders' health data, notes, reminders |
| `CAREGIVER` | Same as FAMILY + create clinical notes |
| `ADMIN` | Manage users, view all data |
| `SUPER_ADMIN` | All ADMIN permissions + FINOPS_READ + FINOPS_MANAGE |

---

## 5. Request Flow — End to End

### Example: Family member views elder's health data

```
1. Browser → GET https://elderping.online/api/health/elder-uuid
   │
   ▼
2. AWS WAF — inspects request, checks rules (SQL injection, XSS, rate limits)
   │
   ▼
3. ALB — path /api/health/* → routes to health-service ClusterIP service in EKS
   │
   ▼
4. EKS Ingress Controller — forwards to health-service pod
   │
   ▼
5. health-service — validateToken middleware:
   │   a. Extracts Bearer token from Authorization header
   │   b. Verifies JWT signature using JWT_SECRET
   │   c. Decodes payload: { userId, role: 'FAMILY', linkedElders: ['elder-uuid'] }
   │
   ├── checkRelationship middleware:
   │   a. Checks if elder-uuid is in req.user.linkedElders array
   │   b. If not found → 403 Forbidden
   │   c. If found → next()
   │
   └── Handler:
       a. Queries health_db_dev: SELECT * FROM health_logs WHERE user_id = $1
       b. Returns JSON array of health records
   │
   ▼
6. Response flows back: pod → EKS → ALB → WAF → Browser

Total path: Browser → WAF → ALB → EKS → Service → RDS → back
```

### Example: Developer pushes code to trigger deployment

```
1. Developer: git push origin main  (in elderping-auth-service)
   │
   ▼
2. GitHub Actions: CI Pipeline triggered
   │
   ├── prepare job: npm install, lint, unit tests
   ├── quality job: SonarCloud scan, Snyk security scan
   ├── build job:   docker build -t ECR_REGISTRY/elderpinq-dev-auth-service:SHA .
   ├── trivy job:   scan Docker image for CVEs
   ├── push job:    docker push ECR_REGISTRY/elderpinq-dev-auth-service:SHA
   └── release job: release note
   │
   ▼
3. CD Pipeline triggered (workflow_run: CI completed)
   │
   ├── Checkout elderping-gitops repo
   ├── yq: update environments/dev/auth-service-values.yaml image.tag = SHA
   └── git push origin main (to gitops repo)
   │
   ▼
4. ArgoCD detects change in elderping-gitops repo (polls every ~3 min)
   │
   ├── Compares desired state (gitops) vs actual state (cluster)
   ├── Detects image tag changed
   └── Runs: kubectl apply (new Deployment with new image tag)
   │
   ▼
5. Kubernetes rolling update:
   ├── Starts new pod with new image
   ├── Waits for readiness probe (/ready) to pass
   ├── Routes traffic to new pod
   └── Terminates old pod
   │
   ▼
6. New version live — zero downtime
```

---

## 6. CI Pipeline — Line by Line

Every microservice has an identical CI pipeline structure. Below is the auth-service CI explained line by line.

```yaml
name: CI Pipeline
```
The name shown in GitHub Actions UI.

```yaml
on:
  push:
    branches: [ main, develop, feature/test ]
  pull_request:
```
**Trigger:** Runs on every push to `main`, `develop`, or `feature/test` branches, AND on every pull request (regardless of branch). Pull requests trigger CI to validate the code before merge.

```yaml
permissions:
  id-token: write
  contents: read
```
**OIDC permissions:** `id-token: write` allows GitHub Actions to request an OIDC token from GitHub's identity provider. This token is exchanged with AWS STS to assume the `AWS_ROLE_ARN` role — no static AWS keys stored anywhere. `contents: read` allows reading the repository code.

---

### Job 1: prepare

```yaml
prepare:
  name: ci / prepare-auth-service
  runs-on: ubuntu-latest
```
Runs on a fresh Ubuntu VM provided by GitHub.

```yaml
  - name: Checkout
    uses: actions/checkout@v4
```
Clones the repository code into the runner.

```yaml
  - name: Setup Node
    uses: actions/setup-node@v4
    with:
      node-version: '18'
```
Installs Node.js 18 — the same version used in production Dockerfiles.

```yaml
  - name: Install Dependencies
    run: |
      npm install --package-lock-only
      npm ci
```
`npm install --package-lock-only` regenerates the lockfile if missing. `npm ci` installs exact dependency versions from `package-lock.json` — faster and deterministic compared to `npm install`.

```yaml
  - name: Lint
    run: npm run lint --if-present
```
Runs ESLint if a lint script exists. `--if-present` prevents failure if no lint script is defined. Catches syntax errors and style violations early.

```yaml
  - name: Unit Tests
    run: npm test --if-present
```
Runs the test suite. `--if-present` skips gracefully if no tests exist yet.

---

### Job 2: quality (needs: prepare)

Only runs after `prepare` succeeds. The `needs:` keyword creates a dependency chain.

```yaml
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```
Exposes the Snyk API token as an environment variable for the Snyk scan step.

```yaml
  - name: SonarCloud Scan
```
Downloads SonarScanner CLI and runs a code quality analysis against SonarCloud. Checks for:
- Code duplication
- Cognitive complexity
- Bug patterns
- Security hotspots

The `sonar.qualitygate.wait=true` flag makes the scanner wait for SonarCloud to process results before continuing. If the quality gate fails, the step is treated as a soft failure (the pipeline continues) due to the error handling in the shell script.

```yaml
  - name: Snyk Security Scan
    if: ${{ env.SNYK_TOKEN != '' }}
    continue-on-error: true
    uses: snyk/actions/node@master
```
Scans `node_modules` for known vulnerabilities. `if: SNYK_TOKEN != ''` skips the step if the secret is not configured. `continue-on-error: true` means a Snyk finding will not block the pipeline — it reports but does not fail.

---

### Job 3: build (needs: quality)

```yaml
  - name: Configure AWS credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
      aws-region: us-east-1
```
Uses OIDC to assume the AWS IAM role. GitHub exchanges the OIDC token for temporary AWS credentials (valid ~1 hour). No permanent AWS keys stored in GitHub secrets.

```yaml
  - name: Login to Amazon ECR
    id: login-ecr
    uses: aws-actions/amazon-ecr-login@v2
```
Authenticates Docker with ECR using the temporary AWS credentials obtained above. Sets output `registry` (the ECR registry URL).

```yaml
  - name: Build Docker Image
    env:
      IMAGE_TAG: ${{ github.sha }}
    run: |
      docker build -t $REGISTRY/$REPOSITORY:$IMAGE_TAG .
      docker save -o image.tar $REGISTRY/$REPOSITORY:$IMAGE_TAG
```
Builds the Docker image tagged with the **full Git commit SHA** (40 characters, e.g. `abc123def456...`). Using the commit SHA as the tag means every image is uniquely and immutably identified — you can always trace a running container back to the exact commit that produced it.

`docker save` exports the built image to a `.tar` file for passing between jobs (jobs run on different VMs and cannot share the Docker daemon).

```yaml
  - name: Upload Docker Image Artifact
    uses: actions/upload-artifact@v4
    with:
      name: docker-image
      path: image.tar
      retention-days: 1
```
Uploads `image.tar` to GitHub's artifact storage. Subsequent jobs (trivy, push) download it. Retained for 1 day only since it's a large binary — no need to keep it after the pipeline completes.

---

### Job 4: trivy (needs: build)

```yaml
  - name: Trivy Image Scan
    uses: aquasecurity/trivy-action@master
    with:
      image-ref: '462355914183.dkr.ecr.us-east-1.amazonaws.com/elderpinq-dev-auth-service:${{ github.sha }}'
      format: 'table'
      exit-code: '0'
      severity: 'CRITICAL,HIGH'
```
Trivy scans the Docker image for known CVEs in:
- OS packages (apt/apk)
- Language runtime packages (npm)
- Application dependencies

`exit-code: '0'` means Trivy reports findings but does not fail the pipeline. `severity: 'CRITICAL,HIGH'` filters output to only show critical and high vulnerabilities — noise from LOW/MEDIUM is excluded.

---

### Job 5: push (needs: trivy)

Downloads the artifact, loads it into Docker, and pushes to ECR:
```yaml
  - name: Push Docker Image
    run: docker push 462355914183.dkr.ecr.us-east-1.amazonaws.com/elderpinq-dev-auth-service:${{ github.sha }}
```
The image is now available in ECR with the commit SHA tag, ready for the CD pipeline to reference.

---

### Jobs 6 & 7: release, notify

Informational steps that print success messages. These exist as explicit pipeline stages to make the GitHub Actions UI show a clear visual flow of all stages.

---

## 7. CD Pipeline — Line by Line

```yaml
on:
  workflow_run:
    workflows: [ "CI Pipeline" ]
    types:
      - completed
```
**Trigger:** CD runs automatically when the CI pipeline completes — not on push directly. This creates a clear separation: CI is about building and testing; CD is about deploying. CD only runs if CI finishes (the `conclusion == 'success'` check below ensures it only deploys on CI success).

```yaml
jobs:
  prepare:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
```
Guard condition — if CI failed (tests failed, build failed, security scan blocked), CD does not run. No broken code reaches the cluster.

---

### Deploy Job

```yaml
  - name: Checkout GitOps Repo
    uses: actions/checkout@v4
    with:
      repository: AWS-Elderping/elderping-gitops
      token: ${{ secrets.GITOPS_PAT }}
      ref: main
      path: gitops-repo
```
Checks out the **gitops repo** (not the service repo). `GITOPS_PAT` is a GitHub Personal Access Token with write access to the gitops repo — needed because a workflow in repo A cannot push to repo B with the default `GITHUB_TOKEN`.

```yaml
  - name: Setup yq
    uses: mikefarah/yq@v4.44.2
```
`yq` is a YAML processor (like `jq` for JSON). Used to update a single field in the Helm values YAML without parsing the entire file manually.

```yaml
      BRANCH="${{ github.event.workflow_run.head_branch }}"
      SHA="${{ github.event.workflow_run.head_sha }}"

      if [ "$BRANCH" = "main" ] || [ "$BRANCH" = "develop" ] || [ "$BRANCH" = "feature/test" ]; then
        ENV="dev"
      fi
```
Maps the source branch to a deployment environment. `main`, `develop`, and `feature/test` all deploy to `dev`. A future `prod` branch would map to `prod`.

```yaml
      yq e -i ".image.tag = \"$SHA\"" "$VALUES_FILE"
```
In-place updates `environments/dev/auth-service-values.yaml`:
```yaml
# Before:
image:
  tag: "abc123..."

# After:
image:
  tag: "def456..."   ← new commit SHA
```
This is the only change made to the gitops repo. ArgoCD detects this single line change and triggers a rolling deployment.

```yaml
      for i in {1..10}; do
        git pull origin HEAD
        # ... make change ...
        git push origin HEAD
        if [ $? -eq 0 ]; then exit 0; fi
        sleep $(( RANDOM % 5 + 2 ))
      done
```
**Retry loop with jitter:** Multiple services deploying simultaneously can cause git push conflicts (two CDs trying to push to gitops at the same time). The loop retries up to 10 times with a random sleep (2–7 seconds) between retries. The random delay prevents two concurrent CDs from retrying at exactly the same time (thundering herd problem).

```yaml
      git commit -m "deploy: update auth-service image tag to $SHA [skip ci]"
```
`[skip ci]` in the commit message prevents the gitops repo's own CI/CD from triggering on this automated commit — it's a deployment commit, not a code change.

---

## 8. GitOps with ArgoCD

### What is GitOps?

GitOps is a deployment practice where Git is the single source of truth for what should be running in the cluster. Instead of running `kubectl apply` manually, you commit the desired state to Git. A controller (ArgoCD) continuously watches the Git repo and reconciles the cluster state to match.

**Traditional deployment:**
```
Developer → kubectl apply -f deployment.yaml → cluster
```
Problem: No audit trail. Easy to make a mistake. Hard to roll back.

**GitOps deployment:**
```
Developer → git commit → GitOps repo → ArgoCD → cluster
```
Benefit: Full audit trail in Git. Roll back = `git revert`. Cluster state is always reproducible.

---

### App-of-Apps Pattern

```yaml
# root-app.yaml
spec:
  source:
    path: argocd/applications    ← watches this directory
```

ArgoCD's root app (`elderping-root`) watches the `argocd/applications/` directory. Every `.yaml` file in that directory is itself an ArgoCD Application definition. When a new service is added, a new Application file is created in that directory and ArgoCD automatically starts managing it.

This avoids manually registering each service with ArgoCD — the root app discovers and manages all child apps automatically.

---

### Sync Waves

```yaml
annotations:
  argocd.argoproj.io/sync-wave: "10"
```

Sync waves control deployment order. Lower numbers deploy first.

| Wave | What deploys |
|------|-------------|
| 1–3  | Namespaces, CRDs |
| 4    | Prometheus (monitoring infrastructure) |
| 10   | Application services (auth, health, etc.) |
| 16   | Grafana, Loki (depends on Prometheus) |

This ensures Prometheus is running before services try to register their ServiceMonitors.

---

### How ArgoCD Detects Changes

1. ArgoCD polls the gitops repo every ~3 minutes (configurable)
2. Compares the Git desired state (Helm chart + values) against the live cluster state
3. If they differ → marks the app as `OutOfSync`
4. With `automated: selfHeal: true` → automatically syncs without human intervention
5. With `automated: prune: true` → removes Kubernetes resources that were deleted from Git

---

### Why ArgoCD over Alternatives?

| Tool | Why not |
|------|---------|
| **Flux** | Good alternative, but ArgoCD has a better UI and more mature RBAC |
| **Spinnaker** | Very complex to operate, heavyweight for a single-team setup |
| **Jenkins X** | Opinionated, complex, tied to Jenkins ecosystem |
| **Manual kubectl** | No audit trail, easy to make mistakes, not reproducible |

---

## 9. Helm Charts

### What is Helm?

Helm is the package manager for Kubernetes. Instead of maintaining 10 separate YAML files per service (Deployment, Service, Ingress, ConfigMap, HPA, PDB, ServiceAccount, etc.), Helm uses templates with variables.

**Without Helm** — auth-service needs 10 YAML files:
```
deployment.yaml      (100 lines)
service.yaml         (20 lines)
ingress.yaml         (40 lines)
configmap.yaml       (30 lines)
externalsecret.yaml  (30 lines)
hpa.yaml             (25 lines)
pdb.yaml             (15 lines)
serviceaccount.yaml  (10 lines)
networkpolicy.yaml   (30 lines)
servicemonitor.yaml  (20 lines)
```

**With Helm** — one chart shared across all services, customized per service via `values.yaml`:
```
values.yaml          ← service-specific configuration
templates/           ← shared templates for all services
```

### How Environment Overrides Work

ArgoCD renders Helm charts with two values files:
```yaml
helm:
  valueFiles:
    - values.yaml                               ← default values (chart defaults)
    - ../../environments/dev/auth-service-values.yaml  ← environment overrides
```

The environment values file only needs to override what changes per environment:
```yaml
# environments/dev/auth-service-values.yaml
env:
  dbHost: "auth-db-dev"
  dbName: "users_db_dev"
image:
  tag: "abc123def456..."    ← updated by CD pipeline on every deploy
```

Everything else (resource limits, health checks, network policies) is inherited from the default `values.yaml`.

---

### Why Helm over Alternatives?

| Alternative | Why not chosen |
|------------|---------------|
| **Kustomize** | Patch-based, harder to DRY across 12 services. ArgoCD supports both. |
| **Raw YAML** | 120+ files to maintain across 12 services. Every change replicated 12 times. |
| **Kapp** | Less ecosystem tooling, less common |

---

## 10. Terraform CI Pipeline — Line by Line

File: `.github/workflows/terraform-ci.yml`

```yaml
on:
  pull_request:
    branches: [main, develop]
    paths:
      - 'environments/**'
      - 'modules/**'
  push:
    branches: [main, develop]
    paths:
      - 'environments/**'
      - 'modules/**'
```
**Smart triggering:** Only runs when Terraform files actually change. A change to a README or workflow file does not trigger a Terraform plan. `paths` filtering saves pipeline minutes and avoids unnecessary AWS API calls.

```yaml
permissions:
  id-token: write      # Required for OIDC AWS authentication
  contents: read       # Read the repository code
  pull-requests: write # Post plan output as PR comments
```

```yaml
env:
  TF_VERSION: "1.7.0"
  WORKING_DIR: environments/dev
  AWS_REGION: us-east-1
```
Central configuration — change `TF_VERSION` once to update all jobs.

---

### Stage 1: Security Scan (Checkov)

```yaml
  scan:
    name: "1 · Security Scan (Checkov)"
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: bridgecrewio/checkov-action@v12
        with:
          directory: ${{ env.WORKING_DIR }}
          framework: terraform
          soft_fail: true
```

**What Checkov does:** Scans all `.tf` files for misconfigurations:
- S3 buckets without encryption
- Security groups with 0.0.0.0/0 ingress
- RDS instances without Multi-AZ
- IAM policies that are too permissive
- CloudTrail not enabled
- 1000+ built-in checks

`soft_fail: true` — reports findings in the logs but exits with code 0 (success). This prevents existing infrastructure code with known findings from permanently blocking the pipeline. In a stricter setup, remove `soft_fail` to enforce zero critical findings.

---

### Stage 2: Terraform Plan

```yaml
      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
```
OIDC authentication — same pattern as the application pipelines. No static AWS keys.

```yaml
      - name: Terraform Init
        run: terraform init -input=false
        working-directory: ${{ env.WORKING_DIR }}
```
`terraform init` does three things:
1. Downloads providers (AWS, Kubernetes, Helm) into `.terraform/`
2. Configures the S3 backend (reads current state location)
3. Downloads module dependencies

`-input=false` prevents Terraform from interactively prompting for variables if any are missing — pipelines must be non-interactive.

```yaml
      - name: Terraform Validate
        run: terraform validate
```
Validates the syntax and internal consistency of all `.tf` files. Catches references to undefined variables, invalid resource arguments, and type mismatches. Runs without AWS credentials — purely static analysis.

```yaml
      - name: Terraform Plan
        run: terraform plan -no-color -input=false 2>&1 | tee plan_output.txt
```
`terraform plan` reads the current state from S3, queries live AWS APIs to get actual resource state, and calculates the diff. **Makes zero changes.**

`-no-color` — removes ANSI color codes (they render as garbage in GitHub log viewer).
`2>&1` — redirects stderr to stdout (Terraform writes some output to stderr).
`| tee plan_output.txt` — writes output to both the terminal (for live viewing) and a file (for the PR comment step).

```yaml
      - name: Post Plan as PR Comment
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const plan = fs.readFileSync('...plan_output.txt', 'utf8');
            await github.rest.issues.createComment({ body: `## Terraform Plan\n\`\`\`hcl\n${plan}\n\`\`\`` });
```
Posts the full Terraform plan output as a comment on the PR. Reviewers can see exactly what infrastructure changes the PR will make before approving. No need to check pipeline logs.

---

### Stage 3: Cost Estimation (Infracost)

```yaml
      - name: Setup Infracost
        if: ${{ secrets.INFRACOST_API_KEY != '' }}
        uses: infracost/actions/setup@v3
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}
```
`if: INFRACOST_API_KEY != ''` — gracefully skips cost estimation if the API key is not configured. The rest of the pipeline continues normally.

```yaml
      - name: Infracost Breakdown
        run: |
          infracost breakdown \
            --path=${{ env.WORKING_DIR }} \
            --format=json \
            --out-file=/tmp/infracost.json
```
Infracost reads your Terraform code and looks up the current AWS pricing for every resource. It outputs a JSON file with:
- Monthly cost of all resources in the plan
- Cost diff — how much more or less the planned changes will cost
- Per-resource cost breakdown

```yaml
      - name: Post Cost Comment on PR
        uses: infracost/actions/comment@v3
        with:
          path: /tmp/infracost.json
          behavior: update
```
Posts a formatted cost table as a PR comment. `behavior: update` — updates the existing comment on subsequent pushes instead of adding a new one (keeps PR comments clean).

---

### Stage 4: Apply (Bypassed)

```yaml
  apply:
    name: "4 · Terraform Apply"
    runs-on: ubuntu-latest
    needs: cost_estimate
    steps:
      - name: Apply Bypassed
        run: |
          echo "Terraform Apply — BYPASSED"
          echo "No infrastructure changes have been made."
          echo "Manual approval is required before applying."
```
This job runs and exits with code 0 (success) — it shows a green checkmark in the pipeline UI. No Terraform commands run. No AWS API calls are made. No infrastructure is changed.

The apply is separated into `terraform-apply.yml` which is `workflow_dispatch` only — it can never auto-trigger. When a human is ready to apply, they go to GitHub Actions UI → select the apply workflow → type `CONFIRM` → run.

---

### The Terraform Apply Workflow (Manual Only)

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [dev, prod]
      confirm:
        description: 'Type CONFIRM to proceed with apply'
        required: true
```
`workflow_dispatch` — only runnable by a human clicking "Run workflow" in GitHub Actions. Cannot be triggered by code push, PR, or any automated event.

The `confirm` input requires typing the word `CONFIRM`. The job only runs `if: github.event.inputs.confirm == 'CONFIRM'`. This two-step manual process prevents accidental applies.

---

## 11. Monitoring & Observability

### Three Pillars of Observability

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│    METRICS   │  │    LOGS      │  │    TRACES    │
│  Prometheus  │  │    Loki      │  │  (future)    │
│  Grafana     │  │  Grafana     │  │              │
└──────────────┘  └──────────────┘  └──────────────┘
```

### Prometheus + Grafana (Metrics)

**Prometheus** scrapes metrics from every pod via ServiceMonitor resources. Each service has a `/metrics` endpoint.

**What gets scraped:**
- HTTP request rates and latencies
- Node.js process metrics (heap, event loop lag)
- Custom business metrics (health check-ins per hour, medications taken)

**Grafana** visualizes these metrics as dashboards. Configured with 10Gi persistent storage so dashboards survive pod restarts.

**kube-prometheus-stack** also deploys:
- Node Exporter — OS-level metrics (CPU, memory, disk per node)
- kube-state-metrics — Kubernetes object state (pod restarts, deployment replica counts)
- AlertManager — routes alerts to email/Slack/PagerDuty

### Loki (Logs)

Loki collects all stdout/stderr from every pod in the cluster via a log shipping agent (Promtail, included in loki-stack).

**Why Loki over Elasticsearch?**
Loki indexes only log metadata (labels like pod name, namespace) not the log content. This makes it 10x cheaper to store than Elasticsearch while still being fully searchable. Grafana queries Loki using LogQL, the same UI used for Prometheus metrics.

### AWS Native Monitoring

| Service | What it monitors |
|---------|-----------------|
| CloudWatch | EKS container logs, CPU alarms, VPC flow logs |
| CloudTrail | All AWS API calls (who did what) |
| GuardDuty | Threat detection, anomalous behavior |
| Security Hub | Aggregated security findings |
| Inspector | Container image CVEs |
| AWS Config | Resource configuration compliance |
| AWS Budgets | Monthly cost thresholds |
| Cost Explorer | Cost anomaly detection |

---

## 12. Security Architecture

### Defense in Depth

```
Layer 1: WAF         — blocks known attack patterns before reaching ALB
Layer 2: ALB         — HTTPS only, SSL termination
Layer 3: VPC         — private subnets, security groups, NACLs
Layer 4: Kubernetes  — Network Policies, Pod Security Standards
Layer 5: Service     — JWT validation on every request
Layer 6: Database    — VPC-only access, KMS encryption at rest
Layer 7: Secrets     — AWS Secrets Manager, never in code or environment files
```

### OIDC Authentication for CI/CD

No static AWS access keys exist in GitHub Actions. Every pipeline authenticates via OpenID Connect:

```
GitHub Actions Runner
        ↓ request OIDC token
GitHub Identity Provider
        ↓ signed JWT
AWS STS AssumeRoleWithWebIdentity
        ↓ temporary credentials (1 hour TTL)
AWS API calls
```

The GitHub OIDC provider is registered in AWS IAM (via the `github-oidc` Terraform module). The IAM role's trust policy restricts which GitHub org/repo/branch can assume it.

### JWT Security

- Signed with HS256 using a randomly generated secret stored in AWS Secrets Manager
- 24-hour expiry
- Contains `linkedElders` claim to avoid inter-service relationship queries
- Never stored in cookies (localStorage only) — immune to CSRF attacks
- Validated by every service independently using the shared `JWT_SECRET`

---

## 13. Why These Tools?

### Why GitHub Actions for CI/CD?

- Native integration with GitHub repos — no separate CI server to maintain
- OIDC support for keyless AWS authentication
- Free for public repos, reasonable cost for private
- Massive ecosystem of pre-built actions

**Alternatives:** Jenkins (complex to operate), CircleCI (good but adds vendor), GitLab CI (requires GitLab), Tekton (Kubernetes-native but verbose)

### Why CI?

CI (Continuous Integration) ensures every code change is automatically tested, scanned, and built into a deployable artifact. Without CI, developers would discover broken code hours or days after writing it. With CI, broken code is caught within minutes of the push.

**CI pipeline stages and why each matters:**
- **Lint** — consistent code style, catches syntax errors before runtime
- **Tests** — validates business logic works as expected
- **SonarCloud** — catches code quality issues (complexity, duplication, bugs) that tests miss
- **Snyk** — catches vulnerable dependencies before they reach production
- **Docker build** — verifies the application can be containerized
- **Trivy** — catches CVEs in the container image itself
- **ECR push** — makes the verified image available for deployment

### Why CD?

CD (Continuous Deployment) automates the path from a merged code change to a running deployment. Without CD, a developer would manually update Kubernetes manifests, run kubectl apply, and hope nothing breaks. With CD, the entire deployment is deterministic and auditable via Git commits.

### Why ArgoCD?

ArgoCD implements GitOps — Git is the only way to change what runs in the cluster. This means:
- Every cluster change has a Git commit as evidence
- Rolling back means reverting a commit
- The cluster is self-healing — ArgoCD continuously reconciles any manual changes back to the Git state
- Multiple environments (dev, prod) managed from one gitops repo with environment-specific values

### Why Helm?

Helm provides templating for Kubernetes manifests. Without it, maintaining 120+ YAML files across 12 services would be unmanageable. A change to the standard health check configuration would require editing 12 files. With Helm, it's edited once in the template.

### Why Terraform?

Terraform allows infrastructure to be defined as code, version-controlled, reviewed in PRs, and reproduced identically. The alternative — clicking through the AWS console — creates infrastructure that:
- Cannot be recreated if accidentally deleted
- Cannot be reviewed by teammates
- Cannot be audited for changes
- Cannot be rolled back

With Terraform, the entire AWS infrastructure can be destroyed and recreated in under 30 minutes from the code in the repository.

### Why Separate CI and CD Pipelines?

**CI** runs on every push to every branch — even feature branches. It's fast and needs no production access.

**CD** runs only after CI succeeds on specific branches. It needs write access to the gitops repo. Separating them means a developer working on a feature branch can run CI (tests, build) without triggering a deployment to dev every time they push.

---

*Documentation generated for ElderPing Platform — June 2026*

# sync-service – CI/CD & Infrastructure Design Document

**Service:** `sync-service` (Spring Boot)  
**Cloud:** Google Cloud Platform (GCP)  
**Environments:** `qa` → `staging` → `prod`  
**Build Tool:** Jenkins  

---

## Part 1 – CI/CD Pipeline Design

---

### 1. Branching Strategy

| Branch | Purpose | Maps to Environment |
|---|---|---|
| `feature/*` | Developer feature work | — (CI only, no deploy) |
| `develop` | Integration branch | `qa` (auto-deploy) |
| `staging` | Pre-production validation | `staging` (auto-deploy) |
| `main` | Production-ready code | `prod` (manual gate) |
| `hotfix/*` | Critical production fixes | `prod` (via PR to `main`) |

#### Rules

- **All merges to `develop`, `staging`, and `main` require a Pull Request** — no direct pushes.
- **`main` additionally requires:**
  - Minimum 2 approving reviewers
  - All CI checks to pass on `staging` first
  - A mandatory Jenkins manual approval gate before any prod deployment
- Feature branches are created from `develop` and merged back via PR.
- Hotfixes branch from `main`, are PR-reviewed, and cherry-picked to `develop` after merge.

This flow guarantees that **code can never reach `prod` without first passing through `qa` and `staging`**, and without a human sign-off.

---

### 2. Jenkins Pipeline

#### 2.1 Pipeline Overview

```
┌────────┐  ┌────────┐  ┌───────────┐  ┌──────────┐  ┌────────────┐  ┌────────┐
│Checkout│→ │ Build  │→ │Unit Tests │→ │  Static  │→ │   Docker   │→ │ Deploy │
│        │  │ (Maven)│  │ + Coverage│  │ Analysis │  │Build & Push│  │        │
└────────┘  └────────┘  └───────────┘  └──────────┘  └────────────┘  └────────┘
```

#### 2.2 Stage Behaviour by Trigger

| Stage | PR (feature → develop) | Merge to `develop` | Merge to `staging` | Merge to `main` |
|---|---|---|---|---|
| Checkout | ✅ | ✅ | ✅ | ✅ |
| Build | ✅ | ✅ | ✅ | ✅ |
| Unit Tests + Coverage | ✅ | ✅ | ✅ | ✅ |
| Static Analysis (SonarQube) | ✅ | ✅ | ✅ | ✅ |
| Docker Build & Push | ❌ | ✅ | ✅ | ✅ |
| Deploy to QA | ❌ | ✅ (auto) | ❌ | ❌ |
| Integration Tests | ❌ | ✅ | ❌ | ❌ |
| Deploy to Staging | ❌ | ❌ | ✅ (auto) | ❌ |
| Smoke Tests | ❌ | ❌ | ✅ | ❌ |
| Manual Approval Gate | ❌ | ❌ | ❌ | ✅ |
| Deploy to Prod | ❌ | ❌ | ❌ | ✅ (post-approval) |

#### 2.3 Rollback Strategy

**Automated rollback** triggers if the post-deployment health check fails within 5 minutes:

1. Jenkins re-deploys the **previous Docker image tag** (stored as `LAST_STABLE_TAG` in Jenkins credentials store).
2. Sends an alert to Slack/email with the failed build link.
3. Marks the build as `UNSTABLE` and blocks further deployments until fixed.

**Manual rollback** command:
```bash
./scripts/rollback.sh --env prod --tag <previous-image-tag>
```

For `prod`, blue/green swap-back is instant (see Section 4).

---

### 3. Configuration Management

#### 3.1 Environment-Specific Config

Spring Boot profiles (`application-qa.yml`, `application-staging.yml`, `application-prod.yml`) are stored in a **separate private Git repository** (`sync-service-config`), not in the application repo. Jenkins pulls the appropriate config at deploy time via:

```bash
git clone git@github.com:org/sync-service-config.git --branch ${ENV}
```

This gives:
- Clean separation of code and config
- Auditable config history
- No risk of leaking config in the app repo

#### 3.2 Secrets Handling

| Secret Type | Storage | Injection Method |
|---|---|---|
| MongoDB credentials | **GCP Secret Manager** | Injected as env vars at startup via Workload Identity |
| API keys | **GCP Secret Manager** | Same as above |
| Jenkins credentials | **Jenkins Credentials Store** | Bound in `withCredentials {}` block |
| Docker registry token | **Jenkins Credentials Store** | Bound in pipeline |

Secrets are **never** stored in `application.yml`, passed as plain CLI args, printed in logs, or committed to Git.

In Spring Boot, secrets are accessed via:
```yaml
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI}  # injected from Secret Manager at runtime
```

The GCP VM's Service Account has `roles/secretmanager.secretAccessor` scoped only to required secrets (least privilege).

---

### 4. Deployment Strategy

**Chosen Strategy: Blue/Green for `prod`, Rolling for `qa`/`staging`**

#### Why Blue/Green for Production?

| Criteria | Blue/Green | Rolling | Recreate |
|---|---|---|---|
| Downtime | Zero | Near-zero | Yes |
| Rollback speed | Instant (DNS/LB flip) | Slow (re-deploy) | Slow |
| Resource cost | 2× during deploy | 1× | 1× |
| Risk | Low | Medium | High |

For a startup, cost matters — but **prod downtime costs more than the brief 2× VM cost during a deployment window**. Blue/Green is justified for `prod`.

`qa` and `staging` use **Rolling** (cheaper, acceptable brief disruption).

#### Blue/Green Deployment Flow (prod)

1. The new version is deployed to the **Green** VM group (identical spec to Blue).
2. Health checks are run against Green.
3. GCP Load Balancer traffic is shifted from Blue → Green.
4. Blue group is kept warm for 15 minutes (instant rollback window).
5. Blue is terminated after the window passes.

```
LB → [Blue: v1.2]           (before)
LB → [Green: v1.3]          (after flip)
LB → [Blue: v1.2] (standby, 15min window)
```

---

## Part 2 – Infrastructure Design

---

### 5. Compute Choice

**Chosen: GCP Compute Engine (VMs) with Managed Instance Groups (MIGs)**

The assignment states the service is **already deployed to GCP VMs**. Migrating to GKE or Cloud Run adds unnecessary complexity and cost for a startup. The right move is to operationalize the existing VM setup properly.

| Option | Verdict | Reason |
|---|---|---|
| **Compute Engine + MIG** | ✅ Chosen | Fits existing setup, simple auto-scaling, lowest overhead |
| GKE | ❌ Overkill now | Steep learning curve, higher cost, better when many services exist |
| Cloud Run | ❌ Mismatch | Designed for stateless HTTP; sync-service has MongoDB connections and may run long jobs |

MIGs provide:
- **Auto-scaling** based on CPU utilization (target 60%)
- **Auto-healing** — unhealthy instances are replaced automatically
- **Rolling updates** baked in

---

### 6. MongoDB Hosting

**Chosen: MongoDB Atlas (GCP region co-location)**

| Option | Verdict |
|---|---|
| **MongoDB Atlas** | ✅ Chosen |
| Self-hosted on GCE | ❌ Too much ops burden for a startup |
| Bitnami on GKE | ❌ Complex |

Atlas in the **same GCP region** (e.g., `us-central1`) gives:
- Low latency via VPC Peering (traffic never hits public internet)
- Automated backups, point-in-time restore
- Built-in HA with replica sets
- Zero operational burden — no patching, no replica management

Connection string stored in **GCP Secret Manager**, accessed at runtime.

---

### 7. Networking (VPC & Ingress)

```
Internet → Cloud Armor (WAF/DDoS) → HTTPS Load Balancer → sync-service VMs (private subnet)
                                                               ↕ VPC Peering
                                                          MongoDB Atlas (private endpoint)
```

- All VMs are in a **private subnet** — no public IPs.
- Only the HTTPS Load Balancer has a public IP.
- **Cloud Armor** sits in front for DDoS protection and IP allowlisting.
- **Firewall rules** allow only LB → VMs on port 8080 (health checks on 8081).
- VMs talk to MongoDB Atlas over **VPC Peering** (private, not public internet).
- Jenkins is in a separate management subnet, reachable only via VPN/IAP tunnel.

---

### 8. Secrets & IAM

**Principle of Least Privilege across all components:**

| Component | GCP Role(s) |
|---|---|
| sync-service VM Service Account | `roles/secretmanager.secretAccessor` (scoped secrets only) |
| Jenkins Service Account | `roles/compute.instanceAdmin.v1`, `roles/storage.objectAdmin` (Artifact Registry) |
| Developers | `roles/viewer` on GCP console; no prod SSH access |
| SRE/Ops | `roles/compute.osLogin` for break-glass prod access (logged) |

Workload Identity binds the Service Account to VMs — no key files, no manual rotation.

---

### 9. Logging & Monitoring Stack

| Concern | Tool | Details |
|---|---|---|
| Application logs | **GCP Cloud Logging** | Spring Boot ships JSON logs; GCP agent ships to Cloud Logging |
| Metrics | **Cloud Monitoring** | CPU, memory, request latency, error rate |
| Alerting | **Cloud Monitoring Alerts** | PagerDuty/Slack integration; alert on error rate > 1% or latency p99 > 500ms |
| Uptime checks | **Cloud Monitoring Uptime** | `/actuator/health` endpoint checked every 60s |
| Distributed tracing | **Cloud Trace** | Spring Boot Micrometer → Cloud Trace for request tracing |
| Dashboard | **Cloud Monitoring Dashboard** | CPU, GC pause, MongoDB connection pool, request rate |

Spring Boot Actuator is enabled with endpoints: `health`, `info`, `prometheus` (for Prometheus scraping if needed later).

---

## Summary of Key Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Branching | Gitflow-lite (develop/staging/main) | Clear env mapping, prevents accidental prod deploys |
| Prod deployment | Blue/Green | Zero downtime, instant rollback |
| Config management | Separate config repo + GCP Secret Manager | Separation of concerns, no secrets in code |
| Compute | GCE + MIGs | Continuity with existing setup, cost-effective |
| Database | MongoDB Atlas (GCP co-located) | Managed HA, zero ops burden, VPC peering |
| Secrets | GCP Secret Manager + Workload Identity | No key files, automatic rotation support |
| Monitoring | GCP-native stack | Unified, no extra services to manage |

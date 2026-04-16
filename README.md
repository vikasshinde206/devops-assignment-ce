# 🚀 CE DevOps Assignment – Infrastructure Design

## 📌 Overview

This project demonstrates a production-ready infrastructure design on **Google Cloud Platform (GCP)** for deploying a Spring Boot service (`sync-service`).

The architecture is designed with a focus on:
- Auto-scaling
- Secure access
- Cost optimization (startup-friendly)

---

## 🏗️ Architecture Diagram

Architecture dia

---

## ⚙️ Architecture Summary

The system is designed using **Compute Engine Managed Instance Groups (MIG)** behind a **GCP HTTP(S) Load Balancer**.

### 🔁 High-Level Flow
User → Load Balancer → Managed Instance Group → Application → MongoDB Atlas


---

## 🧠 Key Design Decisions

### 1️⃣ Compute Choice – Compute Engine (MIG)

**Selected:** Compute Engine + Managed Instance Group

#### Why:
- Supports auto-scaling and auto-healing
- Lower cost compared to GKE
- Simpler to operate (no Kubernetes overhead)
- Aligns with existing VM-based deployment

#### Why not GKE:
- Higher operational complexity
- Overkill for a single service

#### Why not Cloud Run:
- Less runtime control
- Potential cold start latency

---

### 2️⃣ MongoDB Hosting – MongoDB Atlas

**Selected:** MongoDB Atlas (Managed Service)

#### Why:
- No database maintenance required
- Built-in backups and high availability
- Easy scaling
- Reduces operational overhead

#### Security:
- Access restricted via IP/VPC
- Credentials stored in Secret Manager

---

### 3️⃣ Networking Design (VPC)

#### Structure:
- Single VPC
- Separate subnets (QA / Staging / Production)

#### Key Decisions:
- VMs do not have public IPs
- Only Load Balancer is exposed publicly
- Internal communication via private network

#### Traffic Flow:
User → HTTPS Load Balancer → Private VM → MongoDB


#### Security:
- Firewall rules:
  - Allow 80/443 → Load Balancer
  - Allow LB → App (port 8080)
  - Deny all other traffic

---

### 4️⃣ Secrets & IAM

#### Secrets Management:
- GCP Secret Manager used to store:
  - MongoDB credentials
  - API keys
  - Tokens

#### IAM Strategy:
- Service Accounts used for access control
- Least privilege principle applied:
  - App → read-only secrets access
  - Jenkins → deployment permissions only

---

### 5️⃣ Logging & Monitoring

#### Tools Used:
- Cloud Logging
- Cloud Monitoring

#### Monitored Metrics:
- CPU / Memory usage
- Application health
- Error rates (5xx)
- Latency

#### Alerts:
- High error rate
- Instance failures
- High resource usage

---

### 6️⃣ Auto-Scaling Strategy

- Implemented using Managed Instance Group (MIG)
- Scaling based on:
  - CPU utilization
  - Traffic load

#### Benefits:
- Handles traffic spikes automatically
- Optimizes cost during low usage

---

### 7️⃣ Deployment Strategy

#### QA / Staging:
- Rolling deployments

#### Production:
- Blue/Green deployment using two instance groups:
  - `prod-blue`
  - `prod-green`

#### Flow:
1. Deploy to Green environment
2. Run health checks
3. Switch traffic via Load Balancer
4. Rollback instantly if needed

---

## 🔐 Security Highlights

- No public IPs on application VMs
- Secrets managed via Secret Manager
- IAM with least privilege access
- Firewall rules restrict all unnecessary traffic

---

## 💰 Cost Optimization

- Avoided GKE to reduce operational cost
- Used Managed Instance Group instead of always-on infrastructure
- MongoDB Atlas reduces DB maintenance overhead
- Auto-scaling ensures pay-per-use efficiency

---

## 📦 CI/CD Integration

- Jenkins used for build and deployment
- Docker images pushed to GCR
- Automated deployments to QA, Staging, and Production

---

## ✅ Conclusion

This architecture provides:
- Scalable infrastructure using MIG
- Secure access with VPC and IAM
- Cost-efficient design suitable for startups
- Production-ready deployment strategy with zero downtime

---


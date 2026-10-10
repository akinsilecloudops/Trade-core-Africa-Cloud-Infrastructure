# Cost Optimization & Financial Model — TradeCore Africa Platform

**System Architect & Author:** Abdul-Hakeem Akinsile  
**Target Workload:** 340 Active Corporate Clients | > ₦180,000,000 Daily Transaction Volume  
**Deployment Footprint:** Primary in `af-south-1` (Cape Town) | Cold Standby in `eu-west-1` (Ireland)  
**Target SLA & DR Metrics:** 99.95% Availability (< 4.38 hours downtime/year) | RTO $\le$ 15 mins | RPO $\le$ 5 mins  

---

## 1. Executive Summary

TradeCore Africa transitioned from an unmanaged, single-instance architecture in `us-east-1` to a resilient, multi-AZ cloud architecture in `af-south-1` to fulfill Tier-1 Nigerian bank partner integration mandates and regional data sovereignty standards.

This financial model outlines the infrastructure expenditure baseline required to sustain a 99.95% uptime SLA, automated failover capabilities, and end-to-end cryptographic isolation using customer-managed KMS keys.

* **Estimated Monthly Run-Rate (Baseline On-Demand):** **$1,433.06 / month**
* **Estimated Annual Run-Rate (Baseline On-Demand):** **$17,197.11 / year**
* **Optimized Production Run-Rate (Savings Plans + Graviton):** **~$720.00 / month**

---

## 2. Workload & Sizing Assumptions

* **Transaction Scale:** 340 corporate clients executing an average of ₦180M daily volume (~15–35 steady-state RPS, burstable to 350 RPS during 10× month-end invoice financing surges).
* **Compute Architecture:** 5 decoupled microservices (Auth, Core Ledger, Gateway/Invoicing, Reports, Webhook/Notification Dispatcher) running a baseline of 2 tasks per service across 2 Availability Zones (10 tasks minimum).
* **Data Sovereignty:** Customer PII, ledger balances, and audit logs reside in `af-south-1` encrypted with dedicated KMS CMKs.
* **Disaster Recovery (RTO $\le$ 15 min / RPO $\le$ 5 min):** Continuous asynchronous storage replication to `eu-west-1` via Aurora PostgreSQL read replica and S3 replication, maintaining compute resources in a cold standby (zero-task baseline) to minimize ongoing spend.

---

## 3. Tiered Cost Breakdown (Baseline On-Demand in `af-south-1`)

| Architectural Tier | AWS Services Included | Provisioned Specifications | Monthly Cost (USD) |
| :--- | :--- | :--- | :--- |
| **Ingress & Networking** | Elastic Load Balancing | 1 Regional ALB deployed across 2 AZs | $91.25 |
| **Compute & Microservices** | AWS Fargate (ECS) | 10 Tasks (24/7 runtime, 1 vCPU / 2GB RAM baseline) | $486.18 |
| **Relational Database** | Amazon Aurora PostgreSQL | 2 Nodes (`db.t4g.medium` Multi-AZ Writer + Replica), 100GB storage, RDS Proxy, automated backups | $541.02 |
| **In-Memory Caching** | Amazon ElastiCache (Redis) | 2 Nodes (`cache.t4g.small` Multi-AZ cluster), 5KB avg payload | $181.78 |
| **VPC & Egress Connectivity** | Regional NAT Gateway | 1 Regional NAT Gateway shared across 2 AZs + data processing | $83.79 |
| **Object Storage & Registry** | Amazon S3 & Amazon ECR | 200GB S3 Standard, 300GB S3 Intelligent-Tiering, 50GB ECR Container Images | $12.99 |
| **Edge & DNS Routing** | AWS WAF & Route 53 | 1 Regional Web ACL (6 core managed rules), 1 Hosted Zone + Health Checks | $13.50 |
| **Security, Audit & KMS** | AWS KMS, Config, CloudTrail | 1 Customer-Managed Key (500K symmetric requests), Config Conformance Pack, CloudTrail Insights | $22.55 |
| **Total Baseline Run-Rate** | | **Standard Steady-State Workload** | **$1,433.06 / month** |

---

## 4. Cost Scenario Comparison & Dynamic Peak Handling

| Cost Scenario | Description | Monthly Total (USD) |
| :--- | :--- | :--- |
| **Baseline Normal Load** | Steady-state on-demand infrastructure in `af-south-1` | $1,433.06 |
| **10× Month-End Peak Surge** | +30 Fargate Tasks (40 total) + Aurora ACU scaling (72-hour window) | $1,980.50 |
| **Optimized Production Target** | Graviton ARM64 migration + 1-Year Compute Savings Plans | ~$720.00 |

### Cost Profile Under 10× End-of-Month Peak
During invoice processing surges, Target Tracking Auto-Scaling scales Fargate capacity from 10 tasks to 40 tasks while Aurora dynamically manages replica load. Because Fargate and Aurora Serverless bill by fractional resource runtimes, a 72-hour month-end spike adds only **~$547.44** in burst compute and database I/O costs, preventing permanent over-provisioning expenditure during the remaining 27 days of the month.

---

## 5. Architectural Trade-Off Analysis & Value Justification

* **Multi-AZ Aurora ($541.02/mo) vs. Standalone RDS PostgreSQL ($180.00/mo):** Multi-AZ Aurora provides automated failover in < 30 seconds and distributed 6-way storage replication, directly fulfilling the 99.95% uptime SLA and eliminating manual database restore downtime.
* **ECS Fargate ($486.18/mo) vs. Self-Managed EC2 Clusters ($280.00/mo):** Fargate eliminates OS patching, kernel upgrades, and idle hypervisor reservations. The operational engineering hours saved exceed the minor compute cost differential.
* **ElastiCache Cluster ($181.78/mo) vs. Direct DB Queries:** Caching authentication tokens, rate limits, and client session records reduces database CPU utilization by > 65%, keeping Aurora instance sizing at `db.t4g.medium` rather than requiring `db.r6g.xlarge` instances ($850+/mo).
* **Single Shared NAT Gateway ($83.79/mo) vs. Dual NAT Gateways ($167.58/mo):** Routing outbound egress through a single regional NAT Gateway cuts VPC gateway hourly charges in half while providing the required outbound internet access for updates and external APIs.
* **KMS CMKs + Config ($22.55/mo):** Essential for double-entry ledger compliance, audit log immutability, and Tier-1 banking security certification.

---

## 6. Long-Term Cost Optimization Strategies

### 1. Compute & Architecture Efficiencies
* **Migrate ECS Fargate to ARM64 (AWS Graviton):** Recompiling Docker images for ARM64 delivers an immediate 20% cost reduction on compute alongside improved performance per vCPU.
* **Adopt 1-Year Compute Savings Plans:** Committing to steady-state baseline Fargate compute usage lowers task pricing by up to 20%.

### 2. Object Storage Tiering & Lifecycle Policies (Amazon S3)
* **S3 Intelligent-Tiering (Objects $\ge$ 128 KB):** Applied to variable-access documents (PDF invoice uploads, bulk statement exports, and raw CloudTrail/audit logs). S3 automatically transitions data across Frequent, Infrequent, and Archive Instant Access tiers based on real access patterns, delivering 40–70% storage savings without retrieval fees.
* **S3 Standard (Objects < 128 KB):** Applied to transactional metadata, JSON signatures, and thumbnail assets. Objects under 128 KB are retained in S3 Standard to avoid lifecycle transition overhead and maintain predictable baseline costs.

---

## 7. Verification & AWS Pricing Calculator Link

The formal AWS Cost Calculator estimate for this multi-region deployment can be verified below:

## 7. Verification & AWS Pricing Calculator Link

The formal AWS Cost Calculator estimate for this multi-region deployment can be reviewed through either of the following references:

* **Committed Architectural Document:** [📄 TradeCore Africa Cost Estimate (PDF)](./Trade-core-Africa-Cloud-Infrastructure/TradeCore_Africa_Cost_Estimate.pdf)

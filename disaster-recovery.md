# Disaster Recovery & Business Continuity Plan — TradeCore Africa Platform

**System Architect & Author:** Abdul-Hakeem Akinsile  
**Target Recovery Objectives:** Recovery Time Objective (RTO) $\le$ 15 Minutes | Recovery Point Objective (RPO) $\le$ 5 Minutes  
**Primary Region:** `af-south-1` (Cape Town, South Africa)  
**Disaster Recovery Region:** `eu-west-1` (Ireland)  

---

## 1. Executive Summary & Strategy Overview

TradeCore Africa employs an **Active-Passive / Cold-Compute Standby** disaster recovery model across two AWS regions. The primary production environment operates out of `af-south-1` (Cape Town), while `eu-west-1` (Ireland) serves as the standby site to ensure business continuity if the entire primary region experiences a catastrophic outage.


```

+-----------------------------------------------------------------------------------------+
|                                    DISASTER RECOVERY OVERVIEW                           |
+-----------------------------------------------------------------------------------------+
[ Incoming Users ]
│
▼
[ Amazon Route 53 ]
│
▼
[ AWS Global Accelerator ]
(Dedicated AWS Fiber Backbone Routing)
│
┌─────────────────────────┴─────────────────────────┐
│                                                   │
▼ (Primary Route: Normal Ops)                       ▼ (Failover Route: Outage)
+---------------------------------------+           +---------------------------------------+
| PRIMARY REGION: af-south-1            |           | DR REGION: eu-west-1                  |
|                                       |           |                                       |
| Ingress: ALB (AZ-1 + AZ-2)            |           | Ingress: Standby ALB (AZ-1)           |
| Compute: ECS Fargate (Active Tasks)   |           | Compute: ECS Fargate (Desired = 0)    |
| DB: Aurora PostgreSQL (Writer)        |           | DB: Aurora Read Replica (Standby)     |
+---------------------------------------+           +---------------------------------------+
│                                                   ▲
│                                                   │
└────── Continuous Live Storage Replication ────────┘

```

### Core Strategy Principles
1. **Zero-Baseline Compute Standby:** To satisfy Nigerian data residency policies and minimize idle infrastructure costs, compute resources in `eu-west-1` run with a desired count of 0 tasks during normal operations. No customer transactions are actively processed in Ireland while `af-south-1` is healthy.
2. **Continuous Storage Replication:** Relational data is continuously replicated from the primary cluster in `af-south-1` to an Aurora PostgreSQL Read Replica in `eu-west-1`.
3. **Automated Backbone Ingress Switching:** Traffic rerouting is handled at the network edge via AWS Global Accelerator, eliminating the multi-minute propagation delays associated with traditional DNS TTLs.

---

## 2. Key Recovery Metrics

| Recovery Metric | Target SLA | Architecture Implementation | Validation Status |
| :--- | :--- | :--- | :--- |
| **Recovery Time Objective (RTO)** | $\le$ 15 Minutes | AWS Global Accelerator health-check detection + automated ECS Fargate scale-out (60–90s task startup) | Achievable within 5–10 minutes |
| **Recovery Point Objective (RPO)** | $\le$ 5 Minutes | Continuous asynchronous Aurora storage-layer replication to `eu-west-1` | Achievable (typically < 1 second replication lag) |
| **Failover Trigger** | Automated / Health-Checked | Edge health monitoring detecting unrecoverable primary regional failure | Zero manual code redeployment required |

---

## 3. Disaster Recovery Architecture Tiers

### 3.1 Edge & Traffic Rerouting Tier
* **Amazon Route 53:** Serves as the authoritative global DNS resolver.
* **AWS Global Accelerator:**
  * Provisions dual static Anycast IP addresses routed across AWS’s global private fiber backbone.
  * Bypasses the public internet to reduce latency for African and international traffic.
  * Continuously evaluates health-check endpoints exposed by the Application Load Balancer in `af-south-1`.
  * If health checks against `af-south-1` fail, Global Accelerator automatically re-points traffic to the regional ALB in `eu-west-1` without waiting for DNS propagation.

### 3.2 Compute & Scaling Tier (ECS on AWS Fargate)
* **Normal Operating State (`af-south-1`):** ECS Fargate services run across multiple Availability Zones with target-tracking scaling enabled.
* **Standby State (`eu-west-1`):** 
  * The ECS cluster and task definitions are pre-provisioned via Terraform.
  * Desired task count and minimum task count are explicitly set to **0**.
  * The environment incurs zero ongoing compute or CPU/RAM charges.
* **Failover Activation:**
  * Inbound requests arriving at the `eu-west-1` ALB trigger the auto-scaling policy, scaling compute tasks from 0 to active capacity.
  * Fargate container tasks spin up and reach healthy status in approximately 60 to 90 seconds, bypassing the multi-minute operating system boot sequence typical of traditional EC2 virtual machines.

### 3.3 Database & Persistence Tier
* **Primary Region:** The Aurora PostgreSQL primary writer instance operates in `af-south-1` `AZ-1`, replicating locally to an intra-region read replica in `AZ-2`.
* **Standby Region:** An Aurora PostgreSQL Read Replica resides in `eu-west-1` `AZ-1`, continuously receiving asynchronous updates from the primary cluster.
* **RDS Proxy Layer:** Both regions maintain an Amazon RDS Proxy instance within their private subnets to pool, multiplex, and manage client connections, preventing database connection exhaustion when compute tasks scale up rapidly.

---

## 4. End-to-End Failover Workflow


```

[ Step 1: Regional Outage in af-south-1 ]
│
▼
[ Step 2: Global Accelerator Health Checks Fail ]
│
▼
[ Step 3: Ingress Automatically Shifts to eu-west-1 ALB ]
│
▼
[ Step 4: Aurora Read Replica Promoted to Primary Writer in eu-west-1 ]
│
▼
[ Step 5: Fargate Compute Auto-Scales Out (0 ➔ N Active Tasks) ]
│
▼
[ Step 6: Platform Live & Operational in DR Region (RTO < 10 mins) ]

```

### Detailed Sequence of Events

1. **Failure Detection (0 – 30 seconds):**
   * AWS Global Accelerator detects persistent health-check failures from the primary regional Application Load Balancer in `af-south-1`.
2. **Traffic Diversion (30 – 60 seconds):**
   * Global Accelerator shifts incoming Anycast traffic directly to the `eu-west-1` Application Load Balancer.
3. **Database Promotion (1 – 3 minutes):**
   * The standby Aurora PostgreSQL read replica in `eu-west-1` is promoted to independent read-write primary status.
   * The regional Amazon RDS Proxy updates its internal target group endpoint to route write transactions directly to the newly promoted cluster.
4. **Compute Activation (3 – 5 minutes):**
   * As the `eu-west-1` ALB receives inbound traffic, target-tracking auto-scaling metrics trigger Fargate task creation.
   * ECS tasks pull Docker images from Amazon ECR and achieve operational running status in 60–90 seconds.
5. **Full Restoration & Validation (5 – 10 minutes):**
   * Health checks confirm that backend APIs are serving requests, ElastiCache Redis is warming up session data, and database read/write queries are executing successfully.

---

## 5. Failback Strategy (Returning to `af-south-1`)

Once AWS resolves the underlying infrastructure issue in `af-south-1`, traffic is returned to the primary region through a controlled migration:

1. **Re-establish Storage Replication:**
   * Configure the recovered Aurora PostgreSQL cluster in `af-south-1` as a read replica of the active `eu-west-1` database to replicate all transactions executed during the disaster period.
2. **Verify Replication Synchronization:**
   * Monitor replication lag metrics until `af-south-1` is fully synchronized (lag < 1 second).
3. **Promote `af-south-1` to Primary Writer:**
   * Schedule a brief maintenance window to freeze writes in `eu-west-1` and promote the `af-south-1` cluster back to the active primary writer.
4. **Switch Edge Routing:**
   * Update AWS Global Accelerator endpoint weights to redirect client traffic back to the primary ALB in `af-south-1`.
5. **Scale Standby Compute to Zero:**
   * Reset the desired and minimum capacity of the `eu-west-1` ECS Fargate service back to **0**, re-establishing the zero-compute standby state.

---

## 6. Backup & Point-in-Time Recovery (PITR)

* **Continuous Automated Backups:** Amazon Aurora retains continuous transaction logs alongside daily storage volume snapshots with a **35-day retention period**.
* **Point-in-Time Recovery (PITR):** Enables transactional rollback to any specific second within the retention window, protecting against accidental human data corruption or logical application errors.
* **Encrypted Storage Volumes:** All database snapshots, backups, and cross-region replication streams are encrypted at rest using dedicated AWS KMS customer-managed keys.


# Scalability & Traffic Handling Strategy — TradeCore Africa Platform

**System Architect & Author:** Abdul-Hakeem Akinsile  
**Baseline Workload:** ~15–35 Requests Per Second (RPS) (~100 concurrent users)  
**Peak Workload Target:** Up to 10× Baseline (~350 RPS / 1,000 concurrent users)  
**Response Performance Objective:** P99 Latency < 2 seconds | Error Rate < 0.1%  

---

## 1. Scalability Architecture Overview

TradeCore Africa experiences predictable end-of-month surges driven by corporate billing, invoice financing approvals, and bulk ledger reconciliations. To accommodate a **10× surge** without degrading service availability or over-provisioning baseline capacity, scalability is enforced horizontally across compute, caching, and database connection pooling tiers.
--

## 2. Compute Tier Auto-Scaling (Amazon ECS on AWS Fargate)

Application workloads run as containerized microservices on AWS Fargate across `af-south-1a` and `af-south-1b`.

### Scaling Policies & Parameters
* **Launch Type:** AWS Fargate serverless containers. Fargate tasks reach active, healthy status within 60 to 90 seconds, bypassing the multi-minute virtual machine boot time required by traditional EC2 instances.
* **Baseline Footprint:** 10 tasks steady-state (distributed across 5 decoupled services: Auth, Core Ledger, Gateway/Invoicing, Reports, Webhook Dispatcher).
* **Scaling Ceiling:** Dynamically scales up to a maximum of **50 tasks**.
* **Target Tracking Metric 1 (CPU Utilization):** Maintained at **60% average CPU utilization**.
* **Target Tracking Metric 2 (Traffic Volume):** Maintained at **1,000 requests per target** on the Application Load Balancer.
* **Scale-Out Cooldown:** 30 seconds to react to rapid invoice submission surges.
* **Scale-In Policy:** Gradual scale-down during off-peak periods to prevent thrashing and eliminate unnecessary runtime costs.

---

## 3. Caching & In-Memory Read Offloading (Amazon ElastiCache Redis)

Database IOPS are reserved for write-heavy ledger settlements by offloading repetitive read traffic to an in-memory caching tier.

* **Sub-Millisecond Query Response:** ElastiCache Redis clusters sit adjacent to the compute layer in the private subnet.
* **Read-Path Absorption:** Caches JWT authorization tokens, client profile data, exchange rates, and static system configurations.
* **Database Offloading Target:** Absorbs roughly **80% of repetitive read operations**, preventing database CPU spikes during peak periods.
* **Multi-AZ Availability:** Redis operates with automated failover across availability zones, ensuring cache continuity if a single zone degrades.

---

## 4. Database Connection Scaling & Multiplexing (Amazon RDS Proxy)

Sudden horizontal compute scaling from 10 to 50 Fargate tasks risks creating hundreds of concurrent database connections, which can exhaust database memory and CPU.

* **Connection Pooling:** Amazon RDS Proxy sits in the private subnet between ECS Fargate tasks and the Aurora PostgreSQL database engine.
* **Connection Multiplexing:** Consolidates hundreds of ephemeral client connections into a manageable pool of persistent database connections.
* **Resource Protection:** Prevents Aurora from spending CPU cycles opening, authenticating, and closing connection threads during traffic bursts, maintaining stable transaction processing.
* **Failover Buffer:** Holds incoming application queries during database failovers, minimizing transient application errors.

---

## 5. Persistent Storage Scaling (Amazon Aurora PostgreSQL)

* **Storage Auto-Scaling:** Amazon Aurora automatically grows database storage in 10 GB increments up to 128 TB without requiring manual volume resizing or downtime.
* **Read Replica Offloading:** Transaction writes route to the primary writer in `AZ-1`, while read-heavy operations (invoice search, historical statement queries) can be directed to the Aurora Read Replica in `AZ-2`.
* **Multi-Region Cross-Synchronization:** Transaction writes in `af-south-1` are continuously replicated to the `eu-west-1` read replica asynchronously at the storage layer, maintaining sub-second replication lag even under high write load.

---

## 6. Static Asset Decoupling (Amazon S3)

To prevent web asset downloads from consuming ECS compute bandwidth or application memory, frontend assets are isolated:

* Web UI bundles, PDF invoice downloads, and media assets are served directly from **Amazon S3** static storage.
* Compute tasks process dynamic business API requests, while static traffic scales independently on AWS object storage.

---

## 7. Scalability Verification & Load Testing Standards

The architecture is benchmarked against end-of-month volume using automated stress testing:

1. **Test Tool:** Distributed load testing via Grafana K6.
2. **Execution Profile:** Ramping from baseline (35 RPS) to 10× peak load (350 RPS) over a 10-minute window, sustaining peak throughput for 20 continuous minutes.
3. **Acceptance Thresholds:**
   * **P99 Response Latency:** Must remain below 2.0 seconds.
   * **Application Error Rate:** Must not exceed 0.1% across all HTTP requests.
   * **Scale-Out Latency:** Additional ECS Fargate tasks must initialize and enter target group rotation within 3 minutes of threshold detection.
   * **Database Connection Saturation:** RDS Proxy database connection utilization must remain below 80% of capacity.

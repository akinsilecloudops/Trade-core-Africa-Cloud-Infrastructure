# Architecture Overview — TradeCore Africa Cloud Platform

**Architect:** Abdul-Hakeem Akinsile  

---

## 1. System Topology & Regional Distribution

TradeCore Africa is deployed across a multi-region footprint on AWS to balance regional data residency constraints with automated disaster recovery:

* **Primary Production Region (`af-south-1` — Cape Town):** Houses the primary transactional workloads across two Availability Zones (`AZ-1` and `AZ-2`). Cape Town was selected because AWS does not operate a physical data center in Nigeria, making `af-south-1` the closest geographical region to Nigerian businesses.
* **Disaster Recovery Region (`eu-west-1` — Ireland):** Deployed across a single Availability Zone (`AZ-1`) as a cold-compute standby to meet the business RTO and RPO targets.

---

## 2. Global Traffic Ingress & Perimeter Security

* **DNS Resolution:** Client requests resolve through **Amazon Route 53**.
* **Edge Routing & Acceleration:** Ingress traffic enters **AWS Global Accelerator**, providing static Anycast IP addresses routed over AWS’s global private fiber backbone to minimize public internet latency. Global Accelerator actively runs health checks against the primary region (`af-south-1`). If `af-south-1` becomes unavailable, it automatically shifts incoming traffic to `eu-west-1`.
* **Application Firewall:** Traffic passes through **AWS WAF (Web Application Firewall)** to inspect Layer 7 traffic against OWASP exploits, SQL injection, and volumetric abuse before reaching compute boundaries.
* **Network Edge:** Clean traffic crosses the VPC **Internet Gateway** to reach the external **Application Load Balancers (ALBs)**.

---

## 3. Subnet Layout & Component Breakdown

Both regions implement a segregated Virtual Private Cloud (VPC) topology consisting of public and private subnets:

### Tier 1: Public Subnet
* **Application Load Balancer (ALB):** Terminates client TLS connections and distributes traffic evenly across compute tasks in both `AZ-1` and `AZ-2`. If one availability zone fails, the ALB automatically isolates traffic to the healthy zone.
* **Shared Regional NAT Gateway:** A single NAT Gateway is deployed to serve both availability zones, providing outbound internet connectivity for private subnet tasks (e.g., container updates and payment gateway API calls) while avoiding redundant NAT Gateway hourly costs.

### Tier 2: Private Subnet (Compute, Caching, Storage & Database)
* **Serverless Compute (Amazon ECS on AWS Fargate):**
  * Microservices and API workloads run as containerized tasks managed by ECS Fargate.
  * Fargate was chosen over EC2 because it spins up new tasks in 60 to 90 seconds, avoiding the 2+ minute boot times of traditional virtual machines.
  * Configured with target-tracking auto-scaling to scale tasks dynamically up or down based on workload demand.
* **Container Registry (Amazon ECR):** Stores versioned, immutable Docker images pulled by ECS tasks across both availability zones.
* **Static Assets (Amazon S3):** Dedicated S3 buckets store static website bundles and application frontend assets.
* **In-Memory Caching (Amazon ElastiCache Redis):**
  * Sits directly alongside the compute layer to cache frequent read paths and manage session states.
  * Reduces overall read volume and latency before queries touch persistent storage.
* **Database Connection Pooling (Amazon RDS Proxy):**
  * Sits directly in the private subnet between the compute tasks (and cache) and the database engine.
  * Pools, multiplexes, and consolidates database connections, preventing sudden horizontal task scale-outs from overwhelming database connection limits and spiking compute costs.
* **Transactional Persistence (Amazon Aurora PostgreSQL):**
  * Deployed securely within the private subnet boundary without direct exposure to the public internet.
  * Configured with the primary read/write database instance in **AZ-1** and an automated Aurora Read Replica in **AZ-2** for intra-region high availability.
* **User Data Storage (Amazon S3):** Dedicated private S3 buckets securely persist customer uploads, generated invoices, and transactional attachments.

---

## 4. Disaster Recovery Strategy (`eu-west-1` Standby)

To honor data residency requirements while ensuring resilience, the secondary region in Ireland operates as a cold-compute, live-data standby:
1. **Zero-Baseline Compute Standby:**
   * In `eu-west-1`, the ECS Fargate auto-scaling configuration is set to a **desired capacity of 0** and a **minimum capacity of 0**.
   * No backend processing tasks run in Ireland during normal operations, preventing customer data from being processed outside the African region.
2. **Continuous Storage Synchronization:**
   * Persistent data is continuously replicated from `af-south-1` to a live Aurora PostgreSQL Read Replica located in `eu-west-1`.
3. **Automated Regional Activation:**
   * When AWS Global Accelerator detects an unrecoverable failure in `af-south-1`, incoming requests are shifted to the `eu-west-1` ALB.
   * Inbound requests trigger the ECS auto-scaling group to scale out from 0 to active Fargate compute instances.
   * The Aurora PostgreSQL read replica is promoted to primary writer status, restoring complete operational capabilities within the business RTO target.

---

## 5. Security, Auditing & Governance Architecture

* **AWS IAM:** Enforces least-privilege security roles across ECS task execution, database access policies, and administrative operations.
* **AWS KMS:** Manages customer encryption keys, ensuring persistent encryption at rest across S3 buckets, Aurora volumes, and ElastiCache clusters.
* **AWS CloudTrail:** Captures continuous, immutable account-wide audit logs of all API activity and administrative actions across both regions.
* **AWS Config:** Tracks infrastructure configuration state and continuously evaluates resource compliance against security baselines to prevent drift.

# Architecture Decision Records (ADRs) — TradeCore Africa Platform

**System Architect & Author:** Abdul-Hakeem Akinsile  

---

## ADR 01: Primary Production Region Selection (`af-south-1`)

### Status
Accepted

### Context
TradeCore Africa is a B2B company in Nigeria with strict data residency mandates requiring that all customer financial data be hosted and stored locally. However, AWS currently operates no physical data center or Region within Nigeria.

### Decision
Deploy the primary production infrastructure to **`af-south-1` (Cape Town, South Africa)**. 

### Rationale & Alternatives Considered
* **`af-south-1` (Selected):** Serves as the nearest available AWS region on the African continent, delivering acceptable latency for real-time transactions.
* **`eu-west-1` (Ireland) / `us-east-1` (N. Virginia) — Rejected for Primary:** Hosting primary transaction records in Europe or North America directly violates the intent of regional data residency and introduces unnecessary network round-trip delays.

### Trade-offs Accepted
Accepts an inter-country network latency profile between Nigeria and South Africa as the only viable African AWS region currently available.

---

## ADR 02: Disaster Recovery Architecture — Cold-Compute Standby in `eu-west-1`

### Status
Accepted

### Context
The business requires a multi-region deployment spanning at least two AWS regions to guarantee rapid disaster recovery, business continuity, and low RTO/RPO targets. However, continuously processing customer records outside the continent conflicts with the primary data residency mandate.

### Decision
Deploy a secondary disaster recovery site to **`eu-west-1` (Ireland)** configured as a **cold-compute, live-data standby**:
1. The compute tier (ECS on Fargate) maintains a baseline of **0 desired and 0 minimum tasks** during normal business operations.
2. Storage persistence is continuously maintained via an Aurora PostgreSQL Read Replica in `eu-west-1`.
3. Compute tasks automatically scale up from 0 to 1+ only upon an automated failover event triggered by regional downtime in `af-south-1`.

### Rationale & Alternatives Considered
* **Active-Active Across Regions — Rejected:** Running live compute in `eu-west-1` simultaneously would actively process customer financial data outside Africa during normal operations, breaching the residency mandate.
* **Cold Backup & Restore (Snapshots only) — Rejected:** Restoring snapshots into a cold database exceeds permissible downtime targets (RTO).

### Trade-offs Accepted
The standby environment requires a brief 60–90 second startup window for Fargate tasks to initialize upon traffic redirect before serving incoming requests.

---

## ADR 03: Global Traffic Ingress via AWS Global Accelerator & Route 53

### Status
Accepted

### Context
When the primary region (`af-south-1`) suffers an outage or performance degradation, client traffic must immediately and deterministically redirect to the DR region (`eu-west-1`).

### Decision
Implement **Amazon Route 53** for initial DNS resolution, routing client traffic through **AWS Global Accelerator** as the front-door ingress.

### Rationale & Alternatives Considered
* **AWS Global Accelerator (Selected):** Provides static Anycast IP addresses and routes traffic directly across AWS's private global fiber backbone rather than over the public internet, optimizing transit speed. It conducts continuous health checks against regional endpoints and automatically reroutes incoming traffic to `eu-west-1` if `af-south-1` is marked unhealthy.
* **Standard DNS Failover Routing — Deprioritized:** DNS record caching and TTL delays at intermediate ISP resolvers can delay failovers by several minutes, whereas Global Accelerator re-routes traffic across the backbone in seconds.

---

## ADR 04: Compute Tier — Amazon ECS on AWS Fargate

### Status
Accepted

### Context
The platform requires elastic compute capacity capable of scaling during traffic spikes while meeting strict recovery targets.

### Decision
Adopt **Amazon ECS using the AWS Fargate launch type** deployed within private subnets across multiple Availability Zones (`AZ-1` and `AZ-2`).

### Rationale & Alternatives Considered
* **AWS Fargate (Selected):** Serverless container compute boots and initializes tasks in approximately 60 to 90 seconds, enabling fast horizontal scaling during high-traffic surges and fast cold-starts during disaster recovery failovers.
* **Amazon EC2 Auto Scaling Groups — Rejected:** Traditional EC2 virtual instances take 2 or more minutes to boot the underlying operating system before container workloads can initialize, delaying auto-scaling and failover times.

---

## ADR 05: Database Architecture — Aurora PostgreSQL with Amazon RDS Proxy

### Status
Accepted

### Context
Compute workloads scale horizontally based on demand. Spikes in concurrent application containers opening direct database connections can exhaust database connection pools, degrade performance, and increase infrastructure costs.

### Decision
Deploy **Amazon Aurora PostgreSQL** within the private subnet, fronted by **Amazon RDS Proxy**.

### Rationale & Alternatives Considered
* **Amazon RDS Proxy (Selected):** Pools, multiplexes, and consolidates incoming database queries from ECS tasks and caching layers, reducing the total connection count on the database engine. This prevents Aurora from exhausting connection capacity during rapid task scale-outs and lowers database compute costs.
* **Direct Connection to Aurora — Rejected:** Forces the database to maintain an individual connection thread per container worker, requiring larger, more expensive instance classes solely to handle connection overhead.

---

## ADR 06: Subnet Placement of Data and Persistence Layers

### Status
Accepted

### Context
Databases, caches, and persistent user files must be protected from external direct internet exposure.

### Decision
Place the **Amazon Aurora PostgreSQL cluster, Amazon RDS Proxy, Amazon ElastiCache Redis, and S3 user data buckets within the Private Subnet tier**.

### Rationale
Restricting the database and connection proxy to private subnets ensures no public IPv4 addresses are assigned and prevents direct ingress from the internet. Ingress is restricted to internal communication originating from the compute layer and private VPC endpoints.

---

## ADR 07: Outbound Connectivity via a Single Regional Shared NAT Gateway

### Status
Accepted

### Context
ECS tasks running in private subnets require outbound internet access to download container patches, system updates, and communicate with external payment APIs.

### Decision
Deploy **one shared Regional NAT Gateway** in the public subnet of `af-south-1` to service private subnets across both availability zones (`AZ-1` and `AZ-2`).

### Rationale & Alternatives Considered
* **Single Shared Regional NAT Gateway (Selected):** Eliminates the hourly base charge of deploying a dedicated NAT Gateway per Availability Zone while providing the required outbound egress for update workflows.
* **Multi-NAT Gateway Deployment — Rejected on Cost:** Deploying redundant NAT Gateways across each individual AZ doubles fixed networking operational costs without providing proportional value for outbound-only maintenance traffic.
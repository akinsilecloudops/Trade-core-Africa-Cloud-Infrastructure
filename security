# Security & Compliance Architecture — TradeCore Africa Platform

**System Architect & Author:** Abdul-Hakeem Akinsile  
**Classification:** Financial PII / Tier-1 Banking Compliance  
**Scope:** Identity, Network Segmentation, Cryptographic Key Management & Regulatory Alignment  

---

## 1. Security Architecture & Threat Defense Overview

Security across TradeCore Africa is designed around a multi-layered defense-in-depth model. Because the platform processes high-value corporate financial transactions and sensitive financial records, the architecture guarantees that no single perimeter breach can compromise data confidentiality or transactional integrity.
---

## 2. Network Segmentation & Perimeter Protection

### 2.1 Web Application Firewall (AWS WAF)
Traffic entering through the perimeter is inspected at Layer 7 by AWS WAF prior to reaching internal load balancers:
* **Common Rule Set (`AWSManagedRulesCommonRuleSet`):** Blocks typical web application vulnerabilities, including cross-site scripting (XSS), path traversal, and abnormal HTTP methods.
* **SQL Injection Protection (`AWSManagedRulesSQLiRuleSet`):** Inspects incoming URI query parameters and request payloads to block database injection exploits.
* **Known Bad Inputs & IP Reputation:** Filters known malicious IP addresses, tor exit nodes, and automated bot networks.
* **Rate-Based Throttling:** Restricts individual client IPs to a maximum request threshold over a 5-minute rolling window, preventing denial-of-service (DoS) attempts and credential brute-force attacks.

### 2.2 Subnet Segregation
* **Public Subnet:** Houses exclusively the internet-facing Application Load Balancer (ALB) and the shared regional NAT Gateway. No compute workloads or database engines exist here.
* **Private Subnet:** Houses Amazon ECS Fargate tasks, Amazon ElastiCache Redis, the Amazon RDS Proxy layer, Amazon Aurora PostgreSQL, and S3 user data buckets. Resources in this tier have no public IPv4 addresses assigned and are inaccessible directly from the public internet. Outbound traffic (e.g., container updates) is routed exclusively through the NAT Gateway.

### 2.3 Chained Security Group Architecture
Ingress rules use security group referencing rather than CIDR blocks to enforce least-privilege network paths:
1. **ALB Security Group:** Allows inbound TCP 443 (HTTPS) from public internet traffic; blocks all non-TLS traffic (port 80 redirects via 301 to port 443).
2. **ECS Compute Security Group:** Allows inbound traffic strictly on the application container port originating from the ALB Security Group. All direct public ingress is blocked.
3. **ElastiCache Redis Security Group:** Accepts connections strictly on port 6379 originating from the ECS Compute Security Group.
4. **RDS Proxy Security Group:** Accepts inbound database connections on port 5432 strictly from the ECS Compute Security Group.
5. **Aurora Database Security Group:** Accepts PostgreSQL connections strictly on port 5432 originating from the RDS Proxy Security Group, ensuring the database cannot be queried directly even by application containers bypassing the proxy.

---

## 3. Cryptographic Key Management (AWS KMS)

All data at rest across the platform is protected using hardware-backed **Customer-Managed Keys (CMKs)** in AWS Key Management Service (KMS) with annual automatic rotation:

| KMS Key Name | Target Architectural Resources | Cryptographic Purpose |
| :--- | :--- | :--- |
| **Database CMK** | Amazon Aurora PostgreSQL, ElastiCache Redis | Encrypts database storage volumes, automated snapshots, transaction logs, and in-memory cache data at rest. |
| **Documents CMK** | Amazon S3 (`User Data` buckets) | Protects sensitive billing records, customer invoices, and financial statement uploads. |
| **Audit Logs CMK** | CloudWatch Log Groups, VPC Flow Logs, CloudTrail S3 Bucket | Encrypts operational logging streams, system metrics, and audit log files. |
| **Secrets CMK** | AWS Secrets Manager | Secures Aurora database credentials, API integration tokens, and service credentials. |

---

## 4. Disaster Recovery Data Protection ("Zero-Knowledge" Standby)

To maintain disaster recovery capabilities in `eu-west-1` while upholding regional data sovereignty principles:
1. **Ciphertext Storage Only:** The secondary region in Ireland (`eu-west-1`) stores only encrypted storage pages replicated via the Aurora replication mechanism.
2. **KMS Replica Key Access Restriction:** The KMS replica key policy in `eu-west-1` explicitly denies the `kms:Decrypt` permission to all human operators and IAM application roles. Decryption rights are restricted exclusively to the internal AWS Aurora replication service principal (`rds.eu-west-1.amazonaws.com`) to facilitate storage-layer synchronization.
3. **No Standby Compute Processing:** Because ECS Fargate task capacity in `eu-west-1` is kept at a desired count of 0 during normal operations, no compute instances exist to read, decrypt, or process customer transactions in the secondary region.

---

## 5. Identity & Access Management (IAM)

Access governance enforces the principle of least privilege across all service principals:
* **Separation of Task Execution & Task Roles:**
  * **ECS Execution Role:** Grants permissions required to bootstrap the container (pulling images from Amazon ECR, pushing container logs to CloudWatch).
  * **ECS Task Role:** Scoped strictly to application runtime requirements (e.g., querying RDS Proxy, reading/writing to the user data S3 bucket, decrypting application secrets).
* **No Long-Lived Credentials:** Application workloads authenticate to AWS services using temporary, short-lived STS credentials automatically rotated by the Fargate runtime; no static access keys or secret keys are stored within container images.
* **Database Credential Management:** Aurora database master credentials are automatically generated, managed, and rotated by AWS Secrets Manager.

---

## 6. Auditability, Governance & Compliance Tracking

* **AWS CloudTrail (All Regions):** Operates across `af-south-1` and `eu-west-1` with cryptographic log file validation enabled. CloudTrail captures every API invocation, administrative modification, and KMS decryption request for forensic auditability.
* **AWS Config:** Tracks configuration state and continuously evaluates resource settings against security conformance baselines (e.g., verifying that all S3 buckets deny unencrypted HTTP requests and all database instances enforce storage encryption).
* **Log Immutability via S3 Object Lock:** Centralized audit logs deposited in Amazon S3 are configured with **S3 Object Lock in Compliance Mode**, ensuring that log archives cannot be overwritten, modified, or prematurely deleted by any IAM user or administrator during the mandatory retention window.


# Monitoring, Logging & Observability — TradeCore Africa Platform

**System Architect & Author:** Abdul-Hakeem Akinsile  
**Scope:** Observability, Metric Math Alarms, Audit Immutability & Compliance  
**Regions Covered:** `af-south-1` (Primary) & `eu-west-1` (Standby DR)  

---

## 1. Observability Architecture Overview

For a financial infrastructure handling over ₦180M in daily transaction volume, observability is structured into three continuous operational pillars:
1. **Infrastructure & Application Metrics:** Real-time operational telemetry collected across every tier (ALB, ECS Fargate, ElastiCache Redis, RDS Proxy, and Aurora PostgreSQL).
2. **Immutable Audit Trails:** Permanent, tamper-proof audit records of every administrative action, data access attempt, and network flow.
3. **Automated Alarm Dispatch:** Metric-math and threshold alarms that alert operations teams and initiate automated remediation workflows.
---

## 2. Centralized Logging & Audit Trails

To comply with Tier-1 banking audit standards and regulatory frameworks, log streams are cryptographically secured and strictly partitioned:

### 2.1 AWS CloudTrail (Multi-Region Trail)
* **Scope:** Captures all management events and data-plane API actions across both `af-south-1` and `eu-west-1`.
* **Log File Validation:** Enabled using cryptographic SHA-256 digest chains to verify that log files have not been modified, deleted, or altered post-delivery.
* **KMS Event Visibility:** Logs every `kms:Decrypt` and `kms:GenerateDataKey` invocation, ensuring every access attempt on customer PII is permanently attributed to an IAM principal.

### 2.2 VPC Flow Logs
* **Coverage:** Enabled on all subnets across both Availability Zones (`AZ-1` and `AZ-2`).
* **Ingress/Egress Auditing:** Delivered to CloudWatch Logs and archived to S3, establishing a complete history of network traffic across public, private, and database boundaries.
* **Data Residency Verification:** Flow logs actively monitor egress traffic to verify that unencrypted customer data never leaves the `af-south-1` network perimeter.

### 2.3 Application & Container Logging
* **Mechanism:** ECS Fargate tasks stream `stdout` and `stderr` directly to Amazon CloudWatch Logs using the `awslogs` log driver.
* **Log Retention:** Hot log groups retain data for 30 days in CloudWatch for live debugging and query analysis via CloudWatch Logs Insights, followed by automated lifecycle archiving to Amazon S3.

### 2.4 Immutable S3 Audit Destination
* **Destination Bucket:** `tradecore-production-logs`
* **S3 Object Lock:** Deployed in **Compliance Mode** with a **7-year default retention period**, preventing any user—including AWS root accounts—from deleting or altering audit logs.
* **Bucket Policy:** Enforces TLS 1.2+ (`DenyInsecureConnections`) and strictly denies unencrypted PUT requests.

---

## 3. Tier-Specific Monitoring Dashboards

CloudWatch is organized into modular dashboards to provide isolated operational insight into each architectural layer:

| Architectural Tier | Monitored Metrics | Operational Significance |
| :--- | :--- | :--- |
| **Edge & Ingress (ALB & WAF)** | Request Count, Target Response Time, HTTP 4xx/5xx counts, WAF Blocked Requests | Identifies traffic spikes, edge latency bottlenecks, and active Layer 7 DDoS/injection attempts. |
| **Compute Tier (ECS Fargate)** | CPU Utilization, Memory Utilization, Running vs. Desired Task Count | Informs target-tracking auto-scaling and detects task memory leaks or crash loops. |
| **In-Memory Cache (Redis)** | Cache Hits/Misses, `EngineCPUUtilization`, Evictions, Memory Fragmentation Ratio | Tracks cache efficiency, payload sizing, and memory saturation before queries reach the database. |
| **Database Proxy (RDS Proxy)** | `DatabaseConnectionsCurrentlyInUse`, Query Latency, Client Connection Count | Detects connection starvation and ensures pool multiplexing protects Aurora during compute spikes. |
| **Database Tier (Aurora)** | Write/Read Latency, Aurora Buffer Cache Hit Ratio, Replica Lag, Storage Used | Guarantees transactional throughput, detects long-running locking queries, and monitors replication health. |

---

## 4. Production CloudWatch Alarms & Alert Thresholds

Alarms use metric math expressions where applicable to evaluate relative error percentages rather than raw counts, preventing false-positive alerts during natural low-traffic hours:

### 1. ALB Target 5xx Error Rate Alarm
* **Metric Expression:** `(HTTPCode_Target_5XX_Count / RequestCount) * 100`
* **Threshold:** `> 1.0%` of total requests
* **Evaluation Window:** 2 consecutive evaluation periods (1-minute datapoints)
* **Action:** Triggers high-priority SNS alert for immediate application inspection.

### 2. RDS Proxy Connection Pool Saturation
* **Metric Expression:** `(DatabaseConnectionsCurrentlyInUse / MaxConnections) * 100`
* **Threshold:** `> 80.0%` of proxy pool ceiling
* **Evaluation Window:** 2 consecutive datapoints
* **Action:** Alerts backend team to evaluate task connection lifecycles and database pool allocation.

### 3. Aurora Cross-Region Replication Lag
* **Metric:** `AuroraGlobalDBReplicationLag` (evaluating storage sync to `eu-west-1`)
* **Threshold:** `> 30 seconds`
* **Evaluation Window:** 1 datapoint (evaluated every minute)
* **Action:** Pagers on-call engineers regarding potential disaster recovery degradation and RPO risk.

### 4. ElastiCache Redis Memory Utilization
* **Metric:** `DatabaseMemoryUsagePercentage`
* **Threshold:** `> 80.0%`
* **Evaluation Window:** 2 evaluation periods of 5 minutes
* **Action:** Alerts infrastructure team to adjust cache eviction policies or increase node sizing (`cache.t4g.small` scale-up).

### 5. Minimum ECS Task Availability
* **Metric:** `CPUUtilization` paired with `HealthyHostCount` on the ALB target group
* **Threshold:** Healthy task count drops below minimum baseline (`< 3 tasks`)
* **Evaluation Window:** Immediate (1 datapoint of 1 minute)
* **Action:** Alerts on unexpected container terminations and initiates automated task replacement.

---

## 5. Security & Compliance Monitoring

* **AWS Config Rules:**
  * `encrypted-volumes`: Continuously validates that all EBS and Aurora storage volumes are encrypted with KMS.
  * `s3-bucket-ssl-requests-only`: Confirms that every S3 bucket policy blocks unencrypted HTTP traffic.
  * `incoming-ssh-disabled`: Validates that no security group rule permits open port 22 access.
* **CloudTrail Insights:** Automatically analyzes management events to detect anomalous administrative activities, sudden bursts of IAM permission modifications, or unusual credential usage.

---

## 6. Operational Incident Runbooks

When an alert triggers, engineers execute the following standardized diagnostic commands via the AWS CLI:


### 1. Investigate active ECS tasks and task health
aws ecs list-tasks \
  --region af-south-1 \
  --cluster tradecore-primary-cluster \
  --desired-status RUNNING

### 2. Query recent 5xx error events from CloudWatch Logs Insights
aws logs start-query \
  --region af-south-1 \
  --log-group-name "/ecs/tradecore-backend" \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /500|Error|Exception/ | sort @timestamp desc | limit 50'

### 3. Check Aurora PostgreSQL cluster replication lag to DR
aws rds describe-db-clusters \
  --region af-south-1 \
  --db-cluster-identifier tradecore-primary-cluster \
  --query "DBClusters[0].DBClusterMembers[*].[DBInstanceIdentifier,IsClusterWriter]"

### 4. Verify KMS log decryption audit trails
aws cloudtrail lookup-events \
  --region af-south-1 \
  --lookup-attributes AttributeKey=EventName,AttributeValue=Decrypt \
  --max-results 20

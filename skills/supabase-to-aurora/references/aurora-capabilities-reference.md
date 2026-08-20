# Aurora PostgreSQL — Capabilities Reference

Reference for Aurora PostgreSQL capabilities that are relevant when recommending migration from managed PostgreSQL platforms. Use this when presenting value propositions to users during the Assess phase or when answering "why should I migrate?" questions.

## Enterprise Reliability

- **99.99% SLA** for Multi-AZ deployments (~52 minutes allowed downtime/year); 99.9% for Single-AZ
- Storage replicates synchronously across **6 nodes in 3 Availability Zones** — tolerates loss of an entire AZ
- Automatic failover to a read replica, typically **< 30 seconds**
- Service credits up to **100%** of affected charges if SLA is breached
- No quota-based suspensions or project freezes — service continues regardless of traffic spikes

## Multi-Region Disaster Recovery

- **Aurora Global Database** — cross-region replication with sub-second latency
- RPO typically **seconds**, RTO typically **under a minute** for managed cross-region failover
- Up to **10 secondary DB clusters** in different regions with local read access
- Write forwarding from secondary regions to primary (active-passive with write forwarding)
- Cross-region snapshot copy for offline DR

## Scale

- Writer instances up to **192 vCPU / 1,536 GB RAM** (db.r8g.48xlarge)
- Storage auto-scales to **256 TiB** per cluster (PG 17.5+, 16.9+, 15.13+) or **128 TiB** on earlier versions — no manual intervention
- Up to **15 read replicas** sharing the same storage volume (zero per-replica storage cost)
- **Aurora Auto Scaling** — dynamically adds/removes replicas based on CloudWatch metrics
- **Serverless v2** — scales from 0 ACU (auto-pause, on supported versions) or 0.5 ACU to 256 ACUs without disruption
- Supports **hundreds of thousands of transactions per second**
- No connection-count-based throttling or suspension

## Cost Predictability

- **Reserved Instances** — up to 69% savings on compute (3-year All Upfront); up to 60% with Partial Upfront
- **Savings Plans** — flexible committed-use discounts across instance families
- **I/O-Optimized** configuration — eliminates per-request I/O charges for I/O-heavy workloads
- No surprise overages — pricing is usage-based with transparent dimensions (compute, storage, I/O, backup, transfer)
- Read replicas share storage volume — **no additional per-replica storage cost** (unlike platforms that require 1.25x primary disk per replica)
- At production scale (500 GB, read replicas, PITR), Aurora can be significantly cheaper than equivalent managed platform configurations due to shared storage (no per-replica disk cost), included backup storage, and committed-use discounts (RIs up to 69% off)

### Cost Crossover Point

Aurora costs more at small scale (< 50 GB, single instance, no PITR) because of the always-on minimum compute cost (~$44/month for Serverless v2 at 0.5 ACU). The cost advantage shifts to Aurora when:

- Database exceeds ~100 GB (storage pricing favors Aurora)
- PITR is needed (Aurora includes backup storage free up to cluster size; platforms often charge $100–400/month extra)
- Read replicas are added (Aurora replicas share storage at zero additional disk cost)
- Committed pricing is used (RIs reduce Aurora compute by 30–60%)

## Compliance & Certifications

- **FedRAMP** Moderate and High (GovCloud)
- **GovCloud** (US) regions for classified and regulated workloads
- **HIPAA** eligible with encryption at rest (AES-256) and in transit
- **PCI DSS**, SOC 1/2/3, ISO 27001, ISO 27017, ISO 27018
- IL4/IL5 support for defense workloads (GovCloud)
- Full **CloudTrail** audit trail for all API calls
- **pgAudit** extension for database-level activity logging

## Network Isolation & Security

- Runs inside customer VPC — **never exposed to public internet** by default
- Security groups, NACLs, VPC peering, PrivateLink for fine-grained network control
- **IAM database authentication** — applications authenticate with IAM credentials, no password management
- **Secrets Manager** integration with automatic credential rotation
- **KMS encryption** at rest (AWS-managed or customer-managed keys)
- Enforced SSL/TLS in transit via `rds.force_ssl` parameter

## Operational Control

- Extensive **parameter group customization** — shared_buffers, work_mem, effective_cache_size, max_connections, and hundreds more
- **90+ supported extensions** on PostgreSQL 17, including:
  - `aws_s3` — direct S3 import/export from SQL
  - `aws_lambda` — invoke Lambda from SQL
  - `aws_ml` — call SageMaker/Comprehend from SQL
  - `apg_plan_mgmt` — query plan capture and enforcement (prevent plan regressions)
  - `pg_tle` — install custom extensions without AWS involvement
  - `oracle_fdw`, `mysql_fdw`, `tds_fdw` — federated queries to other engines
- **Blue/green deployments** — near-zero-downtime major version upgrades (switchover under 1 minute)
- Custom endpoints, cluster topology control, independent replica sizing

## AWS Ecosystem Integration

- **Performance Insights** — real-time, per-second query-level load analysis with up to 2 years retention
- **Enhanced Monitoring** — OS-level metrics at 1-second granularity (CPU, memory, I/O per process)
- **DevOps Guru for RDS** — ML-powered anomaly detection and automated root cause analysis
- **Zero-ETL to Redshift** — real-time analytics without building ETL pipelines
- **Aurora ML** — invoke SageMaker and Comprehend models directly from SQL queries
- **RDS Proxy** — fully managed connection pooler for Lambda and high-concurrency workloads
- **CloudWatch** — 60+ Aurora-specific metrics with native alerting
- **EventBridge** — capture RDS events (failover, maintenance, snapshots) for automation
- **AWS Backup** — centralized backup management, cross-account copy, lifecycle policies

## Performance

- Up to **3x throughput** of standard PostgreSQL (purpose-built distributed storage subsystem)
- Read replicas use **storage-level physical replication** — lag typically < 100ms (vs seconds for WAL-based logical replication)
- Replicas remain available for reads **even during writer restarts**
- **Query plan management** (apg_plan_mgmt) prevents plan regressions after statistics refresh or upgrades
- Serverless v2 scales compute **instantly** with demand — no cold start or manual tier change
- **Optimized Reads (NVMe SSD)** — `db.r6gd` and `db.r6id` instance classes include locally attached NVMe SSDs; temporary files and temp tables are automatically mapped to local NVMe (via `aurora_temp_tablespace`) instead of EBS, delivering up to **8x query latency improvement** for I/O-intensive workloads (complex sorts, GROUP BY, index builds, CTEs). ~90% of NVMe storage is available for temporary objects (Standard) or split between temp objects and tiered cache (I/O-Optimized)

## When Migration Makes Sense

Teams typically migrate to Aurora when they need one or more of:

1. **Multi-AZ automatic failover** — workload can't tolerate single-region failure
2. **Compliance certification** — FedRAMP, GovCloud, or deep audit requirements
3. **Scale beyond platform ceiling** — hitting connection limits, compute caps, or storage limits
4. **Cost predictability** — need committed pricing without surprise overages or suspensions
5. **VPC isolation** — workload requires no public internet exposure
6. **Operational control** — need full PostgreSQL parameter tuning, custom extensions, or blue/green upgrades
7. **AWS ecosystem** — deep integration with IAM, CloudTrail, Lambda, SageMaker, Redshift, etc.
8. **Performance at scale** — need 3x throughput, automatic read scaling, or hundreds of thousands of TPS

## When Migration May Not Be Worth It

- Small database (< 50 GB) on a free or low-cost plan with no production path
- No compliance, scale, or reliability requirements beyond what the current platform provides
- Team has no AWS experience and the platform's developer experience is a key productivity factor
- Purely prototype/demo with no intent to reach production

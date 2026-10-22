# Week 7: Database Services Decision Record

**Organization:** HarborTech
**Author:** Cloud Operations Team (Camden Elam)
**Date:** October 21, 2026

---

## 1. Current Database Condition

| Item | Current State |
|---|---|
| Database engine and version | MySQL 8.0.34 (Community Edition) |
| Hosting | Self-managed on a single Amazon EC2 instance |
| Instance type / vCPU / RAM | m5.xlarge, 4 vCPU, 16 GiB RAM |
| Storage (EBS type, size, IOPS) | gp3, 500 GB, 6,000 provisioned IOPS |
| Data size and monthly growth | 210 GB used, growing about 6 GB per month |
| Number of databases / schemas | 3 (`orders`, `inventory`, `reporting`) |
| Replication / failover | None. A manual standby was discussed but never built. If the instance fails, recovery means launching a new instance and restoring from backup. |
| Backup method and schedule | Nightly `mysqldump` to Amazon S3 and a weekly EBS snapshot. 14-day retention. No point-in-time recovery. |
| Patching process and last patch date | Manual, quarterly in theory. Last OS and MySQL patching was 7 months ago. |
| Last restore test | 11 months ago. Full restore took about 6.5 hours. |
| Known pain points | Single point of failure; restore time far exceeds the 1-hour RTO target; up to 24 hours of data loss possible between nightly dumps; patching is manual and overdue; month-end CPU spikes; one administrator holds most of the knowledge. |

**Summary:** The MySQL database supports HarborTech's order management and inventory systems and performs adequately day to day. The main risks are operational: no automatic failover, backups that do not meet recovery targets, and overdue patching. These gaps, rather than performance, are driving this evaluation.

---

## 2. Workload Evidence

The decision depends on what the workload does, not on what is popular. Evidence collected before comparing services:

| Evidence Area | Question | Finding |
|---|---|---|
| Data structure | Is the data tabular and fixed-schema, or flexible documents/key-value items? | Highly structured and tabular. 42 tables with a stable schema, changed only through planned releases. |
| Relationships | Are there foreign keys and multi-table joins central to the application? | Yes. 118 foreign key relationships. Core queries join customers, orders, order lines, products, and inventory. |
| Transactions | Are multi-row, multi-table ACID transactions required? | Yes. Order placement updates 4 tables (orders, order lines, inventory, payments) and must succeed or fail as one unit. |
| SQL needs | Does the app, reporting, or BI tooling depend on SQL, stored procedures, triggers, or extensions? | Heavily SQL dependent. 23 stored procedures, 9 triggers, and a BI tool that runs ad hoc SQL. No unsupported MySQL plugins in use. |
| Access patterns | Are queries known and key-based, or ad hoc and varied? Read/write ratio? | Mixed. About 85% of traffic is application queries through an ORM. About 15% is ad hoc reporting joins. Read/write ratio is roughly 75/25. |
| Throughput and latency | Peak and average queries/sec, connection count, latency targets | Average 450 queries/sec, peak 1,800 queries/sec at month-end. Connections average 140, peak 320. p95 query latency is 45 ms. Target is under 100 ms. |
| Scaling shape | Steady, spiky, or unpredictable? Read-heavy? | Steady during the week with predictable month-end spikes. Average CPU 38%, peak 82%. Moderately read-heavy. |
| Availability | Required uptime, acceptable downtime, RTO | 99.9% target (about 8.8 hours of downtime per year). RTO of 1 hour. Business hours are 6 AM to 10 PM Central. |
| Backup and recovery | RPO, retention period, point-in-time recovery needs | RPO of 5 minutes. 30-day retention. Point-in-time recovery required. Current setup meets none of these. |
| Application dependencies | Which applications, jobs, and reports connect? Drivers, ORMs, engine-specific features? | Order portal (Java/Spring, MySQL JDBC driver), warehouse scanner API (Python), nightly ETL job to the reporting schema, and a BI reporting tool. All use standard MySQL connections. |
| Compliance and security | Encryption, auditing, data residency, access control requirements | Encryption at rest and in transit required. Audit trail for access to customer data. Data must stay in US regions. |

**Evidence sources:** CloudWatch metrics (30 days), MySQL slow query log, `information_schema` schema export, and interviews with the order portal and warehouse application owners.

**Conclusion from evidence:** The workload is relational, SQL-driven, and transactional. Its access patterns are not narrow or key-based, so a NoSQL redesign would be costly and risky. The largest gaps are availability and recovery, which managed relational services address directly.

---

## 3. Service Comparison

| Criteria | Keep on EC2 | Amazon RDS (MySQL) | Amazon Aurora (MySQL-compatible) | Amazon DynamoDB |
|---|---|---|---|---|
| Data model | Any engine you install | Relational (MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Db2) | Relational (MySQL- and PostgreSQL-compatible) | NoSQL key-value and document |
| SQL and joins | Full | Full | Full, MySQL compatible | No SQL joins; PartiQL available for limited queries |
| Transactions | Full ACID | Full ACID | Full ACID | Supported, with item-count and design limits |
| Schema | Fixed | Fixed | Fixed | Flexible; access patterns drive design |
| Scaling | Resize instance manually | Vertical scaling and read replicas | Vertical scaling plus up to 15 read replicas on shared storage | Horizontal, on-demand or provisioned capacity |
| High availability | Build and maintain yourself | Multi-AZ option | Storage replicated across 3 AZs; fast failover | Built in across multiple AZs |
| Backups | Build and test yourself | Automated backups, snapshots, point-in-time restore | Continuous backup, snapshots, point-in-time restore | On-demand backup and point-in-time recovery |
| OS and engine patching | Customer | AWS-managed with maintenance windows | AWS-managed with maintenance windows | No servers or engine to patch |
| Migration effort for HarborTech | None | Low. Same engine and version family. | Moderate. Compatibility testing for stored procedures and triggers. | High. Full data model redesign and application rewrite. |
| Fit for HarborTech workload | Poor on availability and recovery | **Strong** | Good, but likely more than needed | Poor. Relational joins and transactions are central. |
| Estimated monthly cost (illustrative) | About $150 to $200 including storage, plus administrator time | About $550 to $700 for Multi-AZ `db.m5.xlarge` with storage and backups | About $700 to $950 depending on I/O and replicas | Not estimated. Redesign cost outweighs any savings. |
| Main risk | Operational burden and error risk | Less OS-level control; higher cost than a single EC2 | Higher cost; benefits not needed at current scale | Poor fit; large rewrite and retesting |

**Reading the table:** DynamoDB is ruled out because HarborTech depends on joins, foreign keys, stored procedures, and multi-table transactions. Aurora would work, but measured load (1,800 queries/sec peak, 82% peak CPU) does not yet justify its extra cost. RDS for MySQL Multi-AZ closes the availability, backup, and patching gaps with the lowest migration effort.

---

## 4. Virtualization Connection

- Self-managed databases on EC2 run on virtual machines. The hypervisor (AWS Nitro) shares physical hardware across isolated instances, and EBS provides virtualized block storage. HarborTech's current MySQL server is one such virtual machine, so a failure of that instance is a failure of the whole database.
- RDS and Aurora are built on the same virtualization foundation. The database still runs on compute and storage resources, but AWS manages the instance, operating system, and database engine layer. With RDS Multi-AZ, a standby instance in a second Availability Zone is kept in sync, which is the kind of redundancy HarborTech has not built itself.
- Aurora goes further by separating compute from a distributed storage layer, which is why storage can replicate across Availability Zones independently of the DB instances.
- DynamoDB abstracts the servers entirely. The customer sees tables and capacity settings, not instances.
- **Takeaway:** Moving up the managed-service ladder hides more of the virtualized infrastructure. That reduces administration but also reduces direct control. Performance still depends on choosing the right instance class, storage, and capacity settings.

---

## 5. Responsibility Change

Under the AWS shared responsibility model, the boundary moves as HarborTech adopts managed services.

| Responsibility | EC2-hosted DB | RDS / Aurora | DynamoDB |
|---|---|---|---|
| Physical hardware and facilities | AWS | AWS | AWS |
| Hypervisor and virtualization | AWS | AWS | AWS |
| Guest OS patching | **HarborTech** | AWS | AWS |
| Database engine install and patching | **HarborTech** | AWS (HarborTech chooses window and version) | AWS |
| Backup configuration and restore testing | **HarborTech** | Shared: AWS provides the feature; HarborTech sets retention and tests restores | Shared: HarborTech enables PITR/backups and tests restores |
| High availability design | **HarborTech** | Shared: HarborTech enables Multi-AZ/replicas | Built in; HarborTech decides on global tables if needed |
| Network access and security groups | **HarborTech** | **HarborTech** | **HarborTech** (IAM, VPC endpoints) |
| Database users, IAM, encryption settings | **HarborTech** | **HarborTech** | **HarborTech** |
| Schema and data design | **HarborTech** | **HarborTech** | **HarborTech** |
| Query tuning and indexing | **HarborTech** | **HarborTech** | **HarborTech** (key and index design) |
| Application integration and testing | **HarborTech** | **HarborTech** | **HarborTech** |
| Data accuracy and business requirements | **HarborTech** | **HarborTech** | **HarborTech** |

**What HarborTech stops doing:** Patching the OS and MySQL, building failover, and maintaining backup scripts.

**What HarborTech still owns:** Schema design, slow query tuning, user and access management, encryption choices, restore testing, application connection changes, and confirming that the data and business results are correct.

**Key point:** A managed service reduces server and database infrastructure administration. It does not remove responsibility for data design, access control, query behavior, application integration, validation, or business requirements.

---

## 6. Migration Considerations

- **Engine compatibility:** RDS for MySQL 8.0 supports the current version. Confirm the 23 stored procedures, 9 triggers, and any custom `my.cnf` settings map to RDS parameter groups. No `SUPER` privilege or OS-level access is available on RDS, so check whether any job depends on them.
- **Tools:** AWS Database Migration Service (DMS) with full load plus change data capture (CDC) for low-downtime cutover. Native `mysqldump` or a Percona XtraBackup restore are fallback options. AWS Schema Conversion Tool is not needed because the engine stays the same.
- **Downtime tolerance:** Target a cutover window under 30 minutes, scheduled on a Sunday night outside business hours and away from month-end.
- **Connection changes:** New RDS endpoint, updated connection strings in the order portal, scanner API, ETL job, and BI tool. Store credentials in AWS Secrets Manager. Require TLS.
- **Network and security:** Place RDS in private subnets across two AZs through a DB subnet group. Restrict the security group to application servers. Enable encryption at rest with AWS KMS. Enable audit logging for customer data access.
- **Performance baseline:** Capture current p95 latency (45 ms), peak QPS (1,800), and peak CPU (82%) so post-migration results can be compared fairly. Enable Performance Insights on the new instance.
- **Sizing:** Start with `db.m5.xlarge` (4 vCPU, 16 GiB) to match the current instance. Use gp3 storage with autoscaling. Right-size after 30 days of data.
- **Cost model:** Estimated $550 to $700 per month for Multi-AZ versus about $150 to $200 for the current EC2 setup. The increase buys automatic failover, point-in-time recovery, automated patching, and reduced administrator time. Verify with the AWS Pricing Calculator.
- **Rollback plan:** Keep the EC2 database intact and in sync (reverse replication or preserved snapshot) for 14 days after cutover. Roll back if data mismatch or severe performance loss is found within the first 48 hours.
- **Rehearsal:** Run the full migration at least twice in a non-production environment, including a timed cutover.
- **Operational handoff:** Update runbooks, CloudWatch alarms, on-call procedures, and the backup and restore documentation.

---

## 7. Recommendation

**Recommended path:** Migrate to **Amazon RDS for MySQL 8.0 with Multi-AZ deployment**, using AWS DMS for a low-downtime cutover. Keep Aurora as a future option if load grows. Do not pursue DynamoDB for this workload.

| If the evidence shows... | Then recommend... | HarborTech result |
|---|---|---|
| Relational data, SQL, joins, standard HA and backup needs | **Amazon RDS** | **Matches.** Structured data, 118 foreign keys, ACID transactions, and standard HA and recovery requirements. |
| Relational workload needing higher throughput, many read replicas, or faster failover | **Amazon Aurora** | Not yet. Peak load is 1,800 QPS with moderate read scaling needs. Revisit if peak load doubles or the reporting load needs dedicated read replicas. |
| Known key-value access patterns, very high scale, flexible schema | **Amazon DynamoDB** | Does not match. Access patterns are relational and ad hoc. |
| Unsupported engine, OS-level access required, or migration not justified | **Keep on EC2** | Does not match. No OS-level dependency identified, and current gaps cannot be closed cheaply on EC2. |

**Rationale:** The workload is relational, transactional, and SQL dependent, so RDS is the natural fit with the lowest migration effort. The biggest HarborTech risks are a single point of failure, a 6.5-hour restore against a 1-hour RTO, and up to 24 hours of potential data loss against a 5-minute RPO. RDS Multi-AZ and automated backups with point-in-time recovery address these directly. Aurora's added capability is not supported by current measurements, and DynamoDB would require a full redesign.

**Risks accepted:** Higher monthly cost; less OS-level control; need to retest stored procedures and triggers; short cutover downtime; staff must learn RDS operations.

**Next step:** Approve a non-production proof of concept (restore a copy of the data to RDS, run the order portal and ETL against it, and measure performance). No production change until validation items in Section 8 are complete and approved.

---

## 8. Validation Items

Complete before any production cutover:

- [ ] Workload evidence in Section 2 confirmed by application owners
- [ ] Application owners confirmed compatibility and dependencies (portal, scanner API, ETL, BI tool)
- [ ] Target engine and version supports all 23 stored procedures and 9 triggers
- [ ] Test migration completed in non-production
- [ ] Row counts and checksums match between source and target for all 42 tables
- [ ] Application functional tests pass against the target, including order placement across 4 tables
- [ ] Month-end load test meets or beats baseline (p95 under 100 ms at 1,800 QPS)
- [ ] Multi-AZ failover test completed and measured
- [ ] Point-in-time restore test completed, with RTO under 1 hour and RPO under 5 minutes confirmed
- [ ] Security review done: KMS encryption, TLS, IAM, security groups, audit logging
- [ ] Monitoring and alerts configured (CloudWatch alarms, Performance Insights)
- [ ] Cost estimate reviewed and approved
- [ ] Rollback plan tested and documented
- [ ] Change window and communication plan approved
- [ ] Runbooks updated

---

## 9. Escalation Notes

Escalate before proceeding if any of these occur:

| Situation | Escalate To |
|---|---|
| Application depends on unsupported engine features or privileges | Application owner / architect |
| Test migration shows data mismatch or loss | Database owner and project lead |
| Performance is worse than baseline | Architect; consider a larger instance class or evaluate Aurora |
| Compliance, encryption, or residency requirement cannot be met | Security / compliance lead |
| Estimated cost exceeds approved budget | IT manager / finance contact |
| Required downtime exceeds what the business accepts | Business owner (Operations Director) |
| Rollback is needed during cutover | Incident lead and application owner |

**Contacts (fictional):**

| Role | Name | Channel |
|---|---|---|
| Project lead | Dana Whitfield | #db-migration (Slack) |
| Database owner | Marcus Reyes | marcus.reyes@harbortech.example |
| Order portal owner | Priya Nair | priya.nair@harbortech.example |
| Warehouse systems owner | Tom Okafor | tom.okafor@harbortech.example |
| Security / compliance | Elena Brandt | security@harbortech.example |
| Business owner | Operations Director | ops-director@harbortech.example |

---

## 10. Lessons Learned

- Evidence should drive the service choice. Starting from a preferred service leads to rework.
- "Managed" does not mean "hands-off." HarborTech remains responsible for data, access, queries, and validation.
- DynamoDB and relational services solve different problems. The choice depends on access patterns, not on newness.
- Aurora's extra capability should be justified by measured need, not assumed.
- Backups only count after a restore has been tested. HarborTech's 11-month-old restore test showed a 6.5-hour recovery that would have missed the 1-hour target.
- A single self-managed database creates single-person and single-server risk.
- Keeping the current design can be the right answer when migration risk outweighs the benefit. Here it does not, because the recovery gaps are serious.
- Documenting the decision before changing production protects the team and makes review possible.
- Migration cost is more than data movement. Connection strings, stored procedures, permissions, testing, and runbooks all take time.

---

## 11. Week 7 Vocabulary

| Term | Definition |
|---|---|
| **Relational database** | Stores data in tables with defined schemas and relationships, queried with SQL |
| **NoSQL database** | Non-relational database using models such as key-value or document |
| **Schema** | The defined structure of tables, columns, and relationships |
| **SQL** | Structured Query Language, used to query and manage relational data |
| **ACID transaction** | Atomic, Consistent, Isolated, Durable transaction guarantees |
| **Foreign key** | A column that links a row in one table to a row in another |
| **Stored procedure / trigger** | Database code stored and run inside the database engine |
| **Access pattern** | How an application reads and writes data; drives database design, especially in DynamoDB |
| **Amazon RDS** | Managed relational database service supporting multiple engines |
| **Amazon Aurora** | AWS-built, MySQL- and PostgreSQL-compatible relational database with a distributed storage layer |
| **Amazon DynamoDB** | Fully managed NoSQL key-value and document database |
| **Managed service** | A service where AWS operates the underlying infrastructure and, in part, the software layer |
| **Shared responsibility model** | Division of security and operational duties between AWS and the customer |
| **Multi-AZ** | Deployment across multiple Availability Zones for higher availability |
| **Read replica** | A copy of a database used to scale read traffic |
| **Failover** | Switching to a standby or replica when the primary fails |
| **RTO** | Recovery Time Objective: how quickly service must be restored |
| **RPO** | Recovery Point Objective: how much data loss is acceptable |
| **Point-in-time recovery** | Restoring a database to a specific moment within the retention period |
| **Snapshot** | A point-in-time backup of a database or volume |
| **AWS DMS** | Database Migration Service, which moves and replicates data between databases |
| **Change data capture (CDC)** | Ongoing replication of changes during a migration |
| **AWS SCT** | Schema Conversion Tool, which converts schemas between engines |
| **Parameter group** | A set of engine configuration values applied to an RDS instance |
| **Hypervisor** | Software layer that allows multiple virtual machines to share physical hardware |
| **Provisioned vs. on-demand capacity** | Pre-set capacity versus pay-per-request capacity (DynamoDB) |
| **Cutover** | The moment production traffic switches to the new database |
| **Rollback** | Reverting to the previous system if the cutover fails |

---

*Prepared for the HarborTech Operations Playbook, Week 7.*

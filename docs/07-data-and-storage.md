# 07 — Data & Storage

## Purpose

This document describes how DGBank stores, caches, and moves data. It covers the primary database, caching layer, object storage, and message queuing, and explains how these services are configured to meet banking-grade requirements for durability, security, and performance.

Service-level detail is in `services/data-services.md`.

---

## Data Principles

The DGBank data architecture follows four principles:

1. **Durability first** — data must survive the failure of a single Availability Zone.
2. **Encryption everywhere** — data is encrypted at rest and in transit without exception.
3. **Read/write separation** — transactional writes and read queries are handled by separate database endpoints to maintain performance.
4. **Decoupling** — components do not call each other synchronously where a queue can absorb the load instead.

---

## Primary Database — Amazon Aurora Serverless

Aurora Serverless is the primary relational database for DGBank. It stores all transactional banking data:

- Customer records (profile, KYC status)
- Account information (balances, account types, status)
- Transaction history (deposits, withdrawals, transfers)
- Deposit processing records
- Bank transfer records

### Writer and Reader Endpoints

Aurora provides separate endpoints for different workloads:

- **Writer endpoint:** Handles all INSERT, UPDATE, and DELETE operations. Used by ECS Fargate for transactional operations (money transfer, account update, batch reconciliation).
- **Reader endpoint:** Handles SELECT queries. Used by ECS Fargate for read-heavy operations (balance checks, transaction history, account summary).

Separating reads from writes prevents a heavy reporting query from blocking transaction processing.

### High Availability

**Current configuration:** Deployed across two Availability Zones (`1a`, `1c`).

**Pending action item:** Configure Aurora as a Multi-AZ cluster with an expanded footprint across three AZs (`1a`, `1c`, `1d`) to guarantee automatic failover with zero data loss in the event of an AZ outage. See `11-risks-and-todos.md`.

### Encryption

Data at rest is encrypted using customer-managed keys in AWS KMS. Automatic backups are also encrypted. Encryption cannot be disabled.

---

## Caching Layer — Amazon ElastiCache Serverless

ElastiCache (Redis-compatible) sits between ECS Fargate and Aurora to reduce database load and improve response times.

**What is cached:**
- Session tokens and authentication context
- Frequently accessed account summaries
- Reference data that changes infrequently (branch information, fee schedules)

**Why caching matters:**
Aurora is powerful but not instantaneous. Serving common reads from an in-memory cache reduces latency from tens of milliseconds to sub-millisecond, and it protects Aurora from traffic spikes.

**Cache invalidation:** Cache entries are invalidated on write. When a customer's balance is updated by a transaction, the corresponding cache entry is cleared so the next read goes back to Aurora.

---

## Object Storage — Amazon S3

Amazon S3 stores several categories of data, each in a separate bucket with its own access policy:

| Bucket Purpose | Contents | Access |
|---|---|---|
| **Frontend assets** | Static files served via CloudFront | Public via CloudFront only |
| **Document uploads** | Customer-submitted identity documents awaiting OCR | Private, accessed via ECS and Step Functions |
| **OCR results** | Extracted text and processed documents | Private, accessed by ECS and batch jobs |
| **Batch input** | CSV files consumed by nightly batch jobs | Private, accessed by batch workflows |
| **Batch output** | CSV files produced by batch jobs | Private, accessed by batch workflows and downstream systems |
| **Application logs** | Log archives from CloudWatch | Private, accessed by operations |

### Security and Access Control

- All S3 buckets **block public access** by default. The only public exposure is via CloudFront, which is restricted by origin access control.
- Access from private subnets is provided through a **VPC Endpoint** for S3, keeping traffic off the public internet.
- Amazon Macie scans the document upload buckets for PII to confirm appropriate protection.

### Lifecycle and Retention

Lifecycle policies are configured to:
- Transition infrequently accessed objects to cheaper storage classes after a defined period.
- Delete batch input and output files after their retention window.
- Retain audit-relevant objects for the period required by DGBank compliance.

---

## Messaging Layer — Amazon SQS

SQS decouples producers from consumers for asynchronous work.

**Usage in DGBank:**
- After an OCR result is stored, a message is queued for the downstream workflow that updates the customer record.
- During batch processing, messages are queued for downstream reconciliation tasks.
- Any async task that must not block the user-facing request path is routed through SQS.

**Why SQS:**
- Absorbs traffic spikes without dropping work
- Allows independent scaling of producers and consumers
- Provides built-in retry and dead-letter queue support

---

## Data Flow Overview

**Transactional write (money transfer):**
ECS Fargate → Aurora Writer → Anser BizSol → confirmation

**Read (balance check):**
ECS Fargate → ElastiCache (cache hit) OR Aurora Reader (cache miss)

**Document upload:**
S3 upload → Lambda → Step Functions → DG AI-OCR → S3 (results) → Aurora update → SQS (downstream)

**Batch processing:**
EventBridge → Step Functions → ECS Task → Aurora (read/write) → S3 (CSV) → SQS → downstream

---

## Backup and Recovery

- **Aurora automated backups:** Continuous, retained per DGBank retention policy, restorable to any point in time within the retention window.
- **S3 versioning:** Enabled on document and batch buckets to allow recovery from accidental deletion or overwrite.
- **Recovery procedures:** Documented in the operational runbooks and reviewed before go-live.

**Recovery targets (RTO/RPO):** To be confirmed with DGBank and documented in `11-risks-and-todos.md`.

---

## Pending Data Items

1. **Confirm three-AZ Aurora deployment** and Multi-AZ cluster configuration.
2. **Define RTO/RPO targets** and confirm backup retention policy.
3. **Confirm Macie scanning scope** for all document-related buckets.
4. **Document lifecycle policies** for each S3 bucket.
5. **Confirm ElastiCache invalidation strategy** for all cached entity types.

Full details in `11-risks-and-todos.md`.

---

## Summary

DGBank's data layer is built on Aurora Serverless for transactional data, ElastiCache for caching, S3 for object storage, and SQS for asynchronous work. Data is encrypted at rest and in transit, backed by continuous automated backups, and protected by VPC Endpoints and Macie. Read and write workloads are separated to preserve performance under load.

---
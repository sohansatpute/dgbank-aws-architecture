# Data & Storage Services

Services that store, cache, and queue the platform's data.

---

### Amazon Aurora Serverless

| Field | Description |
|---|---|
| **Category** | Data |
| **Service overview** | A MySQL- and PostgreSQL-compatible relational database that automatically scales compute capacity based on application demand, with separate Writer and Reader endpoints. |
| **Purpose in DGBank** | Stores all transactional banking data — customer records, account information, transaction history, deposit processing, and bank transfer records. The Writer node handles transactional writes; Reader nodes serve read queries. |
| **Position in architecture** | Primary data store. Accessed by ECS Fargate in all transactional and batch journeys. Planned to be configured as a Multi-AZ cluster. |

---

### Amazon ElastiCache Serverless

| Field | Description |
|---|---|
| **Category** | Data / Cache |
| **Service overview** | A fully managed in-memory caching service compatible with Redis, which automatically scales capacity based on workload. |
| **Purpose in DGBank** | Caches frequently accessed data such as session tokens and account summaries to reduce database load and improve response times for read-heavy journeys. |
| **Position in architecture** | Between ECS Fargate and Aurora. Consulted before the database in read-heavy paths such as balance checks. |

---

### Amazon S3 (Simple Storage Service)

| Field | Description |
|---|---|
| **Category** | Data / Storage |
| **Service overview** | A highly durable object storage service designed for storing and retrieving any amount of data from anywhere. |
| **Purpose in DGBank** | Stores multiple categories of data: static frontend assets served via CloudFront, uploaded customer documents awaiting OCR processing, processed OCR results, batch input and output CSVs, and application logs. |
| **Position in architecture** | Storage layer used across User, Business, and operations workflows. Accessed via VPC Endpoint from private subnets. |

---

### Amazon SQS (Simple Queue Service)

| Field | Description |
|---|---|
| **Category** | Data / Messaging |
| **Service overview** | A fully managed message queuing service that enables decoupling of application components through asynchronous communication. |
| **Purpose in DGBank** | Buffers asynchronous work such as post-OCR processing and downstream batch tasks, allowing producers and consumers to operate independently and absorb traffic spikes without data loss. |
| **Position in architecture** | Between producing services (ECS, Lambda, Step Functions) and consuming workers in document processing and batch workflows. |

---
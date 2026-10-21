# DGBank AWS Architecture — Full Walkthrough & Service Guide

## Executive Summary

This document provides a complete, end-to-end walkthrough of the DGBank Digital Banking Platform hosted on AWS (Tokyo Region). It is designed to give management, stakeholders, and new technical team members a holistic understanding of how the system works.

The architecture is built on three strict principles:
1. **Serverless & Managed Services:** Reducing operational overhead so the team can focus on banking features, not server maintenance.
2. **Strict Isolation:** The system is split into three separate environments (User, Business, and Future Tech Operations) to contain security risks.
3. **High Availability & Compliance:** Using multiple Availability Zones and advanced security tooling to meet banking regulations.

---

## Part 1: The End-to-End Request Flow

How does the system actually work in practice? Here is the journey of traffic through the architecture.

### Flow 1: A Customer Logs In & Checks Balance
1. **The Entry:** A customer opens the DGBank app. The request hits **Amazon Route 53** (DNS), which routes traffic to **Amazon CloudFront** (CDN) to ensure fast loading.
2. **The Security Gate:** Before entering, **AWS WAF** scans the request for malicious payloads (like SQL injection) and **AWS Shield Advanced** blocks any DDoS attack traffic.
3. **Authentication:** The user is redirected to **Amazon Cognito**. Cognito verifies their password and enforces mandatory Two-Factor Authentication (2FA) via SMS/Email. A secure token (JWT) is issued.
4. **API Routing:** The app sends the token to **Amazon API Gateway** (or **AWS AppSync** for GraphQL). API Gateway validates the token with Cognito.
5. **Compute:** API Gateway forwards the request to the **Internal Application Load Balancer (ALB)**, which distributes it to a healthy **Amazon ECS Fargate** container running the Backend API.
6. **Data Retrieval:** ECS Fargate checks **Amazon ElastiCache** (Redis) for the balance. If it's not cached, it queries **Amazon Aurora Serverless (Reader)**.
7. **The Response:** The data travels back through API Gateway to the customer.

### Flow 2: Internal Staff Operations & Batch Processing
1. **Staff Entry:** DGBank staff access the Business System through **AWS WAF** and a separate **Cognito User Pool**.
2. **Batch Triggering:** At night, **Amazon EventBridge** triggers a scheduled rule.
3. **Orchestration:** EventBridge starts an **AWS Step Functions** workflow.
4. **Processing:** Step Functions coordinates multiple tasks: it spins up an **Amazon ECS Fargate** batch task, runs small data-prep scripts via **AWS Lambda**, and uses **Amazon SQS** to queue asynchronous messages.
5. **Data & Storage:** The batch process reads/writes to **Aurora Serverless (Writer)** and stores generated CSV reports in **Amazon S3**.

### Flow 3: Document Upload & OCR (Account Opening)
1. **Upload:** A customer uploads an ID document. It is securely stored in an **Amazon S3** bucket (accessed privately via a **VPC Endpoint**).
2. **Trigger:** The S3 upload triggers a **Lambda** function, which starts a **Step Functions** workflow.
3. **OCR:** Step Functions calls the external **DG AI-OCR** service (via NAT Gateway) to extract text from the image.
4. **Completion:** The extracted data is saved back to S3 and written into **Aurora Serverless**.

### Flow 4: External Integration (Money Transfer)
1. **Transfer Request:** ECS Fargate receives a transfer request.
2. **Core Banking:** ECS Fargate makes an API call to **Anser BizSol** (the middleware connecting to Resona Bank). 
3. **Security:** The connection to Anser BizSol is encrypted, and the credentials are pulled from **AWS Secrets Manager**. Egress traffic goes through a **NAT Gateway** and is inspected by **AWS Network Firewall**.
4. **Notification:** Upon success, ECS Fargate triggers **Amazon SNS** and **Amazon SES** (via **HENNGE** and **Accrete**) to send an SMS and Email receipt to the customer.

### Flow 5: 24/7 Operations (Future Tech)
1. **Monitoring:** **Amazon CloudWatch** collects logs and metrics from all servers.
2. **Alerting:** If ECS Fargate crashes, CloudWatch raises an alarm. **EventBridge** captures this event.
3. **Notification:** EventBridge routes the alert through **AWS Transit Gateway** into the isolated Future Tech VPC, alerting the operator via **LINE**, Email, or Slack.
4. **Action:** The operator logs into the **AWS Management Console** (using secure cross-account IAM roles) to investigate using **CloudTrail** and **CloudWatch Logs**.

---

## Part 2: Comprehensive AWS Service Dictionary

This section defines every service used in the architecture, what it does, and why it is essential for DGBank.

### 🌐 Networking & Edge Services
*   **Amazon Route 53:** The internet's directory (DNS). Translates domain names to IP addresses. *Importance:* The starting point for all web traffic.
*   **Amazon CloudFront:** A Content Delivery Network (CDN). *Importance:* Caches static files closer to users, making the app load faster and shielding the backend from heavy traffic.
*   **AWS WAF (Web Application Firewall):** Inspects HTTP traffic. *Importance:* Blocks hackers trying to exploit web vulnerabilities before they reach the bank.
*   **AWS Shield Advanced:** DDoS protection. *Importance:* Keeps the bank online during coordinated cyberattacks.
*   **Amazon VPC (Virtual Private Cloud):** An isolated network in AWS. *Importance:* Keeps User, Business, and Ops systems completely separated.
*   **Public/Private Subnets:** *Importance:* Public subnets host load balancers facing the internet. Private subnets host databases and servers where hackers cannot reach them directly.
*   **AWS Transit Gateway:** A central router. *Importance:* Connects the separate VPCs securely without complex peering.
*   **NAT Gateway:** Outbound-only internet access. *Importance:* Lets servers call external APIs (like SMS providers) without exposing themselves to inbound attacks.
*   **Internet Gateway (IGW):** The bridge between the VPC and the public internet.
*   **VPC Endpoint:** A private tunnel to AWS services. *Importance:* Allows ECS to talk to S3 without using the public internet, saving costs and increasing security.
*   **Internal Application Load Balancer (ALB):** Distributes traffic across servers. *Importance:* Ensures high availability; if one server dies, traffic is routed to healthy ones.
*   **AWS Network Firewall (NWFW) / DNS Firewall:** Egress security. *Importance:* Controls outbound traffic, ensuring servers only talk to approved external destinations (like Resona Bank).
*   **VPC Flow Logs:** Records network traffic. *Importance:* A CCTV camera for the network, crucial for security audits.

### 🔐 Security & Compliance Services
*   **Amazon Cognito:** Identity and authentication. *Importance:* Handles customer and staff logins, enforcing mandatory 2FA.
*   **AWS IAM:** Access control. *Importance:* Grants least-privilege permissions to servers and operators.
*   **AWS Organizations:** Multi-account governance. *Importance:* Applies strict, unbreakable rules across all DGBank accounts.
*   **AWS KMS (Key Management Service):** Encryption key manager. *Importance:* Ensures all data stored in databases and S3 is encrypted to banking standards.
*   **AWS Secrets Manager:** Secure password vault. *Importance:* Keeps database credentials out of code.
*   **Amazon GuardDuty:** Intelligent threat detection. *Importance:* Uses machine learning to find malicious activity (like a server talking to a hacker IP).
*   **AWS Security Hub:** Central security dashboard. *Importance:* Aggregates all alerts so operators have one place to check system health.
*   **Amazon Inspector:** Vulnerability scanner. *Importance:* Scans Docker containers for known software bugs before deployment.
*   **Amazon Macie:** Sensitive data scanner. *Importance:* Uses ML to ensure customer PII is not accidentally exposed in S3.
*   **AWS Config:** Configuration tracker. *Importance:* Proves compliance by recording every change made to the infrastructure.
*   **AWS CloudTrail:** Audit logs. *Importance:* Records every API call (who did what, when) for regulatory compliance.
*   **AWS Certificate Manager (ACM):** Manages SSL/TLS certificates. *Importance:* Ensures secure HTTPS connections automatically renew.

### ⚙️ Compute & Application Services
*   **Amazon API Gateway:** The front door for APIs. *Importance:* Validates tokens, throttles traffic, and routes requests to backend servers.
*   **AWS AppSync:** Managed GraphQL API. *Importance:* Provides flexible data querying for specific frontend use cases.
*   **Amazon ECS Fargate:** Serverless container compute. *Importance:* Runs the core backend logic without needing to manage servers.
*   **AWS Lambda:** Event-driven serverless functions. *Importance:* Runs small tasks (like generating a thumbnail) without keeping a server running 24/7.
*   **AWS Step Functions:** Workflow orchestrator. *Importance:* Manages complex multi-step processes (like OCR or batch jobs) with built-in retries.
*   **Amazon ECR (Elastic Container Registry):** Docker image store. *Importance:* Holds the application code securely before ECS deploys it.
*   **GitHub Actions:** CI/CD pipeline. *Importance:* Automates code testing and deployment.

### 🗄️ Data & Storage Services
*   **Amazon Aurora Serverless:** The primary relational database. *Importance:* Stores banking data. Auto-scales capacity. Has a Writer (for transactions) and Reader (for queries) to prevent bottlenecks.
*   **Amazon ElastiCache Serverless:** In-memory cache (Redis). *Importance:* Speeds up the app by storing frequently accessed data in memory, reducing database load.
*   **Amazon S3:** Object storage. *Importance:* Stores static files, invoices, CSV batch files, and OCR documents.
*   **Amazon SQS:** Message queue. *Importance:* Decouples services so that traffic spikes don't crash the system.

### 📊 Monitoring & Operations
*   **Amazon CloudWatch:** Metrics and logging. *Importance:* The eyes of the system; triggers alarms when things go wrong.
*   **Amazon EventBridge:** Event router. *Importance:* Reacts to system changes (e.g., "database full") and schedules nightly batch jobs.
*   **Amazon SNS:** Notification service. *Importance:* Sends alerts to operators and triggers SMS messages.
*   **Amazon SES:** Email service. *Importance:* Sends transactional emails to customers.
*   **AWS Management Console:** The web UI. *Importance:* The portal used by Future Tech operators to monitor and investigate.

### 🔌 External Integrations (Vendors)
*   **Anser BizSol:** The critical middleware to Resona Bank. *Importance:* The ONLY path to move money in the core banking ledger.
*   **HENNGE:** Enterprise email provider. *Importance:* Ensures emails actually reach customers' inboxes.
*   **Accrete:** SMS provider. *Importance:* Delivers 2FA codes and transaction alerts.
*   **formrun:** Form builder. *Importance:* Collects data for account opening.
*   **DG AI-OCR:** Document reader. *Importance:* Automates reading ID cards.
*   **Google Drive:** Cloud storage. *Importance:* Staff file sharing.
*   **LINE:** Operations chat. *Importance:* Primary escalation channel for Future Tech.

---

## Part 3: Network Topology & Security Boundaries

The DGBank architecture uses a "Zero Trust" network model:
*   **User VPC:** Hosts CloudFront, WAF, API Gateway, and the customer-facing ECS cluster.
*   **Business VPC:** Hosts internal staff tools and batch processing.
*   **Future Tech VPC:** An isolated environment where operators sit. 
*   **Connectivity:** These VPCs are connected only via **AWS Transit Gateway**, with strict route tables. Operators in the Future Tech VPC can *read* CloudWatch metrics but *cannot* query the Aurora database directly.

---

## Part 4: Current Status & Pending Action Items

Based on the architecture review, the following items require attention before or shortly after launch:

1.  **High Availability (AZ Expansion):** The infrastructure must be expanded from 2 Availability Zones to 3 (`1a`, `1c`, `1d`), and **Aurora** must be configured as a Multi-AZ cluster to guarantee zero data loss during an AZ failure.
2.  **Egress Security Decision:** A final decision is needed between **AWS Network Firewall (NWFW)** and **Security Group + DNS Firewall**. NWFW is more robust but complex; DNSFW is simpler but may lack deep packet inspection. 
3.  **Cognito 2FA Operations:** Since SMS/Email 2FA is mandatory, the operational procedures for handling locked accounts or lost devices must be finalized with the business team.
4.  **IAM Permissions:** The exact least-privilege IAM roles required for the external API calls (Anser BizSol, HENNGE, Accrete, formrun, DG AI-OCR) must be finalized.
5.  **VPC Flow Logs:** Flow logs must be explicitly configured to monitor and alert on external API communication to ensure no unauthorized data exfiltration occurs.
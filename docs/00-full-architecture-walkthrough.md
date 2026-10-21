# DGBank AWS Architecture Review & Walkthrough

## 1. Introduction & Scope

This document provides a comprehensive architectural review of the DGBank Digital Banking Platform hosted on AWS. It is designed for stakeholders, management, and engineering teams to understand the system's design, security posture, and operational flows.

To provide a complete picture, this document is structured in two parts:
1.  **General Architecture Review:** A detailed narrative of how the system is structured, how traffic flows generally, and the role of every AWS service.
2.  **Scenario Walkthroughs:** Real-world examples tracing how specific actions (like a customer login or a nightly batch job) execute across the architecture.

---

## 2. Guiding Architectural Principles

The DGBank platform is not just a collection of services; it is built on four strict architectural principles:

*   **Serverless-First & Managed Services:** We rely on services like Amazon ECS Fargate and Amazon Aurora Serverless. This means AWS manages the underlying servers, patching, and scaling. Our team focuses on banking features, not infrastructure maintenance.
*   **Strict Environment Isolation:** The platform is physically separated into three distinct networks (VPCs): the **User System** (for customers), the **Business System** (for internal staff), and the **Future Tech Operations** (for 24/7 monitoring). A breach in one environment cannot easily compromise the others.
*   **Defence in Depth:** Security is layered. Traffic is inspected at the global edge, authenticated at the API layer, isolated within private networks, encrypted in the database, and continuously monitored for threats.
*   **Infrastructure as Code (CI/CD):** The entire environment is deployed via automated pipelines, ensuring no manual misconfigurations and a fully auditable change history.

---

## 3. General Architecture Review

To understand how the DGBank platform runs, we must trace how a request travels from the outside world into the core of the bank, how data is stored securely, and how the entire system is monitored and governed.

### 3.1 The Global Edge & Network Perimeter
When a user attempts to access DGBank, they do not connect directly to a server. They hit our global edge layer:
*   **Amazon Route 53** acts as the internet's directory (DNS), routing user requests to the correct endpoint.
*   **Amazon CloudFront** (a Content Delivery Network) caches static files at edge locations globally, ensuring fast load times and absorbing traffic spikes.
*   **AWS WAF (Web Application Firewall)** and **AWS Shield Advanced** act as the first line of defence, inspecting all incoming traffic to block malicious payloads (like SQL injection) and mitigating Distributed Denial of Service (DDoS) attacks.
*   **AWS Certificate Manager (ACM)** ensures all connections are securely encrypted via HTTPS by managing SSL/TLS certificates automatically.

### 3.2 The Isolated Network Topology
Once traffic passes the edge, it enters our Virtual Private Cloud (**Amazon VPC**). To ensure maximum security, we utilize **Public and Private Subnets**. 
*   Public subnets host internet-facing components like the **Internet Gateway (IGW)** and **NAT Gateway**. 
*   Private subnets host our sensitive compute and data resources, meaning they are completely hidden from the internet. The **NAT Gateway** allows these private resources to initiate outbound connections to the internet (for updates or external APIs) without allowing inbound connections.
*   To connect our three separate environments (User, Business, Ops), we use **AWS Transit Gateway**, which acts as a central, highly secure network hub.
*   For accessing AWS services like Amazon S3 privately without traversing the public internet, we use **VPC Endpoints**.
*   To strictly control outbound traffic leaving our network, we utilize **AWS Network Firewall (NWFW)** and **DNS Firewall (DNSFW)** as pending egress security measures.
*   Finally, **VPC Flow Logs** record all network traffic, providing an audit trail for security investigations.

### 3.3 Identity, Authentication & API Routing
Before a request reaches the application, it must be authenticated:
*   **Amazon Cognito** manages customer and staff identities. It verifies credentials and enforces mandatory 2-Factor Authentication (2FA) via SMS or Email.
*   Once authenticated, the user receives a secure token. They send this token to **Amazon API Gateway** (for REST APIs) or **AWS AppSync** (for GraphQL). These services validate the token and act as the front door, applying rate limits before routing the request to the backend.
*   **AWS IAM (Identity and Access Management)** ensures that these services, and the humans operating them, only have the strict minimum permissions needed to do their jobs.

### 3.4 The Frontend & Core Compute Engine
The user interfaces and the core business logic of the bank run here:
*   **AWS Amplify (SPA):** Hosts and serves the Single Page Application (SPA) frontend for the internal DGBank staff (Business System), providing a secure, scalable web interface.
*   An **Internal Application Load Balancer (ALB)** receives requests from API Gateway and distributes them evenly across multiple containers to prevent overload.
*   **Amazon ECS Fargate** is the core "brain." It runs our backend application in Docker containers, scaling automatically based on demand without us managing virtual machines.
*   **AWS Lambda** is used for smaller, event-driven tasks. The architecture utilizes specific Lambda functions (labeled **L5, L6, A10, A11, and V.W. Lambda**) to handle distinct microservices, such as CSV data processing, file handling, and data update APIs.
*   For complex, multi-step processes (like reading a document or running a batch job), we use **AWS Step Functions** to orchestrate the workflow and handle retries if a step fails.
*   The Docker images that run on ECS are built by **GitHub Actions** and securely stored in **Amazon ECR (Elastic Container Registry)**.

### 3.5 Data Persistence & Caching
The backend relies on a robust data layer:
*   **Amazon Aurora Serverless** is our primary relational database. It separates the "Writer" node (handling money transfers and updates) from the "Reader" node (handling balance checks), ensuring heavy reporting does not slow down transactions.
*   **Amazon ElastiCache** acts as an in-memory cache, storing frequently requested data to dramatically speed up response times and reduce the load on Aurora.
*   **Amazon S3** stores static assets, batch CSV files, and uploaded customer documents.
*   **Amazon SQS (Simple Queue Service)** is used to decouple services, holding messages (like "process this document") so that traffic spikes do not crash the backend.

### 3.6 Security, Governance & Compliance Wrapper
Wrapping around the entire architecture are the services that ensure banking-grade compliance and security:
*   **AWS KMS (Key Management Service)** manages the encryption keys used to protect data at rest in databases and S3.
*   **AWS Secrets Manager** securely stores and rotates database passwords and API keys, keeping them out of source code.
*   **Amazon GuardDuty** continuously monitors for malicious activity and compromised credentials.
*   **AWS Security Hub** aggregates all security alerts into a single dashboard.
*   **Amazon Inspector** automatically scans our container images for software vulnerabilities before deployment.
*   **Amazon Macie** uses machine learning to scan S3 buckets for accidentally exposed Personally Identifiable Information (PII).
*   **AWS Config** tracks every change made to the infrastructure, proving compliance to regulators.
*   **AWS CloudTrail** records every single API call made in the account.
*   **AWS Organizations** ties the separate environments together and applies unbreakable guardrails across all accounts.

### 3.7 Analytics, Operations & External Integrations
Finally, the system must be monitored 24/7, analyzed for insights, and communicate with the outside world:
*   **Amazon CloudWatch** collects logs and metrics from all services and triggers alarms.
*   **Amazon Kinesis Data Firehose** streams logs and telemetry data from CloudWatch into an Amazon S3 bucket.
*   **Amazon QuickSight** connects to this data pipeline to provide Business Intelligence (BI) dashboards and visual analytics for management and operations.
*   **Amazon EventBridge** routes system events (like alarms) and schedules nightly batch jobs.
*   **AWS Management Console** is the secure portal used by Future Tech operators to monitor the system.
*   For communications, we integrate with external providers: **Anser BizSol** (to connect to Resona Bank's core ledger), **HENNGE** (for Email), **Accrete** (for SMS), **formrun** (for account opening forms), **DG AI-OCR** (for document reading), **Google Drive** (for staff file sharing), and **LINE** (for operational escalations). These outbound calls are triggered by **Amazon SNS** and **Amazon SES**.

---

## 4. Scenario Walkthroughs: How the Architecture Runs

Now that we understand the structural design, let's trace exactly what happens during real-world operations.

### Scenario 1: The Customer Journey (Login, Balance & Transfer)

**The Goal:** An end-user opens the mobile app, logs in, checks their balance, and transfers money.

**Step 1: The Entry & Edge Security**
*   **Amazon Route 53** translates the DGBank domain name into an IP address, directing traffic to **Amazon CloudFront**.
*   **AWS WAF & AWS Shield** inspect the traffic for attacks. **AWS Certificate Manager (ACM)** ensures the connection is encrypted via HTTPS.

**Step 2: Identity & Authentication**
*   **Amazon Cognito** verifies the user's credentials and enforces **2FA** (sending a code via **Accrete** SMS or **HENNGE** Email). It issues a secure JSON Web Token (JWT).
*   **Amazon API Gateway** validates the token before allowing the request through.

**Step 3: Core Processing & Data Retrieval**
*   The **Internal ALB** routes the request to **Amazon ECS Fargate**.
*   ECS Fargate checks **Amazon ElastiCache** for the balance. If not found, it queries the **Amazon Aurora (Reader)** node. 
*   For the money transfer, ECS Fargate writes to the **Amazon Aurora (Writer)** node.

**Step 4: External Integration (The Money Movement)**
*   To move money, ECS Fargate uses credentials from **AWS Secrets Manager** to call **Anser BizSol** (connecting to Resona Bank). The outbound request passes through the **NAT Gateway** and is inspected by **Network Firewall (NWFW)**.
*   The transaction is securely stored, and data is encrypted using **AWS KMS**.

**Step 5: The Confirmation**
*   **Amazon SNS & SES** trigger an SMS (via Accrete) and Email (via HENNGE) receipt to the user. The app displays a success screen.

---

### Scenario 2: Account Opening & Document OCR

**The Goal:** A customer uploads a photo of their ID for KYC (Know Your Customer) verification.

**Step 1: Secure Upload**
*   The customer uploads the image via the API. It is stored in **Amazon S3** (accessed privately via a **VPC Endpoint**).

**Step 2: Orchestration & AI Processing**
*   The S3 upload triggers specific **AWS Lambda** functions (labeled **L5** and **L6**) to process the file. This starts an **AWS Step Functions** workflow.
*   Step Functions calls the external **DG AI-OCR** service to read the ID card. 

**Step 3: Data Storage & Queuing**
*   Step Functions updates the customer's profile in **Amazon Aurora** using the **V.W. Lambda** (Data Update API).
*   A message is placed in **Amazon SQS** to handle downstream processing asynchronously.

---

### Scenario 3: Internal Staff Portal & Nightly Batch Processing

**The Goal:** Staff manage operations via a web portal, and at 2:00 AM, the system reconciles daily transactions.

**Step 1: Staff Web Portal**
*   DGBank staff access the internal portal hosted on **AWS Amplify (SPA)**. 
*   They authenticate via **Amazon Cognito** and interact with backend systems via **API Gateway** and **AWS AppSync**.

**Step 2: The Batch Trigger**
*   **Amazon EventBridge** emits a scheduled event at 2:00 AM.
*   **AWS Step Functions** orchestrates the batch workflow, spinning up high-compute **Amazon ECS Fargate** tasks.
*   These tasks read and write CSV files in **Amazon S3** and ultimately transmit reconciliation data to **Anser BizSol**.

**Step 3: Developer CI/CD Deployment**
*   **GitHub Actions** builds the code and pushes the Docker image to **Amazon ECR**.
*   **Amazon Inspector** scans the image for vulnerabilities.
*   **Amazon ECS Fargate** pulls the secure image and deploys it.

---

### Scenario 4: 24/7 Operations (Future Tech's Role)

**The Goal:** Ensure the system is running smoothly and respond to incidents immediately.

**Step 1: Telemetry Collection & Analytics**
*   **Amazon CloudWatch** collects logs and metrics from all services. **VPC Flow Logs** capture network traffic data.
*   **Amazon Kinesis Data Firehose** streams these logs into **Amazon S3** for permanent storage.
*   **Amazon QuickSight** pulls data from this pipeline to generate visual dashboards for management reporting.

**Step 2: Alerting**
*   If an error occurs, CloudWatch triggers an alarm. **Amazon EventBridge** routes the alarm through **AWS Transit Gateway** into the isolated Future Tech environment.

**Step 3: Operator Response**
*   Operators receive the alert via **LINE**, Email, or Slack. They log into the **AWS Management Console** using **AWS IAM** roles.
*   They investigate using **CloudWatch Logs**, **AWS CloudTrail**, and **AWS Security Hub**, while **Amazon GuardDuty** and **Amazon Macie** continuously monitor the background for threats.

---

## 5. Pending Action Items & Architectural Recommendations

Based on our review of the architecture, the following items are critical for launch:

1.  **Expand to Three Availability Zones (AZs):** The network must be expanded to three AZs, and **Amazon Aurora** must be configured as a Multi-AZ cluster to guarantee zero data loss.
2.  **Egress Security Decision:** We need to finalize the choice between **AWS Network Firewall (NWFW)** and **Security Groups + DNS Firewall**. NWFW provides deep packet inspection but is complex to manage. DNSFW is simpler but only blocks domains.
3.  **IAM Permission Hardening:** We must ensure the IAM roles assigned to ECS tasks have the strict minimum permissions required to call Anser BizSol, HENNGE, Accrete, and DG AI-OCR.
4.  **Cognito 2FA Procedures:** Operational runbooks must define what to do if a customer loses their phone or cannot receive an OTP.

---

## 6. Summary

The DGBank architecture is a secure, modern, cloud-native platform. By utilizing **Amazon Cognito** for robust authentication, **AWS WAF and Shield** for edge protection, **Amazon ECS Fargate** for resilient compute, and **Amazon Aurora** for reliable data storage, the system is engineered to handle the demands of digital banking. The strict network isolation, combined with comprehensive monitoring and analytics via **Amazon CloudWatch, Amazon Kinesis, Amazon QuickSight, AWS CloudTrail, and AWS Security Hub**, ensures that security and compliance are maintained at every layer.
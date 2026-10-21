# DGBank AWS Architecture Review & Walkthrough

## 1. Introduction & Scope

This document provides a comprehensive architectural review of the DGBank Digital Banking Platform hosted on AWS. It is designed for stakeholders, management, and engineering teams to understand the system's design, security posture, and operational flows.

This document is divided into two main parts:
1.  **General Architecture Overview:** A structural breakdown of the environment and every AWS service used.
2.  **Scenario Walkthroughs:** Real-world examples of how a user, an operator, or a batch job moves through the architecture.

---

## 2. Guiding Architectural Principles

Before diving into the technical details, it is important to understand the core philosophy behind DGBank's architecture:

*   **Serverless-First & Managed Services:** We rely on services like ECS Fargate and Aurora Serverless. AWS handles the heavy lifting of server maintenance, patching, and scaling. Our team focuses on banking features, not infrastructure.
*   **Strict Environment Isolation:** The platform is split into three completely separate networks (VPCs): the **User System** (for customers), the **Business System** (for internal staff), and the **Future Tech Operations** (for 24/7 monitoring). If one is compromised, the others remain safe.
*   **Defence in Depth:** Security is layered. Traffic is inspected at the edge, authenticated at the API layer, encrypted in the database, and continuously monitored for threats. 
*   **Infrastructure as Code (CI/CD):** The entire environment is deployed via automated pipelines, ensuring no manual misconfigurations.

---

## 3. General Architecture Overview

To understand the whole system, we break it down into six functional layers.

### 3.1 Networking & Edge Layer
This layer handles how traffic enters the system, how the internal networks are isolated, and how external traffic is managed.
*   **Amazon Route 53:** The DNS (Domain Name System) service. It acts as the internet's directory, translating human-readable domain names into IP addresses.
*   **Amazon CloudFront:** A Content Delivery Network (CDN). It caches static content (images, web pages) at edge locations worldwide, ensuring fast load times for users in Japan.
*   **AWS WAF (Web Application Firewall):** Inspects all incoming web traffic. It blocks malicious payloads like SQL injection and cross-site scripting (XSS) before they reach the servers.
*   **AWS Shield Advanced:** Provides continuous DDoS (Distributed Denial of Service) protection. It protects the bank from coordinated cyberattacks aimed at taking the system offline.
*   **AWS Certificate Manager (ACM):** Handles the creation and automatic renewal of SSL/TLS certificates, ensuring all web traffic is encrypted via HTTPS.
*   **Amazon VPC (Virtual Private Cloud):** The isolated network boundary. DGBank uses three separate VPCs (User, Business, Operations) to ensure strict separation of resources.
*   **Public & Private Subnets:** Public subnets host internet-facing components (like Load Balancers). Private subnets host sensitive components (like databases and application servers) that should never be directly accessible from the internet.
*   **Internet Gateway (IGW):** The bridge that allows the public subnets to communicate with the internet.
*   **NAT Gateway:** Allows servers in the private subnets to initiate outbound internet traffic (e.g., to call an external API) while preventing the internet from initiating inbound connections to them.
*   **AWS Transit Gateway:** Acts as a central network hub. It securely connects the three different VPCs together, simplifying network routing.
*   **VPC Endpoint:** Provides a private, direct connection between the VPC and AWS services (like Amazon S3). Traffic stays within the AWS network and does not go over the public internet.
*   **AWS Network Firewall (NWFW):** Inspects and controls outbound (egress) traffic leaving the VPC, ensuring servers only talk to approved destinations.
*   **DNS Firewall:** Blocks outbound DNS queries to known malicious or unapproved domains, acting as a secondary egress control.
*   **VPC Flow Logs:** Captures IP traffic data across the network. It acts as a CCTV camera for network traffic, recording accepted and rejected connections for security audits.
*   **Internal Application Load Balancer (ALB):** Distributes incoming API traffic across multiple ECS containers, ensuring high availability and preventing any single server from being overwhelmed.

### 3.2 Security, Identity & Governance Layer
This layer ensures that only the right people and services can access the right resources, and that all actions are auditable.
*   **Amazon Cognito:** The identity provider. It handles user sign-up, sign-in, and mandatory 2-Factor Authentication (2FA) via SMS or Email.
*   **AWS IAM (Identity and Access Management):** Controls access to AWS resources. It ensures that services (like ECS) and operators have the strict minimum permissions required to do their jobs (Least Privilege).
*   **AWS Organizations:** Manages multiple AWS accounts centrally. It allows DGBank to apply unbreakable security policies across all accounts.
*   **AWS KMS (Key Management Service):** Creates and manages the encryption keys used to protect data at rest in databases and storage.
*   **AWS Secrets Manager:** A secure vault for passwords and API keys. It prevents hardcoding credentials in the source code.
*   **Amazon GuardDuty:** Uses machine learning to continuously monitor for malicious activity and unauthorized behavior (e.g., a server talking to a known hacker IP).
*   **AWS Security Hub:** Aggregates all security alerts from various AWS services into a single dashboard, providing a holistic view of the bank's security posture.
*   **Amazon Inspector:** Scans Docker container images for known software vulnerabilities (CVEs) before they are deployed.
*   **Amazon Macie:** Uses machine learning to scan S3 buckets for Personally Identifiable Information (PII) to ensure data is not accidentally exposed.
*   **AWS Config:** Records the configuration history of all resources. It proves compliance by tracking any changes made to the environment.
*   **AWS CloudTrail:** Records every API call made in the AWS account. It is the ultimate audit log, showing exactly who did what, and when.

### 3.3 Compute & Application Layer
This is the "brain" of the system, where business logic is executed and workflows are orchestrated.
*   **Amazon API Gateway:** The front door for REST APIs. It validates tokens, applies rate limiting, and routes requests to the backend.
*   **AWS AppSync:** A managed GraphQL API service, providing a flexible data querying alternative to API Gateway for specific frontend needs.
*   **Amazon ECS Fargate:** The core compute engine. It runs the backend application code in Docker containers without the need to manage underlying virtual machines.
*   **AWS Lambda:** Serverless compute for event-driven tasks. It runs lightweight code (e.g., processing a file upload) without provisioning servers.
*   **AWS Step Functions:** A workflow orchestrator. It coordinates complex, multi-step processes (like OCR processing or batch jobs) and handles retries automatically.
*   **Amazon ECR (Elastic Container Registry):** A secure registry to store the Docker images built by the developers.
*   **GitHub Actions:** The CI/CD pipeline that automates building, testing, and deploying application code to AWS.

### 3.4 Data & Storage Layer
Where the bank's most critical asset—data—is securely stored and managed.
*   **Amazon Aurora Serverless:** The primary relational database. It stores customer data and transactions. It features a "Writer" node (for transactions) and a "Reader" node (for queries) to optimize performance.
*   **Amazon ElastiCache:** An in-memory caching layer. It stores frequently accessed data to dramatically speed up response times and reduce database load.
*   **Amazon S3 (Simple Storage Service):** Object storage for static frontend files, uploaded customer documents (for OCR), and batch CSV files.
*   **Amazon SQS (Simple Queue Service):** A message queue that decouples services. It holds tasks (like sending an email or processing a document) so the system doesn't crash during traffic spikes.

### 3.5 Monitoring & Operations Layer
How Future Tech monitors the system 24/7 without direct access to customer data.
*   **Amazon CloudWatch:** The central telemetry service. It collects logs and metrics from all services and triggers alarms if something goes wrong.
*   **Amazon EventBridge:** An event router. It reacts to system state changes (e.g., an alarm triggering) and schedules automated nightly jobs.
*   **Amazon SNS (Simple Notification Service):** Sends outbound notifications (like SMS alerts to operators or customers).
*   **Amazon SES (Simple Email Service):** Sends outbound transactional emails.
*   **AWS Management Console:** The secure web portal used by Future Tech operators to investigate alerts and view system health.

### 3.6 External Integrations (Vendors)
Critical third-party services that DGBank relies on to deliver banking and communication features.
*   **Anser BizSol:** The middleware connecting DGBank to the Resona Bank Core Banking System. It is the only path to move money in the core ledger.
*   **HENNGE:** An enterprise email provider used for secure customer communications.
*   **Accrete:** An SMS provider used for 2FA codes and transaction alerts.
*   **formrun:** A form creation tool used for account opening workflows.
*   **DG AI-OCR:** An external service that reads and extracts text from uploaded customer documents (like ID cards).
*   **Google Drive:** Used by internal staff for secure file storage and sharing.
*   **LINE:** The primary communication channel between Future Tech operators and DGBank during incidents.

---

## 4. Scenario Walkthroughs: How the Architecture Runs

Now that we understand the components, let's trace exactly what happens during real-world operations.

### Scenario 1: The Customer Journey (Login, Balance & Transfer)

**The Goal:** An end-user opens the mobile app, logs in, checks their balance, and transfers money.

**Step 1: The Entry & Edge Security**
*   **Amazon Route 53 (DNS):** The user taps the app. Route 53 translates the DGBank domain name into an IP address, directing the traffic to the cloud.
*   **Amazon CloudFront (CDN):** Static files (like the app’s login screen and images) are served instantly from edge locations close to the user.
*   **AWS WAF & AWS Shield Advanced:** Before the request reaches our servers, WAF inspects it for malicious payloads (like SQL injection), while Shield actively blocks any DDoS attacks. 
*   **AWS Certificate Manager (ACM):** Automatically provisions and renews the SSL/TLS certificates for CloudFront, ensuring all traffic is securely encrypted via HTTPS.

**Step 2: Identity & Authentication**
*   **Amazon Cognito:** The user submits their credentials. Cognito handles the authentication. Because this is a bank, it enforces **mandatory 2FA** by sending an SMS (via Accrete) or Email (via HENNGE). Once verified, Cognito issues a secure JSON Web Token (JWT).
*   **Amazon API Gateway / AWS AppSync:** The app sends the token with every subsequent request. API Gateway (for REST APIs) and AppSync (for GraphQL) validate this token with Cognito. If the token is invalid, the request is rejected immediately.

**Step 3: Core Processing & Data Retrieval**
*   **Internal Application Load Balancer (ALB):** API Gateway routes the request to the ALB. The ALB distributes incoming traffic evenly to ensure no single server is overwhelmed.
*   **Amazon ECS Fargate (Backend API):** The ALB directs the request to the core "brain" of the bank—our containerized backend application running on ECS Fargate. This computes the business logic.
*   **Amazon ElastiCache:** Before querying the main database, ECS Fargate checks the Redis cache. If the user recently checked their balance, it is returned instantly.
*   **Amazon Aurora Serverless:** For transactions (like a money transfer), ECS Fargate writes to the **Aurora Writer** node. For read-only queries, it uses the **Aurora Reader** node. This separation prevents heavy reporting from slowing down real-time money transfers.

**Step 4: External Integration (The Money Movement)**
*   **Anser BizSol:** To actually move money, ECS Fargate securely calls Anser BizSol, the middleware connecting us to Resona Bank's core ledger. 
*   **AWS Secrets Manager:** The credentials to call Anser BizSol are never hardcoded; they are fetched securely from Secrets Manager at runtime.
*   **Amazon VPC & Subnets:** The application runs in Private Subnets, meaning it cannot be reached directly from the internet.
*   **NAT Gateway & Network Firewall (NWFW):** The outbound request leaves our private network through the NAT Gateway. It passes through the Network Firewall, which inspects egress traffic to ensure data is only going to approved external destinations.

**Step 5: The Confirmation**
*   **Amazon SNS & SES:** Once the transfer is successful, the system triggers a notification. Amazon SES sends an email receipt (via HENNGE), and SNS triggers an SMS alert (via Accrete). The user sees a success screen.

---

### Scenario 2: Account Opening & Document OCR

**The Goal:** A customer uploads a photo of their ID for KYC (Know Your Customer) verification.

**Step 1: Secure Upload**
*   **Amazon S3:** The customer uploads the image. The file is stored securely in an S3 bucket. 
*   **VPC Endpoint (S3):** Crucially, S3 is accessed via a VPC Endpoint, meaning the traffic stays entirely within the AWS private network and never traverses the public internet.

**Step 2: Orchestration & AI Processing**
*   **AWS Lambda & AWS Step Functions:** The upload to S3 triggers a Lambda function. Lambda kicks off a Step Functions workflow. Step Functions acts as the conductor, managing the multi-step process and handling retries if something fails.
*   **DG AI-OCR:** Step Functions securely calls the external DG AI-OCR service to read the text on the ID card. 

**Step 3: Data Storage & Queuing**
*   Once the OCR service returns the extracted text, Step Functions updates the customer's profile in **Aurora**.
*   **Amazon SQS (Simple Queue Service):** If downstream processing is required (like notifying the core banking system), a message is placed in the SQS queue. This decouples the services, ensuring that a spike in uploads doesn't crash the backend.

---

### Scenario 3: Nightly Batch Processing & CI/CD

**The Goal:** At 2:00 AM, the system reconciles daily transactions, while developers deploy new code during the day.

**Step 1: The Batch Trigger**
*   **Amazon EventBridge:** At exactly 2:00 AM, EventBridge emits a scheduled event, replacing traditional cron jobs.
*   **AWS Step Functions:** EventBridge triggers the batch workflow. 
*   **Amazon ECS Fargate:** Step Functions spins up dedicated, high-compute ECS Fargate tasks to process the data.
*   **Amazon S3:** The ECS tasks read CSV files from S3, process the data, and write the reconciled output CSVs back to S3.
*   **Anser BizSol:** At the end of the batch, the final reconciliation data is transmitted to Resona Bank.

**Step 2: Developer CI/CD Deployment**
*   **GitHub Actions:** Developers push code to GitHub, which triggers an automated build and test pipeline.
*   **Amazon ECR (Elastic Container Registry):** The built Docker image is stored securely in ECR.
*   **Amazon Inspector:** Automatically scans the ECR image for software vulnerabilities (CVEs) before deployment.
*   **Amazon ECS Fargate:** The new, secure image is pulled by ECS Fargate and rolled out to production.

---

### Scenario 4: 24/7 Operations (Future Tech's Role)

**The Goal:** Ensure the system is running smoothly and respond to incidents immediately.

**Step 1: Telemetry Collection**
*   **Amazon CloudWatch:** Every service (ECS, Lambda, Aurora, API Gateway) continuously sends logs and metrics to CloudWatch. 
*   **VPC Flow Logs:** Captures IP traffic data across the network, providing a record of all connections for security auditing.

**Step 2: Alerting**
*   If an error occurs (e.g., an ECS task crashes), CloudWatch triggers an alarm.
*   **Amazon EventBridge:** The alarm state change is routed through EventBridge.
*   **AWS Transit Gateway:** The alert travels securely across the Transit Gateway into the isolated **Future Tech VPC**.

**Step 3: Operator Response**
*   **AWS Management Console:** Future Tech operators in India and Daikanyama receive the alert via **LINE**, Email, or Slack. They log into the AWS Management Console using secure, read-only IAM roles.
*   They use **CloudWatch Logs** to investigate the issue and **AWS CloudTrail** to see exactly what API call caused the problem.

---

## 5. Deep Dive: Security, Governance & Compliance

While the scenarios explain the processing flow, the underlying security architecture ensures DGBank meets strict regulatory requirements.

*   **Amazon GuardDuty:** Constantly monitors CloudTrail, VPC Flow Logs, and DNS logs for malicious activity.
*   **AWS Security Hub:** Aggregates all alerts from GuardDuty, Inspector, and Macie into a single dashboard, scored against banking compliance standards.
*   **Amazon Macie:** Scans S3 buckets using Machine Learning to ensure that customer PII is not accidentally exposed.
*   **AWS Config:** Tracks every change made to the infrastructure. If someone changes a security group, AWS Config flags it.
*   **AWS CloudTrail:** The ultimate audit log. It records every single API call.
*   **AWS Organizations:** Ties the accounts together under one master account using unbreakable Service Control Policies (SCPs).
*   **AWS IAM (Identity and Access Management):** Controls who can do what using strict least-privilege roles.
*   **AWS KMS (Key Management Service):** Manages the encryption keys used to protect data at rest.
*   **DNS Firewall:** Acts as a secondary egress control, blocking the servers from resolving malicious domain names.

---

## 6. Pending Action Items & Architectural Recommendations

Based on our review of the architecture, the following items are critical for launch:

1.  **Expand to Three Availability Zones (AZs):** Currently, resources are deployed in two AZs. To guarantee zero data loss in a data center failure, the network must be expanded to three AZs, and **Aurora** must be configured as a Multi-AZ cluster.
2.  **Egress Security Decision:** We need to finalize the choice between **AWS Network Firewall (NWFW)** and **Security Groups + DNS Firewall**. NWFW provides deep packet inspection but is complex to manage. DNSFW is simpler but only blocks domains.
3.  **IAM Permission Hardening:** We must ensure the IAM roles assigned to ECS tasks have the strict minimum permissions required to call Anser BizSol, HENNGE, Accrete, and DG AI-OCR.
4.  **Cognito 2FA Procedures:** Since SMS/Email 2FA is mandatory, the operational runbooks for Future Tech must clearly define what to do if a customer loses their phone or cannot receive an OTP.

---

## 7. Summary

The DGBank architecture is a secure, modern, cloud-native platform. By utilizing **Amazon Cognito** for robust authentication, **AWS WAF and Shield** for edge protection, **ECS Fargate** for resilient compute, and **Aurora** for reliable data storage, the system is engineered to handle the demands of digital banking. The strict network isolation, combined with comprehensive monitoring via **CloudWatch, CloudTrail, and Security Hub**, ensures that security and compliance are maintained at every layer.
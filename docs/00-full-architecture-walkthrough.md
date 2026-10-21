# DGBank AWS Architecture Review & Walkthrough

## 1. Introduction & Scope

This document provides a comprehensive architectural review of the DGBank Digital Banking Platform hosted on AWS. It is designed for stakeholders, management, and engineering teams to understand the system's design, security posture, and operational flows.

Instead of simply listing services, this document walks through the system using **real-world scenarios**. By following the journey of a customer and an operator, we can clearly see how every AWS service works together to deliver a secure, highly available, and seamless banking experience.

---

## 2. Guiding Architectural Principles

Before diving into the flows, it is important to understand the core philosophy behind DGBank's architecture:

*   **Serverless-First & Managed Services:** We rely on services like ECS Fargate and Aurora Serverless. This means AWS handles the heavy lifting of server maintenance, patching, and scaling. Our team can focus on banking features, not infrastructure.
*   **Strict Environment Isolation:** The platform is split into three completely separate networks (VPCs): the **User System** (for customers), the **Business System** (for internal staff), and the **Future Tech Operations** (for 24/7 monitoring). If one is compromised, the others remain safe.
*   **Defence in Depth:** Security is layered. Traffic is inspected at the edge, authenticated at the API layer, encrypted in the database, and continuously monitored for threats. 
*   **Infrastructure as Code (CI/CD):** The entire environment is deployed via automated pipelines, ensuring no manual misconfigurations.

---

## 3. Scenario Walkthroughs: How the Architecture Runs

To understand the architecture, let's trace exactly what happens when the system is used.

### Scenario 1: The Customer Journey (Login, Balance & Transfer)

**The Goal:** An end-user opens the mobile app, logs in, checks their balance, and transfers money.

**Step 1: The Entry & Edge Security**
*   **Amazon Route 53 (DNS):** The user taps the app. Route 53 translates the DGBank domain name into an IP address, directing the traffic to the cloud.
*   **Amazon CloudFront (CDN):** Static files (like the app’s login screen and images) are served instantly from edge locations close to the user.
*   **AWS WAF & AWS Shield Advanced:** Before the request reaches our servers, WAF inspects it for malicious payloads (like SQL injection), while Shield actively blocks any DDoS (Distributed Denial of Service) attacks. 
*   **AWS Certificate Manager (ACM):** Automatically provisions and renews the SSL/TLS certificates for CloudFront, ensuring all traffic is securely encrypted via HTTPS.

**Step 2: Identity & Authentication**
*   **Amazon Cognito:** The user submits their credentials. Cognito handles the authentication. Because this is a bank, it enforces **mandatory 2FA** by sending an SMS (via Accrete) or Email (via HENNGE). Once verified, Cognito issues a secure JSON Web Token (JWT).
*   **Amazon API Gateway / AWS AppSync:** The app sends the token with every subsequent request. API Gateway (for REST APIs) and AppSync (for GraphQL) validate this token with Cognito. If the token is invalid, the request is rejected immediately.

**Step 3: Core Processing & Data Retrieval**
*   **Internal Application Load Balancer (ALB):** API Gateway routes the request to the ALB. The ALB distributes incoming traffic evenly to ensure no single server is overwhelmed.
*   **Amazon ECS Fargate (Backend API):** The ALB directs the request to the core "brain" of the bank—our containerized backend application running on ECS Fargate. This computes the business logic without us managing virtual machines.
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

### Scenario 3: Nightly Batch Processing & CI/CD (Internal Operations)

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

## 4. Deep Dive: Security, Governance & Compliance Architecture

While the scenarios explain the processing flow, the underlying security architecture ensures DGBank meets strict regulatory requirements.

*   **Amazon GuardDuty:** Constantly monitors CloudTrail, VPC Flow Logs, and DNS logs for malicious activity (e.g., a server trying to communicate with a known hacker IP).
*   **AWS Security Hub:** Aggregates all alerts from GuardDuty, Inspector, and Macie into a single dashboard, scored against banking compliance standards.
*   **Amazon Macie:** Scans S3 buckets using Machine Learning to ensure that customer PII (Personally Identifiable Information) is not accidentally exposed in the wrong bucket.
*   **AWS Config:** Tracks every change made to the infrastructure. If someone changes a security group to "allow all," AWS Config flags it and logs it.
*   **AWS CloudTrail:** The ultimate audit log. It records every single API call. If a database is deleted, CloudTrail tells us exactly *who* did it and *when*.
*   **AWS Organizations:** Ties the accounts together under one master account. We use Service Control Policies (SCPs) to apply unbreakable rules (e.g., "No one can disable CloudTrail").
*   **AWS IAM (Identity and Access Management):** Controls who can do what. Both internal staff and AWS services use strict least-privilege IAM roles.
*   **AWS KMS (Key Management Service):** Manages the encryption keys used to protect data at rest in Aurora and S3.
*   **DNS Firewall:** Acts as a secondary egress control, blocking the servers from resolving malicious or unapproved domain names.

---

## 5. Pending Action Items & Architectural Recommendations

Based on our review of the architecture, the following items are critical for launch:

1.  **Expand to Three Availability Zones (AZs):** Currently, resources are deployed in two AZs. To guarantee zero data loss in a data center failure, the network must be expanded to three AZs, and **Aurora** must be configured as a Multi-AZ cluster.
2.  **Egress Security Decision:** We need to finalize the choice between **AWS Network Firewall (NWFW)** and **Security Groups + DNS Firewall**. NWFW provides deep packet inspection but is complex to manage. DNSFW is simpler but only blocks domains.
3.  **IAM Permission Hardening:** We must ensure the IAM roles assigned to ECS tasks have the strict minimum permissions required to call Anser BizSol, HENNGE, Accrete, and DG AI-OCR.
4.  **Cognito 2FA Procedures:** Since SMS/Email 2FA is mandatory, the operational runbooks for Future Tech must clearly define what to do if a customer loses their phone or cannot receive an OTP.

---

## 6. Summary

The DGBank architecture is a secure, modern, cloud-native platform. By utilizing **Amazon Cognito** for robust authentication, **AWS WAF and Shield** for edge protection, **ECS Fargate** for resilient compute, and **Aurora** for reliable data storage, the system is engineered to handle the demands of digital banking. The strict network isolation, combined with comprehensive monitoring via **CloudWatch, CloudTrail, and Security Hub**, ensures that security and compliance are maintained at every layer.
# DGBank AWS Architecture Assessment & Flow Documentation

**Date:** [Insert Date]
**Prepared by:** [Your Name/Team]
**Audience:** DGBank & Future Tech Stakeholders
**Version:** 1.0

## 1. Executive Summary
We have conducted a thorough review of the proposed DGBank AWS Architecture (Tokyo Region). The architecture is a modern, cloud-native, serverless-first design built to ensure high availability, strict security, and seamless integration with Resona Bank's core systems. 

The system is deliberately isolated into three distinct environments (VPCs) to minimize risk:
1.  **User System:** For external customers.
2.  **Business System:** For internal DGBank staff.
3.  **Future Tech Operations:** For 24/7 monitoring and incident response.

This document outlines the end-to-end data flow and provides a comprehensive dictionary of every AWS service deployed, explaining its purpose and importance in the banking ecosystem.

---

## 2. End-to-End Architectural Flow

### Flow A: External Customer Journey (User System)
*How a customer accesses their account and performs transactions.*

1.  **DNS & Content Delivery:** A customer opens the DGBank app. The request hits **Amazon Route 53** (DNS routing) and is accelerated by **Amazon CloudFront** (global content delivery).
2.  **Security Inspection:** Before entering the system, traffic passes through **AWS WAF** (Web Application Firewall), which blocks malicious attacks (e.g., SQL injection).
3.  **Authentication:** The user logs in via **Amazon Cognito**, which verifies their identity. *(Note: 2FA via SMS/Email is mandatory here).*
4.  **API Routing:** The authenticated request is routed by **Amazon API Gateway** (REST) or **AWS AppSync** (GraphQL) to the backend.
5.  **Core Processing:** The request reaches **Amazon ECS Fargate** (Backend API) running in a private subnet. This is the "brain" of the application, handling business logic.
6.  **Data Retrieval:** ECS Fargate fetches data from **Amazon Aurora Serverless** (Writer/Reader nodes). For frequently accessed data, it checks **Amazon ElastiCache Serverless** (Redis cache) first to improve speed.
7.  **Document Processing (Optional):** If the user uploads a document, it is stored in **Amazon S3**. **AWS Lambda** (L5/L6) triggers **AWS Step Functions**, which calls **DG AI-OCR** to extract data.
8.  **Response:** The processed data is sent back to the user's device.

### Flow B: Internal Staff Journey & Batch Processing (Business System)
*How DGBank employees manage operations and how nightly batch jobs run.*

1.  **Staff Access:** DGBank staff access the system via **AWS WAF** and **Cognito**.
2.  **API Routing:** Requests are routed via **API Gateway** to the shared **ECS Fargate** backend.
3.  **Batch Processing:** Scheduled jobs (e.g., end-of-day reconciliation) are triggered by **Amazon EventBridge**. 
4.  **Workflow Orchestration:** EventBridge triggers **AWS Step Functions**, which coordinates **AWS Lambda** functions and **ECS Fargate Tasks** to:
    *   Pull data from **Aurora Serverless**.
    *   Process CSV files stored in **Amazon S3**.
    *   Send messages to **Amazon SQS** (Simple Queue Service) for asynchronous processing.
    *   Save the final output back to S3 (Input CSV storage/File linkage).
5.  **External Integration:** The Business System connects to **Anser BizSol** (Middleware for Resona Bank Core Banking), **HENNGE** (Email), **Accrete** (SMS), **formrun**, and **Google Drive** via NAT Gateways and VPC Endpoints.

### Flow C: Operations & Monitoring (Future Tech)
*How Future Tech ensures the bank is running 24/7.*

1.  **Telemetry Collection:** **Amazon CloudWatch** and **EventBridge** collect logs and metrics from all services (ECS, Aurora, Lambda, etc.).
2.  **Alerting:** If an error occurs (e.g., an ECS task fails), CloudWatch triggers an alert.
3.  **Notification:** The alert is sent to the **Future Tech VPC** via **AWS Transit Gateway**. 
4.  **Response:** Future Tech operators log into the **AWS Management Console** to investigate. 
5.  **Escalation:** If unresolved, operators follow the runbook and contact DGBank via **LINE**, Email, or Phone.
6.  **Audit:** Every action taken is logged by **AWS CloudTrail**.

### Flow D: Deployment (CI/CD)
*How code gets from developers to production.*

1.  **Source Control:** Developers push code to **GitHub**.
2.  **Build:** **GitHub Actions** builds the Docker image.
3.  **Registry:** The image is pushed to **Amazon ECR** (Elastic Container Registry).
4.  **Deploy:** ECS Fargate pulls the latest image from ECR and updates the running services.

---

## 3. Complete Service Dictionary (What & Why)

To ensure clarity for all readers (technical and non-technical), here is the role of every service in this architecture.

### 🌐 Networking & Edge
*   **Amazon Route 53:** The internet's phonebook. It translates human-readable domain names into IP addresses so users can find the bank.
*   **Amazon CloudFront:** A global content delivery network (CDN). It caches static content (images, scripts) closer to the user, making the app load faster.
*   **AWS WAF (Web Application Firewall):** The security gate. It inspects incoming traffic and blocks common cyberattacks like SQL injection and Cross-Site Scripting (XSS).
*   **AWS Shield Advanced:** Protects against Distributed Denial of Service (DDoS) attacks, ensuring the bank stays online even under malicious traffic floods.
*   **Amazon VPC (Virtual Private Cloud):** An isolated, private network within AWS where the bank's resources live.
*   **Public/Private Subnets:** Public subnets hold internet-facing resources (like Load Balancers). Private subnets hold sensitive resources (like Databases and App Servers) and cannot be accessed directly from the internet.
*   **AWS Transit Gateway:** A central hub that connects the User VPC, Business VPC, and Future Tech VPC together.
*   **NAT Gateway:** Allows private servers (like ECS) to reach out to the internet for updates without exposing themselves to inbound internet traffic.
*   **VPC Endpoint (S3):** A private, secure tunnel for ECS to talk to Amazon S3 without going over the public internet.
*   **VPC Flow Logs:** Records all IP traffic going to and from the network, used for security auditing and troubleshooting.

### 🔐 Identity & Security
*   **Amazon Cognito (User Pool):** The authentication system. It handles user sign-up, sign-in, and Multi-Factor Authentication (MFA/2FA via SMS/Email).
*   **AWS IAM (Identity and Access Management):** Governs who (developers, operators) can do what within the AWS accounts.
*   **AWS Organizations:** Manages multiple AWS accounts centrally to enforce security policies.
*   **AWS KMS (Key Management Service):** Manages the encryption keys used to secure bank data at rest.
*   **AWS Secrets Manager:** Securely stores and automatically rotates database passwords and API keys.
*   **Amazon GuardDuty:** Intelligent threat detection. It continuously monitors for malicious activity.
*   **AWS Security Hub:** A central dashboard that aggregates security alerts from GuardDuty, Inspector, and Macie.
*   **Amazon Inspector:** Automatically scans the application for software vulnerabilities.
*   **Amazon Macie:** Uses machine learning to discover and protect sensitive data (like customer PII) stored in S3.
*   **AWS Config:** Tracks changes to AWS resources over time, ensuring compliance with banking regulations.
*   **AWS CloudTrail:** Records every single API call made in the AWS account (Who did what, when, and from where).

### ⚙️ Compute & Application
*   **Amazon ECS Fargate:** The core "Backend API." It runs the banking application logic in containers without needing to manage servers. It scales automatically.
*   **AWS Lambda:** Serverless functions that run lightweight, event-driven tasks (e.g., processing an uploaded file).
*   **AWS Step Functions:** A workflow orchestrator. It coordinates complex, multi-step processes (like batch jobs or document OCR flows).
*   **Internal ALB (Application Load Balancer):** Distributes incoming API traffic evenly across the ECS Fargate containers.

### 🗄️ Data & Storage
*   **Amazon Aurora Serverless (Writer/Reader):** The primary relational database. It automatically scales up and down based on demand. The Writer handles transactions; the Reader handles queries.
*   **Amazon ElastiCache Serverless:** An in-memory cache (Redis). It stores frequently accessed data to reduce the load on Aurora and speed up response times.
*   **Amazon S3 (Simple Storage Service):** Used for hosting static frontend files, storing uploaded invoices/CSVs, and acting as a data lake.
*   **Amazon SQS (Simple Queue Service):** A message queue used to decouple services. For example, ECS can drop a message in SQS, and a Lambda function will pick it up and process it later.

### 📊 Management & Monitoring
*   **Amazon CloudWatch:** The eyes of the operation. It collects logs, metrics, and triggers alarms.
*   **Amazon EventBridge:** A serverless event bus. It reacts to events (e.g., "A file was uploaded to S3") and triggers targets (e.g., Step Functions).
*   **AWS Management Console:** The web interface Future Tech operators use to monitor and manage the infrastructure.
*   **Amazon ECR (Elastic Container Registry):** A private Docker registry where application images are stored before deployment.

### 🔌 External Integrations
*   **Anser BizSol:** The middleware connecting DGBank to Resona Bank's Core Banking System.
*   **HENNGE:** External email service provider.
*   **Accrete:** External SMS service provider.
*   **formrun:** External form creation and management tool.
*   **DG AI-OCR:** External Optical Character Recognition service for digitizing documents.
*   **Google Drive:** External file storage integration.
*   **Network Firewall (NWFW) / DNS Firewall (DNSFW):** Egress security controls to monitor and restrict outbound internet traffic.

---

## 4. Key Architectural Findings & Pending Action Items (TODOs)

Based on our review of the architecture diagram, we have identified the following items that require attention before or during launch:

1.  **High Availability (AZ Expansion):** Currently, resources are deployed in two Availability Zones (`1a`, `1c`). **Action:** Migrate to 3 AZs (`1a`, `1c`, `1d`) and configure Aurora as a Multi-AZ cluster to guarantee maximum uptime.
2.  **Egress Security Decision:** The team is deciding between **AWS Network Firewall (NWFW)** and **Security Groups + DNS Firewall (DNSFW)**. NWFW is more robust but complex to design. A decision is pending.
3.  **2FA Mandatory Requirement:** Cognito SMS/Email 2-Factor Authentication is a mandatory requirement. Operational procedures must be finalized to support this.
4.  **IAM Permissions:** Ensure IAM roles are properly configured to grant the necessary permissions for all external service integrations.
5.  **VPC Flow Logs:** Must be configured to monitor external API communication (AnserBizSol, HENNGE, Accrete, formrun, DG AI-OCR).

---
*End of Document*
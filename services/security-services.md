# Security & Identity Services

Services that authenticate users, protect data, control access, and detect threats across the DGBank platform.

---

### Amazon Cognito (User Pool)

| Field | Description |
|---|---|
| **Category** | Security / Identity |
| **Service overview** | A managed identity service that handles user registration, authentication, and access control for web and mobile applications. |
| **Purpose in DGBank** | Provides customer authentication with mandatory 2FA via SMS or email, issues JWTs for API authorization, and eliminates the need to build authentication infrastructure. |
| **Position in architecture** | Login journey. Every subsequent request carries a Cognito-issued JWT that API Gateway validates. |

---

### AWS IAM (Identity and Access Management)

| Field | Description |
|---|---|
| **Category** | Security / Identity |
| **Service overview** | A service that manages access to AWS resources by defining who can perform which actions on which resources. |
| **Purpose in DGBank** | Enforces least-privilege access for users, services, and automated processes across all AWS accounts. |
| **Position in architecture** | Underpins every API call, console login, and service-to-service interaction. |

---

### AWS Organizations

| Field | Description |
|---|---|
| **Category** | Security / Governance |
| **Service overview** | A service that centrally manages multiple AWS accounts, applies service control policies, and consolidates billing. |
| **Purpose in DGBank** | Enforces governance across the three-account structure (User, Business, Future Tech) and applies organization-wide guardrails. |
| **Position in architecture** | Governance layer spanning all accounts. |

---

### AWS KMS (Key Management Service)

| Field | Description |
|---|---|
| **Category** | Security / Encryption |
| **Service overview** | A managed service for creating and controlling encryption keys used to encrypt data across AWS services. |
| **Purpose in DGBank** | Provides auditable, regulatory-compliant encryption for data at rest in Aurora, S3, EBS, and Secrets Manager, with support for customer-managed key rotation. |
| **Position in architecture** | Encryption layer for all data stores and sensitive fields in transit. |

---

### AWS Secrets Manager

| Field | Description |
|---|---|
| **Category** | Security / Credentials |
| **Service overview** | A service that securely stores, retrieves, and rotates credentials such as database passwords and API keys. |
| **Purpose in DGBank** | Eliminates hard-coded credentials, provides automatic rotation for database credentials, and secures access to external API keys. |
| **Position in architecture** | Accessed by ECS Fargate at startup for database and external service credentials. |

---

### Amazon GuardDuty

| Field | Description |
|---|---|
| **Category** | Security / Threat Detection |
| **Service overview** | A threat detection service that continuously monitors for malicious activity using machine learning, anomaly detection, and threat intelligence. |
| **Purpose in DGBank** | Provides continuous threat detection across CloudTrail, VPC Flow Logs, and DNS logs. Alerts are routed to Security Hub for operator response. |
| **Position in architecture** | Passive monitoring layer with findings delivered to Future Tech operators. |

---

### AWS Security Hub

| Field | Description |
|---|---|
| **Category** | Security / Operations |
| **Service overview** | A centralized dashboard that aggregates, organizes, and prioritizes security findings from multiple AWS services and partner tools. |
| **Purpose in DGBank** | Provides a single-pane view of security posture, normalizes findings, and scores compliance against industry standards such as CIS and AWS Foundational Security Best Practices. |
| **Position in architecture** | Primary security console for Future Tech operators. |

---

### Amazon Inspector

| Field | Description |
|---|---|
| **Category** | Security / Vulnerability Management |
| **Service overview** | An automated vulnerability management service that scans EC2 instances, container images, and Lambda functions for software vulnerabilities. |
| **Purpose in DGBank** | Identifies CVEs in ECS container images before deployment and in running Lambda functions. Findings surface in Security Hub. |
| **Position in architecture** | Integrated with Amazon ECR to scan images on push. |

---

### Amazon Macie

| Field | Description |
|---|---|
| **Category** | Security / Data Protection |
| **Service overview** | A data security service that uses machine learning to discover and protect sensitive data stored in Amazon S3. |
| **Purpose in DGBank** | Detects personally identifiable information (PII) and sensitive customer data in S3 buckets, particularly uploaded documents and OCR results. |
| **Position in architecture** | Passive scanning of S3 buckets used for document processing. |

---

### AWS Config

| Field | Description |
|---|---|
| **Category** | Security / Compliance |
| **Service overview** | A service that records resource configurations and evaluates them against desired compliance rules. |
| **Purpose in DGBank** | Provides configuration history for regulatory audits and alerts on non-compliant changes to infrastructure. |
| **Position in architecture** | Passive monitoring across all accounts. |

---

### AWS CloudTrail

| Field | Description |
|---|---|
| **Category** | Security / Audit |
| **Service overview** | A service that records API activity across AWS accounts, capturing who performed what action, when, and from where. |
| **Purpose in DGBank** | Provides the authoritative audit log required for regulatory compliance and incident investigation. |
| **Position in architecture** | Behind every action in the platform. |

---

### AWS Certificate Manager (ACM)

| Field | Description |
|---|---|
| **Category** | Security / Encryption |
| **Service overview** | A service that provisions, manages, and renews TLS/SSL certificates for use with AWS services. |
| **Purpose in DGBank** | Enables HTTPS on public endpoints with automatic certificate renewal, eliminating certificate expiry risks. |
| **Position in architecture** | Attached to CloudFront distributions and internal Application Load Balancers. |

---
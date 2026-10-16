# Security & Identity Services

Services that authenticate users, protect data, control access, and detect threats across the DGBank platform.

This file covers the services that make DGBank defensible to regulators and safe for customer data. Each service plays a specific role — none of them is redundant.

---

### Amazon Cognito (User Pool)

| Field | Description |
|---|---|
| **Category** | Security / Identity |
| **Plain English** | The sign-in system. It stores user accounts, checks passwords, and issues a secure token the rest of the system trusts. |
| **Why DGBank uses it** | DGBank needs customer authentication that supports modern standards (OAuth 2.0, OIDC) without building authentication from scratch. Cognito handles password hashing, MFA, account recovery, and token issuance. |
| **Where it sits in the flow** | Login journey (Journey 1). Every subsequent request carries a Cognito-issued JWT that API Gateway validates. |
| **Real-world analogy** | The receptionist at a secure building. You show ID once, and you receive a visitor badge the rest of the building accepts. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Free tier for the first 50,000 monthly active users; charges beyond that. |
| **Failure impact** | If Cognito is unavailable, no user can log in. All authenticated journeys halt. Critical dependency. |
| **Key configuration notes** | **Mandatory 2FA via SMS or email** is a regulatory requirement. Cognito User Pool is used separately for the User System and Business System (different pools, different rules). |

---

### AWS IAM (Identity and Access Management)

| Field | Description |
|---|---|
| **Category** | Security / Identity |
| **Plain English** | Controls who can do what inside AWS. Every API call, console login, and automated action is governed by IAM permissions. |
| **Why DGBank uses it** | Principle of least privilege. Developers, operators, and services must only have the exact permissions they need — nothing more. IAM enforces this at every level. |
| **Where it sits in the flow** | Behind every action. Every service-to-service call, every operator login, every deployment uses an IAM role or user. |
| **Real-world analogy** | The key card system in an office. Different cards open different doors, and every swipe is logged. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Free. |
| **Failure impact** | Misconfigured IAM policies are one of the most common causes of "silent" outages — services fail because they cannot assume a required role. Tightly controlled IAM is essential. |
| **Key configuration notes** | A pending action item is to grant the exact IAM permissions required for each external service integration (Anser BizSol, HENNGE, Accrete, formrun, DG AI-OCR). |

---

### AWS Organizations

| Field | Description |
|---|---|
| **Category** | Security / Governance |
| **Plain English** | The umbrella that holds all DGBank AWS accounts together and applies company-wide rules. |
| **Why DGBank uses it** | Multi-account architecture (User, Business, Future Tech) needs central governance. Organizations lets DGBank apply Service Control Policies (SCPs) that prevent actions across all accounts — for example, "no account may disable CloudTrail." |
| **Where it sits in the flow** | Governance layer. Not on a transaction path, but constrains every account in the organization. |
| **Real-world analogy** | The parent company that sets the rules every subsidiary must follow. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Free. |
| **Failure impact** | If SCPs are too restrictive, they can block legitimate operations — this is a common misconfiguration risk. |
| **Key configuration notes** | Foundation for the three-account structure. SCPs should be reviewed before go-live. |

---

### AWS KMS (Key Management Service)

| Field | Description |
|---|---|
| **Category** | Security / Encryption |
| **Plain English** | Creates and manages the encryption keys used to protect DGBank's data at rest and in transit. |
| **Why DGBank uses it** | Banking regulations require that sensitive data be encrypted with controlled, auditable keys. KMS provides key rotation, access policies, and full audit trails of every key use. |
| **Where it sits in the flow** | Encrypts data in Aurora, S3, EBS volumes, and Secrets Manager. Called during the transfer journey for encrypting sensitive fields. |
| **Real-world analogy** | A bank vault's lock. The vault matters, but the lock and who holds the key matter more. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Per key per month, plus per 10,000 API requests. |
| **Failure impact** | If KMS keys are unavailable or deleted, encrypted data becomes permanently unreadable. Key deletion is deliberately difficult for this reason. |
| **Key configuration notes** | Customer-managed keys (CMKs) are preferred over AWS-managed keys for regulatory clarity. Key rotation policy must be documented. |

---

### AWS Secrets Manager

| Field | Description |
|---|---|
| **Category** | Security / Credentials |
| **Plain English** | A secure vault for passwords, API keys, and other secrets. Applications fetch them at runtime instead of storing them in code. |
| **Why DGBank uses it** | Hard-coded credentials are a critical security risk. Secrets Manager stores them centrally, rotates them automatically, and audits every access. |
| **Where it sits in the flow** | ECS Fargate retrieves database credentials and external API keys at startup and on rotation. Referenced in the transfer journey. |
| **Real-world analogy** | A locked safe where the office keeps the keys — not taped under the desk. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Per secret per month, plus per 10,000 API calls. |
| **Failure impact** | If Secrets Manager is unavailable, applications cannot retrieve credentials and will fail to connect to databases or external APIs. |
| **Key configuration notes** | Aurora credentials and Anser BizSol API keys should both be stored here. Rotation schedule to be defined with DGBank. |

---

### Amazon GuardDuty

| Field | Description |
|---|---|
| **Category** | Security / Threat Detection |
| **Plain English** | Continuously watches for suspicious activity — unusual logins, traffic to known-malicious IPs, compromised credentials — and raises alerts. |
| **Why DGBank uses it** | Banks face constant attempted intrusions. GuardDuty uses machine learning and threat intelligence to detect patterns humans would miss. |
| **Where it sits in the flow** | Passive monitoring. It watches CloudTrail, VPC Flow Logs, and DNS logs. It does not sit on a transaction path. |
| **Real-world analogy** | A security guard walking the building at night, looking for anything out of place. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Based on volume of events analysed. |
| **Failure impact** | No impact on traffic, but loss of threat visibility. Alerts feed into Security Hub for operator response. |
| **Key configuration notes** | GuardDuty findings should trigger CloudWatch alarms that route to Future Tech operators. |

---

### AWS Security Hub

| Field | Description |
|---|---|
| **Category** | Security / Operations |
| **Plain English** | A single dashboard that aggregates security alerts from GuardDuty, Inspector, Macie, Config, and external tools. |
| **Why DGBank uses it** | Without a central view, security findings are scattered across many services. Security Hub normalises findings and scores them against standards like CIS and AWS Foundational Security Best Practices. |
| **Where it sits in the flow** | Aggregation layer. Operators review Security Hub as their primary security console. |
| **Real-world analogy** | The main control room where all CCTV feeds and alarm systems converge. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Per finding ingested, plus per security check. |
| **Failure impact** | No direct impact on the platform, but operators lose the single-pane view. |
| **Key configuration notes** | Critical and high-severity findings should be routed to Future Tech's notification channel (LINE/email/Slack). |

---

### Amazon Inspector

| Field | Description |
|---|---|
| **Category** | Security / Vulnerability Management |
| **Plain English** | Automatically scans EC2 instances, container images in ECR, and Lambda functions for known software vulnerabilities. |
| **Why DGBank uses it** | Vulnerable dependencies are a common attack vector. Inspector continuously evaluates what is deployed and flags CVEs that require patching. |
| **Where it sits in the flow** | Runs against ECS container images and Lambda functions. Findings appear in Security Hub. |
| **Real-world analogy** | A health checkup for your software — it finds problems before symptoms appear. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Per resource scanned per month. |
| **Failure impact** | No impact on traffic, but loss of vulnerability visibility. |
| **Key configuration notes** | Should be enabled on ECR so that images are scanned on push, before deployment. |

---

### Amazon Macie

| Field | Description |
|---|---|
| **Category** | Security / Data Protection |
| **Plain English** | Uses machine learning to detect sensitive data — names, addresses, ID numbers, bank account details — stored in Amazon S3. |
| **Why DGBank uses it** | S3 buckets can accidentally be left with sensitive data. Macie finds PII and flags where it lives, so DGBank can apply the right protections. |
| **Where it sits in the flow** | Passively scans S3 buckets. Particularly relevant for the document upload journey. |
| **Real-world analogy** | A metal detector for data — it finds what should not be there. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Based on GB of data scanned. |
| **Failure impact** | No impact on traffic, but loss of data classification visibility. |
| **Key configuration notes** | Should scan the S3 buckets holding uploaded customer documents and processed OCR results. |

---

### AWS Config

| Field | Description |
|---|---|
| **Category** | Security / Compliance |
| **Plain English** | Records the configuration of every AWS resource over time and flags when something drifts from the approved baseline. |
| **Why DGBank uses it** | Regulatory audits require proof of what the environment looked like at any point in time. Config provides that history and can alert when a resource is changed in a non-compliant way. |
| **Where it sits in the flow** | Passive observer. Records resource state changes across all accounts. |
| **Real-world analogy** | A building inspector's logbook — every change to the structure is recorded, with time and reason. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Per configuration item recorded, plus per rule evaluation. |
| **Failure impact** | No impact on the platform, but gaps in compliance history. |
| **Key configuration notes** | Conformance packs aligned to banking regulation should be applied across the organization. |

---

### AWS CloudTrail

| Field | Description |
|---|---|
| **Category** | Security / Audit |
| **Plain English** | Records every API call made in the AWS account — who did what, when, and from where. |
| **Why DGBank uses it** | CloudTrail is the authoritative audit log. Regulators, security teams, and incident responders all depend on it. Without CloudTrail, incident investigation is impossible. |
| **Where it sits in the flow** | Behind every action in the platform. Every service call, console login, and CLI command is logged. |
| **Real-world analogy** | A tamper-proof diary that records every action taken in the building, with timestamp and signature. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Free for the first copy of management events; charges for additional trails and data events. |
| **Failure impact** | Loss of CloudTrail is a serious compliance issue. It should be protected by SCPs so no account can disable it. |
| **Key configuration notes** | Trails should deliver to a centralised S3 bucket in a dedicated log-archive account (a common pattern beyond the three-VPC structure). |

---

### AWS Certificate Manager (ACM)

| Field | Description |
|---|---|
| **Category** | Security / Encryption |
| **Plain English** | Provides and automatically renews the TLS certificates that enable HTTPS on DGBank's public endpoints. |
| **Why DGBank uses it** | Customers expect HTTPS. ACM handles issuance and renewal so certificates never expire unexpectedly — a common cause of embarrassing outages. |
| **Where it sits in the flow** | Attached to CloudFront distributions and internal ALBs. |
| **Real-world analogy** | The padlock icon on a browser — you rarely think about it, but its absence is alarming. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Free for public certificates used with AWS services. |
| **Failure impact** | An expired certificate breaks HTTPS for all users. ACM's automatic renewal prevents this — as long as DNS validation records are not removed. |
| **Key configuration notes** | Certificates are used on CloudFront (public) and internal ALBs (private). DGBank's DNS records must remain valid so ACM can renew. |

---

### AWS Shield Advanced (Security Context)

| Field | Description |
|---|---|
| **Category** | Security / Availability |
| **Plain English** | Enhanced protection against DDoS attacks, with cost protection and access to AWS's DDoS response team. |
| **Why DGBank uses it** | A DDoS attack against a bank is not just a technical incident — it is a reputational and regulatory one. Shield Advanced provides faster mitigation and financial protection against attack-driven traffic spikes. |
| **Where it sits in the flow** | Always-on protection for CloudFront, Route 53, and ALBs. Detailed in `services/networking-services.md`. |
| **Real-world analogy** | A dedicated emergency response team on standby, in addition to the standard alarm system. |
| **Managed or self-hosted** | Fully managed by AWS. |
| **Cost model** | Monthly subscription plus data transfer out. |
| **Failure impact** | Loss of enhanced mitigation and the DDoS response team during an attack. |
| **Key configuration notes** | Shield Advanced is a paid tier above the automatic (free) Shield Standard. |

---
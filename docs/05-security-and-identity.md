# 05 — Security & Identity

## Purpose

This document describes the security and identity architecture of the DGBank platform: how users and services are authenticated, how data is protected, how access is controlled, and how threats are detected and responded to.

It is written for security reviewers, auditors, and engineers. Service-level detail is in `services/security-services.md`.

---

## Security Principles

The DGBank security architecture is built on five principles:

1. **Defence in depth** — multiple, independent controls protect the same asset. No single control is trusted alone.
2. **Least privilege** — users and services receive only the permissions required to perform their function.
3. **Zero trust between environments** — the three environments (User, Business, Future Tech) do not trust each other by default. Access is explicit and controlled.
4. **Auditability** — every action, access, and configuration change is logged and retained.
5. **Automation over manual control** — security controls are enforced through code and policy, not manual process.

---

## Authentication and Identity

### Customer Authentication — Amazon Cognito (User Pool)

End customers authenticate via Cognito User Pool. Cognito handles:
- User registration and account recovery
- Password policy enforcement
- **Mandatory two-factor authentication via SMS or email**
- Issuance of JSON Web Tokens (JWTs) used to authorise subsequent API calls

Downstream services validate the JWT through API Gateway before processing any request. This means the application never handles raw credentials.

### Staff Authentication — Separate Cognito User Pool

DGBank internal staff (operations, sales, risk) authenticate against a separate Cognito User Pool with different policies, separate user records, and stricter session controls. Staff cannot log into the customer-facing application, and customers cannot access staff functions.

### Service-to-Service Identity — AWS IAM

Services authenticate to each other using IAM roles, not credentials. For example:
- ECS Fargate assumes a task role to call S3, Aurora, and external APIs.
- Lambda functions assume execution roles scoped to their function.
- Future Tech operators assume cross-account roles for read-only monitoring.

No long-lived access keys are used for service-to-service calls.

---

## Authorisation

### API Authorisation — API Gateway + Cognito

Every API call carries a JWT issued by Cognito. API Gateway validates the token signature, expiry, and audience before allowing the request to reach the backend. Requests with invalid or expired tokens are rejected at the edge.

### Role-Based Access — IAM Policies

IAM policies define what each role can do. Policies are written to grant the minimum permissions necessary. For example, an ECS task running the transfer workflow has permission to call Anser BizSol, but an ECS task running the login workflow does not.

### Governance Across Accounts — AWS Organizations

The three AWS accounts sit under a single AWS Organizations structure with Service Control Policies (SCPs) that apply to all accounts. SCPs enforce organisation-wide guardrails, such as:
- CloudTrail cannot be disabled.
- The root user cannot be used for day-to-day operations.
- Data cannot be replicated outside approved regions.

SCPs are one of the strongest available controls — they cannot be overridden, even by an account administrator.

---

## Data Protection

### Encryption at Rest — AWS KMS

Data at rest in Aurora, S3, EBS, and Secrets Manager is encrypted using customer-managed keys in AWS KMS. Key rotation is configured and the rotation history is auditable.

### Encryption in Transit — TLS

All traffic to public endpoints uses TLS 1.2 or higher, with certificates managed by AWS Certificate Manager. Internal service-to-service traffic within the VPC uses TLS where supported.

### Credential Management — AWS Secrets Manager

Database credentials and external API keys are stored in Secrets Manager and retrieved at runtime by ECS Fargate and Lambda. Automatic rotation is configured for database credentials. No credentials appear in source code, environment variables, or configuration files.

### Sensitive Data Discovery — Amazon Macie

Amazon Macie scans S3 buckets for personally identifiable information (PII) and other sensitive data. This is particularly relevant for uploaded customer documents and OCR results, and it allows DGBank to apply appropriate protections and retention policies.

---

## Threat Detection and Response

### Continuous Threat Detection — Amazon GuardDuty

GuardDuty monitors CloudTrail, VPC Flow Logs, and DNS logs for indicators of compromise — unusual API activity, communication with known-bad IPs, credential misuse, and similar patterns. Findings are generated automatically and routed to Security Hub.

### Centralised Security View — AWS Security Hub

Security Hub aggregates findings from GuardDuty, Inspector, Macie, Config, and partner tools, and scores the environment against industry standards such as CIS and AWS Foundational Security Best Practices. It is the primary console used by Future Tech operators for security oversight.

### Vulnerability Management — Amazon Inspector

Inspector scans ECS container images in Amazon ECR and running Lambda functions for known vulnerabilities (CVEs). Images are scanned on push to ECR, so vulnerabilities are caught before deployment.

### Compliance Monitoring — AWS Config

AWS Config continuously records resource configurations and evaluates them against rules that encode DGBank's compliance requirements. Non-compliant changes trigger alerts and are surfaced in Security Hub.

### Audit Logging — AWS CloudTrail

CloudTrail records every API call made in every account — including who made the call, when, from where, and what the result was. Trails deliver to a central S3 bucket for long-term retention and analysis. CloudTrail is protected by SCPs so it cannot be disabled.

### DDoS Protection — AWS Shield Advanced

Shield Advanced provides enhanced DDoS mitigation for CloudFront, Route 53, and Application Load Balancers, with access to the AWS DDoS Response Team and cost protection against attack-driven traffic spikes.

---

## Network-Layer Security

- **WAF rules** are attached to CloudFront distributions and API Gateway, blocking common web exploits before they reach the application.
- **Security Groups** restrict traffic at the resource level, following least-privilege principles.
- **Network ACLs** provide a secondary, subnet-level layer of defence.
- **VPC Flow Logs** capture all network traffic metadata for security investigation.
- **Egress inspection** (Network Firewall or DNS Firewall) is under evaluation to control and log outbound traffic. See `11-risks-and-todos.md`.

---

## Operational Security

### Separation of Duties

Three distinct roles are maintained:
- **Development** — writes and deploys application code, has no access to production data.
- **Operations (Future Tech)** — monitors and responds to incidents, has read-only access to production, no data access.
- **Administration** — manages AWS accounts and policies, no access to application data.

No single person holds more than one of these roles in the same environment.

### Incident Response

Security incidents follow the runbook process described in `runbooks/incident-response.md`. Alerts route to Future Tech, who follow documented procedures and escalate to DGBank using the escalation matrix in `runbooks/escalation-matrix.md`.

### Change Management

All infrastructure changes are made through IaC (Infrastructure as Code) and reviewed via pull request. Manual changes in the AWS console are restricted by IAM policy and logged by CloudTrail.

---

## Compliance Alignment

The security architecture is designed to align with:
- **AWS Well-Architected Framework — Security Pillar**
- **CIS AWS Foundations Benchmark**
- **AWS Foundational Security Best Practices**
- **Japanese banking regulations** (specific requirements to be mapped in a separate compliance document)

---

## Pending Security Items

1. **Finalise egress inspection decision** — Network Firewall vs. Security Group + DNS Firewall.
2. **Complete IAM permission review** for all external service integrations.
3. **Confirm operational procedures for mandatory 2FA.**
4. **Document key rotation policy** for KMS customer-managed keys.
5. **Confirm CloudTrail centralisation** and retention period with DGBank compliance.

Full details in `11-risks-and-todos.md`.

---

## Summary

Security at DGBank is layered, automated, and auditable. Identity is managed through Cognito for users and IAM for services. Data is encrypted at rest and in transit, with credentials secured in Secrets Manager. Threats are detected continuously by GuardDuty and surfaced in Security Hub for operator response. Every action is logged by CloudTrail. The design satisfies banking-grade requirements for segregation, encryption, and audit.

---
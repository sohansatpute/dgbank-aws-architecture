# 09 — External Integrations

## Purpose

This document describes how the DGBank platform integrates with external systems: partner banks, communication providers, document processing services, and operational tools. It explains how these integrations are secured, monitored, and controlled.

Service-level detail for each vendor is in `services/external-services.md`.

---

## Integration Principles

External integrations at DGBank follow four principles:

1. **Egress-only** — external systems initiate outbound connections from private subnets via NAT Gateway. No external system can initiate an inbound connection into the DGBank VPC.
2. **Explicit allow-listing** — only approved external endpoints are reachable. This is enforced by Network Firewall or DNS Firewall.
3. **Credential isolation** — API keys and secrets are stored in AWS Secrets Manager, never in application code.
4. **Full observability** — all outbound traffic to external endpoints is logged via VPC Flow Logs and reviewed by Future Tech.

---

## Resona Bank Core Banking — Anser BizSol

**What it is:** Anser BizSol is the middleware platform that connects DGBank to Resona Bank's core banking system.

**What it handles:**
- Account opening workflows
- Deposit processing
- Bank transfer processing
- Branch-level operations (DGBK Branch)
- Daily reconciliation reporting

**How it is called:**
ECS Fargate in the Business System makes synchronous HTTPS calls to Anser BizSol over NAT Gateway. The calls are authenticated with credentials stored in AWS Secrets Manager.

**Why it is critical:**
Anser BizSol is the **only path** between DGBank's AWS environment and Resona Bank's core ledger. If Anser BizSol is unavailable:
- Transfers cannot be recorded
- Account openings cannot be completed
- End-of-day reconciliation cannot run

There is no fallback path. Anser BizSol availability is a top-tier operational dependency.

---

## Customer Communication — HENNGE and Accrete

DGBank uses two external providers for customer communications:

### HENNGE (Email)
**Purpose:** Enterprise email delivery for transactional emails and operational notifications.
**Integration:** Called by Amazon SES. ECS Fargate and Lambda route outbound emails through SES, which delivers via HENNGE.
**Use cases:** Account notifications, transaction confirmations, password reset instructions.

### Accrete (SMS)
**Purpose:** SMS delivery for OTPs and urgent alerts.
**Integration:** Called by Amazon SNS. Cognito uses Accrete for SMS-based 2FA codes.
**Use cases:** Two-factor authentication codes, transaction alerts, account security notifications.

**Why external providers:**
DGBank does not maintain its own email or SMS infrastructure. Using specialised providers ensures deliverability, compliance with Japanese carrier regulations, and enterprise-grade filtering.

---

## Document Processing — DG AI-OCR

**What it is:** An AI-powered Optical Character Recognition service that extracts text and structured data from images.

**Integration flow:**
1. Customer uploads a document (e.g., ID card) in the DGBank app.
2. File is stored in an S3 upload bucket.
3. S3 event triggers a Lambda function.
4. Lambda starts a Step Functions workflow.
5. Step Functions calls DG AI-OCR with a signed URL or document payload.
6. DG AI-OCR returns extracted text and confidence scores.
7. Step Functions stores the result in S3 and updates the customer record in Aurora.

**Why external:**
OCR models require significant specialised ML expertise and continuous training. DG AI-OCR provides this without DGBank needing to maintain the models.

---

## Form Management — formrun

**What it is:** A form creation and data collection platform.

**Integration flow:**
formrun is used within DGBank's business workflows to collect structured data for account opening and other intake processes. Submitted data is routed to the Business System for processing.

**Position in architecture:** Used primarily by the Business System, feeding into Resona Bank workflows via Anser BizSol.

---

## File Storage — Google Drive

**What it is:** A cloud file storage and collaboration platform.

**Integration flow:**
Used by DGBank staff for document exchange and storage of non-transactional files. Access is restricted to authorised staff and governed by DGBank's data handling policies.

**Position in architecture:** Reached from the Business System via NAT Gateway.

---

## Operational Communication — LINE

**What it is:** A messaging platform used for real-time operational communication.

**Integration flow:**
LINE is the primary escalation channel between Future Tech operators and DGBank stakeholders during incidents.

**Position in architecture:** Operational communications only. Not on a transaction path. No customer data flows through LINE.

---

## Egress Control and Monitoring

All outbound traffic to external services is controlled and monitored:

- **Allow-listing:** Only specific domains and IP ranges are reachable. Unapproved destinations are blocked at the network layer.
- **Logging:** VPC Flow Logs capture every outbound connection to external endpoints for audit and troubleshooting.
- **Alerting:** Unusual patterns (e.g., attempts to reach unapproved destinations) trigger CloudWatch alarms reviewed by Future Tech.

**Pending decision:** The mechanism for egress inspection is under evaluation — AWS Network Firewall vs. Security Group + DNS Firewall. See `11-risks-and-todos.md`.

---

## Integration Resilience

External services can fail. DGBank's design accounts for this:

- **Timeouts and retries:** All external API calls are configured with timeouts and bounded retries via Step Functions or application logic.
- **Circuit breaking:** Where appropriate, applications stop calling a failing external service for a cool-down period to prevent cascading failures.
- **Graceful degradation:** If a non-critical external service (e.g., formrun) is unavailable, the rest of the platform continues to operate.
- **Critical path isolation:** Anser BizSol is the only critical-path external dependency. Its health is monitored separately with dedicated alarms and runbooks.

---

## Pending Integration Items

1. **Finalise egress inspection decision** (Network Firewall vs. DNS Firewall).
2. **Configure VPC Flow Logs** for external API monitoring.
3. **Complete IAM permission review** for each external integration.
4. **Document API rate limits** and quotas for each vendor.
5. **Confirm data processing agreements** are in place for each vendor (HENNGE, Accrete, DG AI-OCR, formrun, Google Drive).

Full details in `11-risks-and-todos.md`.

---

## Summary

DGBank integrates with external systems for banking operations (Anser BizSol), customer communications (HENNGE, Accrete), document processing (DG AI-OCR), forms (formrun), file storage (Google Drive), and operational communications (LINE). All integrations are egress-only, allow-listed, credentialed via Secrets Manager, and monitored via VPC Flow Logs. Anser BizSol is the only critical-path dependency, monitored with dedicated alarms and runbooks.

---
# 01 — Executive Summary

## What DGBank Is Building

DGBank is a digital banking platform hosted on AWS in the Tokyo region. It provides online banking services to end customers and internal tools to DGBank staff, while connecting to Resona Bank's core banking systems for account opening, deposits, transfers, and branch operations.

The system is operated 24 hours a day, 7 days a week, by Future Tech System Operations.

---

## The Three Environments

The platform is intentionally split into three separate, isolated environments. This is a deliberate security decision: if one environment is compromised, the others remain protected.

| Environment | Purpose | Users |
|---|---|---|
| **User System** | Customer-facing banking | End customers via mobile and web |
| **Business System** | Internal operations | DGBank staff (operations, sales, risk) |
| **Future Tech Operations** | 24/7 monitoring and incident response | Future Tech operators |

These environments are connected through a central network hub (Transit Gateway) but remain logically separated.

---

## Key Architectural Strengths

**1. Serverless-first design**
The platform uses managed, auto-scaling services (ECS Fargate, Aurora Serverless, Lambda, Step Functions). This reduces the operational burden on the team and allows the system to scale automatically with demand.

**2. Defence in depth**
Security is layered. Traffic passes through a Web Application Firewall, then authentication, then network isolation, then encryption, then continuous threat monitoring. No single control is relied upon alone.

**3. Clear separation of duties**
Development, operations, and business functions are separated across accounts and networks. Future Tech operates the system without needing access to customer data.

**4. Event-driven integration**
Integration with external partners (Resona Bank, email, SMS, OCR) is handled through dedicated workflows, keeping the core application decoupled from partner systems.

---

## How a Customer Request Travels (Summary)

When a customer performs an action in the DGBank app — for example, checking a balance — the request passes through the following layers:

1. **DNS resolution** — Route 53 directs the request
2. **Content delivery** — CloudFront serves cached content
3. **Security inspection** — AWS WAF filters malicious traffic
4. **Authentication** — Cognito verifies the customer's identity
5. **API routing** — API Gateway routes the request
6. **Application logic** — ECS Fargate processes the request
7. **Data retrieval** — Aurora Serverless provides the data
8. **Response** — The result travels back to the customer

A full step-by-step walkthrough is available in [03-request-journeys.md](03-request-journeys.md).

---

## Operational Model

Future Tech operators monitor the system continuously using CloudWatch and EventBridge. When an alert fires, the operator follows a documented runbook, and if the issue cannot be resolved, escalates to DGBank through the agreed contact channels (LINE, email, or phone).

---

## Open Items Requiring Attention

The following items are pending and must be resolved before or shortly after launch:

1. **Expansion to three Availability Zones** for maximum resilience
2. **Final decision on egress security** (Network Firewall vs. Security Group + DNS Firewall)
3. **Operational procedures for mandatory two-factor authentication**
4. **IAM permission review** for all external integrations
5. **VPC Flow Logs configuration** for external API monitoring

Full details are in [11-risks-and-todos.md](11-risks-and-todos.md).

---

## Bottom Line

DGBank is building a modern, secure, cloud-native banking platform on AWS. The architecture is well-aligned with AWS best practices for security, resilience, and operational efficiency. The remaining open items are known, tracked, and manageable.
# 02 — Architecture Overview

## Purpose

This document describes the overall shape of the DGBank platform: the three environments it is divided into, what lives inside each, and how they relate to one another. It is the bridge between the business context in `01-executive-summary.md` and the operational detail in `03-request-journeys.md`.

---

## The Three Environments

DGBank is not one system. It is three systems that share a common architecture but serve different users, hold different data, and operate under different security boundaries.

| Environment | Serves | Data Held | Access |
|---|---|---|---|
| **User System** | End customers via mobile and web | Customer profile, account data, transaction history | Public internet |
| **Business System** | DGBank internal staff | Operational data, batch results, reconciliation records | Internal, restricted |
| **Future Tech Operations** | Future Tech operators | Logs, metrics, alarms — no customer data | Restricted, isolated |

These environments are separated at the AWS account and VPC level. A breach in one does not grant access to another. This is a deliberate design choice that reduces blast radius and satisfies banking regulators who expect clear segregation of duties.

---

## Environment 1 — User System

**Purpose:** Deliver banking services to end customers.

**What lives here:**
- Public-facing edge services (Route 53, CloudFront, AWS WAF, Shield Advanced)
- Authentication (Amazon Cognito User Pool for customers)
- API layer (API Gateway, AppSync)
- Application compute (ECS Fargate, Lambda, Step Functions)
- Data stores (Aurora Serverless, ElastiCache Serverless, S3)
- Supporting services (KMS, Secrets Manager, CloudWatch)

**What does NOT live here:**
- Internal staff tools
- Operational monitoring dashboards
- Customer documents beyond what is required for active processing

For the full list of services and their roles, see `services/networking-services.md`, `services/security-services.md`, `services/compute-services.md`, and `services/data-services.md`.

---

## Environment 2 — Business System

**Purpose:** Support DGBank's internal operations — reconciliation, batch processing, account opening workflows, and staff-driven customer management.

**What lives here:**
- Separate Cognito User Pool for staff
- Business-facing APIs (API Gateway, AppSync)
- Batch orchestration (EventBridge, Step Functions, ECS Fargate tasks)
- Shared data services (Aurora, S3)
- Integration endpoints to Resona Bank via Anser BizSol

**Key characteristic:**
The Business System is largely driven by **scheduled events** rather than inbound user requests. Most of its work is batch processing that runs on a defined schedule (end-of-day reconciliation, report generation, file exchange) and workflows triggered by staff actions.

---

## Environment 3 — Future Tech Operations

**Purpose:** Provide 24/7 monitoring and incident response without exposing customer data or requiring direct access to production systems.

**What lives here:**
- Operator access to AWS Management Console (read-mostly)
- CloudWatch dashboards and EventBridge rule subscriptions
- Notification routing (email, LINE, Slack)
- No customer data, no application code, no database access

**Key characteristic:**
The Operations environment is intentionally **minimalist**. It receives telemetry and alerts from the other environments via Transit Gateway, and it can view operational state, but it cannot directly read customer data or modify application logic. This separation is a security feature — it means an operator account compromise does not become a data breach.

---

## How the Environments Connect

The three environments connect through **AWS Transit Gateway**, which acts as a central hub. Route tables on the Transit Gateway control which environments can reach which others.

- **User System ↔ Business System:** Selective peering for shared data services where required.
- **User System / Business System → Future Tech:** One-directional telemetry flow (CloudWatch metrics, EventBridge events).
- **Future Tech → User System / Business System:** Read-only monitoring access via the AWS Management Console with cross-account roles.

No environment has unrestricted access to another.

---

## Architectural Patterns Used

**Serverless-first**
Where possible, DGBank uses managed, auto-scaling services (ECS Fargate, Aurora Serverless, Lambda, Step Functions, ElastiCache Serverless) rather than provisioned infrastructure. This reduces operational burden and aligns cost with usage.

**Event-driven**
Asynchronous work — document processing, batch jobs, downstream notifications — is triggered by events (EventBridge rules, S3 uploads) and orchestrated through Step Functions. This decouples components and improves resilience.

**Defence in depth**
Security controls are layered. A request passes through DNS, CDN, WAF, authentication, network isolation, application validation, and encryption before reaching data. No single control is trusted alone.

**Multi-account isolation**
Environments are separated at the account level, not just the network level. This allows Service Control Policies (SCPs) to enforce guardrails that no single compromised credential can override.

**Least privilege**
IAM roles, Security Groups, and network ACLs are configured to grant only the permissions required. Access is periodically reviewed.

---

## What This Document Does Not Cover

- **How a specific request flows through the system:** see `03-request-journeys.md`.
- **Detailed network design:** see `04-network-foundation.md`.
- **Security service specifics:** see `05-security-and-identity.md`.
- **Individual service definitions:** see the `services/` directory.

---

## Summary

DGBank is a cloud-native banking platform built on three isolated AWS environments, connected through a controlled network hub, using serverless and event-driven patterns, and secured with layered controls. This structure satisfies banking-grade requirements for segregation, resilience, and auditability while remaining operationally manageable for a 24/7 outsourced operations team.

---
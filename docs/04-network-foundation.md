# 04 — Network Foundation

## Purpose

This document describes the network design of the DGBank platform: how the three environments are structured, how traffic enters and exits, and how the environments communicate with each other and with external systems.

It is written for engineers, operators, and security reviewers. Non-technical readers should start with `02-architecture-overview.md`.

---

## Region and Availability Zones

**Region:** `ap-northeast-1` (Tokyo)

The Tokyo region was chosen for data residency (Japanese banking data stays in Japan), latency to Japanese users, and integration proximity to Resona Bank's systems.

**Availability Zones:**
The platform is currently deployed across two AZs (`ap-northeast-1a` and `ap-northeast-1c`).

**Pending action item:** Expand to three AZs (`1a`, `1c`, `1d`) and configure Aurora as a Multi-AZ cluster to meet the target availability requirements. See `11-risks-and-todos.md`.

---

## VPC Layout

Three VPCs, one per environment:

| VPC | Purpose | Internet-facing |
|---|---|---|
| **User VPC** | Customer-facing services | Yes (via CloudFront and ALB) |
| **Business VPC** | Internal staff services and batch jobs | Restricted (no public endpoints) |
| **Future Tech VPC** | Operational monitoring | Restricted |

Each VPC has its own CIDR block and is isolated by default. Cross-VPC communication is only possible through Transit Gateway with explicit route table configuration.

---

## Subnet Design

Within each VPC, subnets are divided by function:

**Public subnets**
- Host internet-facing components such as NAT Gateways and Application Load Balancers.
- Have a route to the Internet Gateway.
- Deployed across at least two AZs.

**Private subnets**
- Host ECS Fargate tasks, Aurora instances, ElastiCache nodes, and Lambda functions.
- Do not have a route to the Internet Gateway.
- Have a route to NAT Gateway for outbound internet access only.
- Deployed across at least two AZs.

**Design principle:** A resource is only placed in a public subnet if it must be directly reachable from the internet. Everything else lives in a private subnet.

---

## Traffic Flow — Ingress (North-South)

Inbound traffic from end users follows this path:

1. **Amazon Route 53** resolves the domain to a CloudFront endpoint.
2. **Amazon CloudFront** serves cached content and forwards dynamic requests to the origin.
3. **AWS WAF** filters malicious requests at the edge.
4. Traffic enters the **User VPC** via the internet-facing endpoint.
5. **Internal ALB** distributes traffic to healthy ECS Fargate tasks in private subnets.

At no point does inbound traffic reach a database or application server directly.

---

## Traffic Flow — Egress (North-South)

Outbound traffic from private subnets to external services (Anser BizSol, HENNGE, Accrete, formrun, DG AI-OCR, Google Drive) follows this path:

1. Resource in a private subnet initiates an outbound connection.
2. Traffic is routed to **NAT Gateway** in a public subnet.
3. **NAT Gateway** forwards the request to the internet.
4. Responses return along the same path.

**Egress inspection:** Currently under evaluation — either AWS Network Firewall or Security Group + DNS Firewall. See `11-risks-and-todos.md`.

---

## Traffic Flow — East-West (Internal)

Traffic between the three environments is controlled by **AWS Transit Gateway**.

| Source | Destination | Purpose |
|---|---|---|
| Business VPC | User VPC | Shared data access where required |
| User VPC | Future Tech VPC | Telemetry, metrics, alerts |
| Business VPC | Future Tech VPC | Telemetry, metrics, alerts |
| Future Tech VPC | User VPC / Business VPC | Read-only monitoring via cross-account IAM roles |

Transit Gateway route tables enforce these paths. No environment has unrestricted east-west access.

---

## Private AWS Service Access

Private subnets access AWS services without traversing the public internet:

- **VPC Endpoint for S3:** Enables ECS Fargate and Lambda to read/write S3 without NAT Gateway charges or public exposure.
- **VPC Endpoints for other services:** ECR, Secrets Manager, KMS, and CloudWatch are typically accessed via interface endpoints to keep traffic off the public internet.

---

## Network Observability

**VPC Flow Logs** are enabled on all VPCs and delivered to CloudWatch Logs (or S3 for long-term retention). Flow Logs capture:
- Accepted and rejected connections
- Source and destination IP addresses and ports
- Bytes transferred

A pending action item is to configure Flow Logs specifically to monitor external API communication to Anser BizSol, HENNGE, Accrete, formrun, and DG AI-OCR.

---

## Security Group and NACL Design

- **Security Groups** are stateful, applied at the resource level, and follow least-privilege principles. Each service has its own security group.
- **Network ACLs** are stateless, applied at the subnet level, and provide a secondary layer of defence.

Both are configured to deny by default and permit only the specific traffic required for each service.

---

## DNS Design

- **Route 53** hosts public and private hosted zones.
- **Route 53 Resolver** handles DNS queries from within the VPCs.
- **Route 53 Resolver DNS Firewall** is being evaluated as part of the egress security decision.

---

## Pending Network Items

1. **Expand to three AZs** and configure Aurora as a Multi-AZ cluster.
2. **Finalise egress inspection** — Network Firewall vs. Security Group + DNS Firewall.
3. **Configure VPC Flow Logs** for external API monitoring.
4. **Review Transit Gateway route tables** before go-live to confirm least-privilege east-west access.

Full details in `11-risks-and-todos.md`.

---

## Summary

The DGBank network is designed around the principle of isolation by default. Inbound traffic is filtered at the edge, private resources are never directly reachable from the internet, outbound traffic is controlled and (when finalised) inspected, and east-west traffic between environments passes through a central controlled hub. This design supports the platform's banking-grade security and availability requirements.

---
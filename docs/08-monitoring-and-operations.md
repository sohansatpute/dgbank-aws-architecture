# 08 — Monitoring & Operations

## Purpose

This document describes how the DGBank platform is monitored, how alerts are routed, and how Future Tech operates the system 24/7 without direct access to customer data.

It is written for operations teams, incident responders, and stakeholders who need to understand the operational model. Incident procedures are in `runbooks/`.

---

## Operational Model

DGBank operates on a **24/7 monitoring and incident response** model delivered by Future Tech from India and Daikanyama.

**Operational responsibilities:**
- Continuous monitoring of all DGBank environments
- First-line response to alarms and alerts
- Execution of documented runbook procedures
- Escalation to DGBank engineering for issues beyond the runbook scope

**Out of scope for Future Tech:**
- Application code changes
- Infrastructure provisioning
- Customer data access
- Business decisions

The separation is deliberate. Future Tech operators can observe the health of the platform, but cannot access customer data or modify application logic.

---

## Telemetry Collection — Amazon CloudWatch

CloudWatch is the primary telemetry service. Every component in the platform publishes to it:

| Source | What CloudWatch receives |
|---|---|
| ECS Fargate | Task health, CPU/memory utilisation, request counts |
| Aurora Serverless | Connections, query latency, capacity utilisation |
| Lambda | Invocations, errors, duration, throttles |
| API Gateway | Request count, latency, 4XX/5XX rates |
| Step Functions | Execution success/failure, duration |
| ALB | Target health, request counts, latency |
| SQS | Queue depth, message age |
| VPC Flow Logs | Network traffic metadata |

CloudWatch Logs collects application logs, audit logs, and error output from all services. Log groups are organised by service and environment.

---

## Alerting — CloudWatch Alarms + Amazon EventBridge

CloudWatch Alarms trigger on defined thresholds:
- ECS Fargate healthy task count below target
- Aurora CPU or connection count above threshold
- Lambda error rate above threshold
- API Gateway 5XX rate above threshold
- SQS message age above threshold

When an alarm enters the ALARM state, it emits an event to EventBridge. EventBridge rules route the event to:

1. **Amazon SNS** for delivery to notification channels
2. **AWS Lambda** for custom formatting or enrichment
3. **Future Tech VPC** via **AWS Transit Gateway** for operator notification

---

## Notification Channels

Future Tech operators receive alerts via:
- **Email** (via Amazon SES / HENNGE)
- **LINE** (operational contact)
- **Slack** (internal ops channel)

The primary escalation channel and contact list is documented in `runbooks/escalation-matrix.md`.

---

## The Operations VPC

The Future Tech Operations environment is a separate AWS VPC connected to the DGBank environments via Transit Gateway.

**What operators can do from the Operations VPC:**
- Access the AWS Management Console with cross-account read-only roles
- View CloudWatch dashboards and logs
- Receive EventBridge alerts
- Access CloudTrail for investigation

**What operators cannot do:**
- Read customer data in Aurora or S3
- Modify application code or ECS configurations
- Access Cognito user records
- Bypass WAF or security controls

This design ensures that even if an operator account is compromised, no customer data is exposed.

---

## Incident Response Flow

When an alarm fires:

1. **Detection** — CloudWatch raises the alarm.
2. **Notification** — EventBridge routes the event; SNS and Lambda deliver it to Future Tech.
3. **Triage** — Operator acknowledges the alert and opens the relevant runbook.
4. **Investigation** — Operator uses AWS Management Console, CloudWatch Logs, and CloudTrail to diagnose.
5. **Resolution** — Operator follows the runbook procedure.
6. **Escalation** — If the issue is beyond the runbook, or requires DGBank engineering or business decisions, the operator escalates via the escalation matrix.
7. **Post-incident** — The operator records the incident and any actions taken for review.

Detailed procedures are in `runbooks/incident-response.md`.

---

## Operational Dashboards

CloudWatch dashboards provide at-a-glance views of platform health:
- **Customer-facing health** — API success rate, ECS task health, ALB response times
- **Data health** — Aurora connections, query latency, cache hit ratio
- **Batch health** — Step Functions execution status, SQS queue depth
- **Security health** — GuardDuty findings count, WAF blocked requests

Dashboards are organised so that an operator on shift can identify an incident within seconds of opening the console.

---

## Communication Between Future Tech and DGBank

Routine:
- Dashboards reviewed on schedule
- Weekly summary of health and incidents

Incident-related:
- Immediate notification for critical alarms
- Escalation via the escalation matrix
- Post-incident review within the agreed window

---

## Pending Operational Items

1. **Finalise runbooks** for each alarm type before go-live.
2. **Define escalation thresholds** and contact windows with DGBank.
3. **Confirm 24/7 staffing model** across India and Daikanyama.
4. **Complete knowledge transfer** for new operators.
5. **Establish dashboard standards** and review cadence.

Full details in `11-risks-and-todos.md`.

---

## Summary

DGBank operates on a 24/7 monitoring model delivered by Future Tech. CloudWatch and EventBridge collect telemetry and route alerts to operators through the Operations VPC. The Operations VPC is deliberately isolated: operators can observe system health and respond to incidents, but cannot access customer data or modify application logic. All incidents follow documented runbooks with defined escalation paths.

---
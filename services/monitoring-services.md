# Monitoring & Operations Services

Services that collect telemetry, trigger automated responses, and support 24/7 operational oversight of the DGBank platform.

---

### Amazon CloudWatch

| Field | Description |
|---|---|
| **Category** | Monitoring |
| **Service overview** | A monitoring and observability service that collects logs, metrics, and events from AWS resources and applications, and triggers alarms based on configurable thresholds. |
| **Purpose in DGBank** | Provides end-to-end visibility across ECS Fargate, Lambda, Aurora, API Gateway, and other services. Alarms trigger on unhealthy states such as ECS task failures, database capacity thresholds, or elevated error rates. |
| **Position in architecture** | Cross-cutting. Every service in the platform publishes logs and metrics to CloudWatch. Serves as the primary data source for Future Tech operations. |

---

### Amazon EventBridge

| Field | Description |
|---|---|
| **Category** | Monitoring / Integration |
| **Service overview** | A serverless event bus that routes events between AWS services, custom applications, and SaaS integrations based on rules. |
| **Purpose in DGBank** | Triggers scheduled batch jobs, reacts to state changes (such as CloudWatch alarm state transitions), and routes operational events to the Future Tech operations environment. |
| **Position in architecture** | Event orchestration layer. Feeds Step Functions for batch jobs and delivers operational alerts across the Transit Gateway to Future Tech. |

---

### Amazon SNS (Simple Notification Service)

| Field | Description |
|---|---|
| **Category** | Monitoring / Messaging |
| **Service overview** | A fully managed publish-subscribe messaging service that delivers notifications to subscribers via multiple channels including email, SMS, and HTTP endpoints. |
| **Purpose in DGBank** | Delivers operational alerts to Future Tech operators and provides SMS notifications to end users through integration with Accrete. |
| **Position in architecture** | Notification layer. Invoked by CloudWatch alarms for operational alerts and by application logic for customer notifications. |

---

### Amazon SES (Simple Email Service)

| Field | Description |
|---|---|
| **Category** | Monitoring / Communication |
| **Service overview** | A cloud-based email sending service designed for transactional and marketing email at scale. |
| **Purpose in DGBank** | Sends transactional emails such as transaction confirmations, account notifications, and 2FA codes in conjunction with Cognito. |
| **Position in architecture** | Communication layer used across customer-facing journeys. Integrated with HENNGE for enterprise email delivery. |

---

### AWS Management Console

| Field | Description |
|---|---|
| **Category** | Monitoring / Operations |
| **Service overview** | A web-based interface for managing and monitoring AWS services, resources, and account-level settings. |
| **Purpose in DGBank** | Primary interface used by Future Tech operators to investigate alerts, review logs, verify resource health, and take corrective action during incidents. |
| **Position in architecture** | Operational access layer. Used by Future Tech operators in the operations VPC. |

---

### AWS Transit Gateway (Operations Connectivity)

| Field | Description |
|---|---|
| **Category** | Monitoring / Networking |
| **Service overview** | The central network hub described in detail in `services/networking-services.md`. |
| **Purpose in DGBank** | Enables the Future Tech operations environment to receive alerts and query monitoring data across the DGBank VPCs without requiring direct access to customer data. |
| **Position in architecture** | Connectivity path between the DGBank and Future Tech environments. |

---
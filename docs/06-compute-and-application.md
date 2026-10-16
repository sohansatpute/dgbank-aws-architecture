# 06 — Compute & Application

## Purpose

This document describes the compute and application architecture of the DGBank platform: where application logic runs, how requests are routed, how long-running workflows are orchestrated, and how code is built and deployed.

Service-level detail is in `services/compute-services.md`.

---

## Compute Principles

The DGBank platform is built around three compute principles:

1. **Serverless-first** — managed, auto-scaling services are preferred over provisioned infrastructure.
2. **Containers for application logic** — the core Backend API runs in containers for consistency across environments.
3. **Functions for event handling** — lightweight, event-driven tasks run in Lambda rather than a long-running service.

This combination reduces operational overhead while allowing the platform to scale elastically with demand.

---

## The Application Layer

### Amazon ECS Fargate — The Backend API

ECS Fargate hosts the core business logic of the platform. It runs Docker containers without requiring management of the underlying EC2 instances.

**What runs in ECS Fargate:**
- Customer-facing API endpoints (login, balance, transfer, account summary)
- Business System API endpoints for internal staff
- Batch processing tasks triggered by Step Functions
- Async workers that consume messages from SQS

**Why Fargate:**
- Task-level isolation — each task runs in its own environment
- Automatic scaling based on CPU, memory, or request count
- No patching or capacity planning required
- Integration with ALB for health checks and load distribution
- IAM task roles for least-privilege access to AWS services

**Position in architecture:** Central compute layer between the ALB and the data layer.

### Application Load Balancer — Internal Only

The Internal ALB distributes requests to healthy ECS Fargate tasks. It is **internal** — it has no public endpoint. All inbound traffic reaches it only after passing through CloudFront, WAF, and API Gateway.

**Why internal:**
- Prevents any path to the application that bypasses WAF and authentication.
- Simplifies security group rules — the ALB only accepts traffic from the API Gateway layer.

### Amazon API Gateway — Public API Entry

API Gateway is the front door of the application layer. It:
- Terminates TLS
- Validates Cognito-issued JWTs
- Applies throttling and rate limits
- Routes requests to the Internal ALB or directly to Lambda

**Why API Gateway:**
- Centralised authentication enforcement
- Request/response transformation if needed
- Built-in logging and metrics via CloudWatch

### AWS AppSync — GraphQL Entry

AppSync provides a GraphQL API for use cases where a GraphQL interface is preferred. It integrates with Cognito for authorisation and with backend data sources for efficient data retrieval.

Both API Gateway and AppSync coexist — API Gateway serves REST-style endpoints, AppSync serves GraphQL queries and subscriptions.

---

## Event-Driven Compute

### AWS Lambda — Lightweight, Event-Driven Tasks

Lambda handles short-running, event-driven work:
- Preprocessing API requests before routing
- Responding to S3 upload events (triggering the OCR workflow)
- Formatting and routing CloudWatch alarms to Future Tech
- Small transformation steps inside Step Functions workflows

**Why Lambda for these tasks:**
- Zero cost when idle
- Scales automatically with event volume
- No server management

### AWS Step Functions — Workflow Orchestration

Step Functions coordinates multi-step workflows that involve multiple services, external calls, and waiting periods.

**Key workflows:**
1. **Document OCR pipeline** — S3 upload → Lambda trigger → OCR call → result storage → database update
2. **Nightly batch process** — scheduled EventBridge rule → batch workflow → ECS tasks → CSV output → core banking report
3. **Multi-step business workflows** — validation, fraud checks, external API calls, and completion

**Why Step Functions:**
- Built-in retries and error handling
- Full execution history for audit
- Visual representation of workflow state
- Coordinates ECS, Lambda, SQS, and external APIs in a single state machine

### Amazon EventBridge — Event Routing and Scheduling

EventBridge acts as the platform's event bus:
- **Scheduled rules** trigger nightly and periodic batch jobs
- **Event-driven rules** react to state changes (S3 uploads, alarm state transitions)
- **Cross-environment routing** delivers operational events to the Future Tech environment

---

## Request Routing — End to End

When a request reaches the platform:

1. **CloudFront** serves cached assets or forwards dynamic requests.
2. **AWS WAF** filters malicious traffic.
3. **API Gateway** or **AppSync** validates the JWT and applies throttling.
4. **Internal ALB** distributes the request to a healthy ECS Fargate task.
5. **ECS Fargate** processes the request, calling Aurora, ElastiCache, or external APIs as needed.
6. The response returns along the same path.

For event-driven work, the entry point is **S3** (uploads), **EventBridge** (schedules), or **SQS** (queued work), and processing is handled by **Lambda** or **Step Functions**.

---

## Container Build and Storage

### Amazon ECR — Container Registry

Docker images are built by GitHub Actions and pushed to Amazon ECR. ECS Fargate pulls images from ECR during deployment.

**Why ECR:**
- Private, IAM-controlled registry
- Integrated with Amazon Inspector for vulnerability scanning on push
- Image immutability and lifecycle policies for cost control

### GitHub Actions — CI/CD

Code is stored in GitHub. On push or pull request:
- GitHub Actions builds the Docker image
- Runs automated tests
- Pushes the image to ECR
- Triggers ECS deployment via a rolling or blue/green strategy

Detailed deployment design is covered in `10-cicd-and-deployment.md`.

---

## Scaling Behaviour

| Service | Scaling model |
|---|---|
| ECS Fargate | Task count scales on CPU, memory, or ALB request count target tracking |
| Lambda | Automatic per-invocation scaling |
| API Gateway | Managed, scales automatically |
| Step Functions | Managed, scales per workflow execution |
| EventBridge | Managed, no scaling concern |

No service requires manual capacity provisioning.

---

## Failure Handling

- **ECS Fargate tasks** are health-checked by the ALB. Unhealthy tasks are removed from rotation and replaced automatically.
- **Lambda functions** retry on failure per configured policy; async invocations can route to dead-letter queues.
- **Step Functions workflows** retry on transient failures and can route to catch states for handling.
- **API Gateway** returns standard HTTP error responses; client applications handle retries where appropriate.

Detailed runbook procedures are in `runbooks/incident-response.md`.

---

## Pending Compute Items

1. **Confirm ECS task sizing** for peak load scenarios.
2. **Define blue/green deployment strategy** for ECS releases.
3. **Confirm Step Functions retry and timeout policies** for each workflow.
4. **Document ECS task role permissions** per workflow.

Full details in `11-risks-and-todos.md`.

---

## Summary

The DGBank application layer runs on ECS Fargate for core business logic, Lambda for event-driven tasks, and Step Functions for orchestration. Requests enter through CloudFront and WAF, are authenticated by Cognito, validated by API Gateway, and routed through an internal ALB to the application. Container images are built by GitHub Actions and stored in ECR. The compute layer is entirely serverless or managed, scales automatically, and follows least-privilege access through IAM task roles.

---
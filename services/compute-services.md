# Compute & Application Services

Services that run application logic, orchestrate workflows, expose APIs, and build the deployment pipeline.

---

### Amazon ECS Fargate

| Field | Description |
|---|---|
| **Category** | Compute |
| **Service overview** | A serverless compute engine for containers that runs Docker images without requiring management of underlying EC2 instances. |
| **Purpose in DGBank** | Hosts the Backend API — the core business logic of the banking platform — with automatic scaling, task-level isolation, and integration with load balancers and service discovery. |
| **Position in architecture** | Central compute layer. Receives requests from the Internal ALB and communicates with Aurora, ElastiCache, and external APIs. |

---

### AWS Lambda

| Field | Description |
|---|---|
| **Category** | Compute |
| **Service overview** | A serverless compute service that runs code in response to events without provisioning or managing servers. |
| **Purpose in DGBank** | Handles lightweight, event-driven tasks such as S3 upload triggers, API request preprocessing, alert formatting, and short-running workflow steps. |
| **Position in architecture** | Distributed across the platform. Invoked by API Gateway, S3 events, EventBridge, and Step Functions. |

---

### AWS Step Functions

| Field | Description |
|---|---|
| **Category** | Compute / Orchestration |
| **Service overview** | A serverless orchestration service that coordinates multiple AWS services into a state machine, with built-in retries, error handling, and audit history. |
| **Purpose in DGBank** | Orchestrates long-running workflows such as document OCR processing, nightly batch jobs, and multi-step business processes that require coordination between Lambda, ECS tasks, and external services. |
| **Position in architecture** | Workflow layer. Triggered by EventBridge or API Gateway; invokes ECS Fargate tasks, Lambda functions, and external APIs. |

---

### Amazon API Gateway

| Field | Description |
|---|---|
| **Category** | Compute / Integration |
| **Service overview** | A fully managed service for creating, publishing, and securing REST, HTTP, and WebSocket APIs at scale. |
| **Purpose in DGBank** | Serves as the entry point for the Backend API. Validates Cognito-issued JWTs, applies throttling and rate limits, and routes requests to ECS Fargate. |
| **Position in architecture** | Between AWS WAF and the Internal ALB. Front door of the application layer. |

---

### AWS AppSync

| Field | Description |
|---|---|
| **Category** | Compute / Integration |
| **Service overview** | A fully managed GraphQL service that provides real-time data queries, mutations, and subscriptions with built-in authorization. |
| **Purpose in DGBank** | Provides GraphQL-based API access for specific client use cases, integrating with Cognito for authentication and with data sources for efficient data retrieval. |
| **Position in architecture** | Alternative API entry point alongside API Gateway. Routes to backend data sources. |

---

### Amazon ECR (Elastic Container Registry)

| Field | Description |
|---|---|
| **Category** | Compute / Deployment |
| **Service overview** | A fully managed Docker container registry that stores, manages, and deploys container images. |
| **Purpose in DGBank** | Stores the Docker images built by GitHub Actions. ECS Fargate pulls images from ECR during deployment. Integrated with Amazon Inspector for vulnerability scanning. |
| **Position in architecture** | Deployment pipeline. Sits between the CI build step and the ECS runtime. |

---

### GitHub Actions

| Field | Description |
|---|---|
| **Category** | Compute / CI-CD |
| **Service overview** | A continuous integration and continuous delivery platform that automates build, test, and deployment workflows triggered by repository events. |
| **Purpose in DGBank** | Builds Docker images from source code, runs automated tests, pushes images to Amazon ECR, and triggers deployment to ECS Fargate. |
| **Position in architecture** | External to AWS. Feeds the deployment pipeline at the top of the architecture. |

---
# 03 — Request Journeys

## Purpose

This document follows real user actions from start to finish. Each journey shows every service the request touches, in order, and explains what happens at each step.

If you read only one document in this repository, read this one.

---

## Journey 1: A Customer Logs Into the DGBank App

**Scenario:** A customer opens the DGBank mobile app and signs in.
**Trigger:** The customer taps "Login" and enters their credentials.

### Step-by-step

| # | Service | What Happens |
|---|---|---|
| 1 | **Amazon Route 53** | The app needs to reach `api.dgbank.jp`. Route 53 translates this name into the correct IP address. |
| 2 | **Amazon CloudFront** | The request is served from the nearest edge location. Static assets (app shell, images) are delivered from cache, making the login screen load quickly. |
| 3 | **AWS WAF** | Every incoming request is inspected. WAF checks for common attacks such as SQL injection or bot traffic. Malicious requests are blocked here, before they reach the application. |
| 4 | **Amazon Cognito (User Pool)** | Cognito receives the customer's credentials. It verifies the username and password against the user pool. |
| 5 | **Amazon Cognito — MFA** | Because two-factor authentication is mandatory, Cognito sends a one-time code via SMS or email. The customer enters the code. |
| 6 | **Amazon Cognito — Token Issuance** | On success, Cognito issues a signed JWT (JSON Web Token) containing the customer's identity and permissions. |
| 7 | **API Gateway** | The app includes the token in every subsequent request. API Gateway validates the token with Cognito before allowing the request through. |
| 8 | **Internal ALB** | The Application Load Balancer distributes the request to a healthy ECS Fargate task. |
| 9 | **Amazon ECS Fargate** | The backend API receives the request, verifies the user's permissions, and prepares a response. |
| 10 | **Amazon Aurora Serverless (Reader)** | ECS reads the customer's profile and account summary from the read replica. |
| 11 | **CloudWatch** | The entire login attempt is logged — successful or failed — for security auditing. |

### Outcome
The customer sees their account dashboard.

### Behind the scenes
- Every login attempt is logged in CloudWatch and CloudTrail.
- GuardDuty watches for suspicious login patterns (for example, logins from unusual locations).
- If MFA fails repeatedly, the account can be locked per policy.

---

## Journey 2: A Customer Checks Their Account Balance

**Scenario:** The customer taps "Account Balance" in the app.
**Trigger:** The app sends a GET request with the customer's token.

### Step-by-step

| # | Service | What Happens |
|---|---|---|
| 1 | **Route 53 → CloudFront → WAF** | Same entry path as the login journey. |
| 2 | **API Gateway** | Validates the JWT token. Rejects the request if the token is expired or invalid. |
| 3 | **Internal ALB** | Routes the request to ECS Fargate. |
| 4 | **Amazon ECS Fargate** | The backend service receives the request and determines what data is needed. |
| 5 | **Amazon ElastiCache Serverless (Redis)** | ECS checks the cache first. If the balance was recently retrieved, it is returned immediately — no database call needed. |
| 6 | **Amazon Aurora Serverless (Reader)** | On a cache miss, ECS queries the Aurora Reader node for the account balance. |
| 7 | **Response path** | The data travels back through the ALB, API Gateway, and CloudFront to the customer. |
| 8 | **CloudWatch** | The request duration and outcome are recorded. |

### Outcome
The customer sees their current balance.

### Why the cache matters
Aurora is powerful but relatively slow compared to an in-memory cache. By serving common queries from ElastiCache, the system responds faster and reduces load on the database — which matters during peak hours.

---

## Journey 3: A Customer Transfers Money

**Scenario:** The customer transfers funds to another account.
**Trigger:** The customer enters the recipient and amount, then confirms.

### Step-by-step

| # | Service | What Happens |
|---|---|---|
| 1 | **Route 53 → CloudFront → WAF** | Standard entry path. |
| 2 | **API Gateway** | Validates the token. |
| 3 | **Amazon ECS Fargate** | The backend begins the transfer workflow. Validates account status, checks balance, applies business rules. |
| 4 | **Amazon Aurora Serverless (Writer)** | The transaction is written to the Aurora Writer node inside a database transaction. |
| 5 | **Anser BizSol** | ECS calls Anser BizSol, the middleware connecting DGBank to Resona Bank's core banking system. The actual transfer is recorded in the core ledger. |
| 6 | **AWS KMS** | Sensitive fields are encrypted using keys managed by KMS. |
| 7 | **AWS Secrets Manager** | The credentials used to call Anser BizSol are retrieved securely from Secrets Manager — never hard-coded. |
| 8 | **Amazon SES / Amazon SNS (via HENNGE)** | A confirmation email is sent to the customer through HENNGE (external email service). |
| 9 | **CloudTrail** | The full API call is recorded for audit and regulatory compliance. |

### Outcome
The customer sees a confirmation. The money moves in the core banking ledger.

### Why this flow is critical
This is the most sensitive journey in the platform. Three controls are essential:
- **The Writer node** ensures the transaction is ACID-compliant.
- **Anser BizSol** is the only path to the core ledger — it is not bypassable.
- **KMS and Secrets Manager** ensure no credentials or sensitive data are exposed.

---

## Journey 4: A Customer Uploads an Identity Document

**Scenario:** During account opening, the customer uploads a photo of their ID.
**Trigger:** The customer selects a file in the app and taps "Upload."

### Step-by-step

| # | Service | What Happens |
|---|---|---|
| 1 | **Route 53 → CloudFront → WAF → API Gateway** | Standard entry path. |
| 2 | **Amazon S3 (Upload Bucket)** | The file is uploaded to a private S3 bucket. The bucket is not publicly accessible. |
| 3 | **AWS Lambda (L5/L6)** | The S3 upload triggers a Lambda function, which kicks off the processing workflow. |
| 4 | **AWS Step Functions** | Step Functions orchestrates the workflow: validate file → call OCR → wait for result → store result → update database. |
| 5 | **DG AI-OCR** | The document is sent to DG AI-OCR, an external service that extracts text and data from the image. |
| 6 | **Amazon S3 (Processed Bucket)** | The extracted data and a copy of the processed document are stored. |
| 7 | **Amazon Aurora Serverless** | The customer's record is updated with the extracted information. |
| 8 | **Amazon SQS** | If further downstream processing is needed, a message is queued. Consumers process it asynchronously. |
| 9 | **CloudWatch** | The entire workflow is logged. Failures trigger alerts. |

### Outcome
The customer sees "Document received." The account opening process continues in the background.

### Why Step Functions is used here
OCR is not instant — it can take seconds or minutes. Step Functions allows the system to wait, retry on failure, and coordinate multiple steps without blocking the customer's app. This is a classic long-running workflow.

---

## Journey 5: The Nightly Batch Process

**Scenario:** At the end of each business day, the system runs reconciliation and reporting jobs.
**Trigger:** A scheduled event fires at a fixed time.

### Step-by-step

| # | Service | What Happens |
|---|---|---|
| 1 | **Amazon EventBridge** | At the scheduled time, EventBridge emits a rule-triggered event. |
| 2 | **AWS Step Functions** | Step Functions starts the batch workflow. |
| 3 | **AWS Lambda** | Small preprocessing functions run — for example, verifying that source files are present. |
| 4 | **Amazon ECS Fargate (Task)** | A dedicated Fargate task runs the heavy batch job. |
| 5 | **Amazon Aurora Serverless (Writer)** | Transaction data is read and reconciled. |
| 6 | **Amazon S3 (Input/Output CSV)** | Input CSVs are read; results are written to output CSVs. |
| 7 | **Amazon SQS** | Downstream work is queued so it can be processed without blocking. |
| 8 | **Anser BizSol** | Final reconciliation is reported to the core banking system. |
| 9 | **CloudWatch & EventBridge** | Every step is logged. Failures trigger alerts to Future Tech. |

### Outcome
Daily reports are produced. Balances reconcile with the core banking system.

### Operational note
If the batch fails at any step, Step Functions retries according to policy. If it still fails, CloudWatch raises an alarm and Future Tech is paged. See [runbooks/incident-response.md](../runbooks/incident-response.md).

---

## Journey 6: An Alert Fires — Future Tech Responds

**Scenario:** At 2 AM, the ECS Fargate service in the User System drops below healthy task count.
**Trigger:** CloudWatch detects the unhealthy state.

### Step-by-step

| # | Service | What Happens |
|---|---|---|
| 1 | **Amazon CloudWatch** | Detects that healthy ECS tasks have fallen below the threshold. |
| 2 | **Amazon EventBridge** | The alarm state change is emitted as an event. |
| 3 | **AWS Lambda** | A small function formats the alert and sends it to the Future Tech environment. |
| 4 | **AWS Transit Gateway** | The alert traverses the network connection between the DGBank VPC and the Future Tech VPC. |
| 5 | **Future Tech notification channel (email / LINE / Slack)** | The operator on shift receives the alert. |
| 6 | **AWS Management Console** | The operator logs into the console to investigate. |
| 7 | **CloudWatch Logs** | The operator reviews logs to identify the cause. |
| 8 | **Runbook** | The operator follows the documented procedure for this alert type. |
| 9 | **Escalation** | If the issue cannot be resolved within the agreed window, the operator contacts DGBank via the escalation matrix. |
| 10 | **AWS CloudTrail** | Every action the operator takes is recorded. |

### Outcome
The service is either restored, or the issue is escalated. Either way, the event is logged and traceable.

### Why this journey matters for Future Tech
This is the core of the operations contract. The runbook must be written before go-live, and every operator must be trained on it.

---

## Summary Table: Services by Journey

| Service | Login | Balance | Transfer | Upload | Batch | Ops |
|---|---|---|---|---|---|---|
| Route 53 | ✔ | ✔ | ✔ | ✔ | | |
| CloudFront | ✔ | ✔ | ✔ | ✔ | | |
| AWS WAF | ✔ | ✔ | ✔ | ✔ | | |
| Amazon Cognito | ✔ | ✔ | ✔ | ✔ | | |
| API Gateway | ✔ | ✔ | ✔ | ✔ | | |
| Internal ALB | ✔ | ✔ | ✔ | ✔ | | |
| ECS Fargate | ✔ | ✔ | ✔ | ✔ | ✔ | |
| Aurora Serverless | ✔ | ✔ | ✔ | ✔ | ✔ | |
| ElastiCache | | ✔ | ✔ | | | |
| Anser BizSol | | | ✔ | | ✔ | |
| Amazon S3 | | | | ✔ | ✔ | |
| AWS Lambda | | | | ✔ | ✔ | ✔ |
| Step Functions | | | | ✔ | ✔ | |
| DG AI-OCR | | | | ✔ | | |
| Amazon SQS | | | | ✔ | ✔ | |
| EventBridge | | | | | ✔ | ✔ |
| CloudWatch | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| CloudTrail | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| KMS / Secrets Manager | | | ✔ | | | |
| Transit Gateway | | | | | | ✔ |
# External Integration Services

Third-party systems and partner services that DGBank integrates with to deliver banking functionality, communications, and document processing.

These services are external to AWS and are reached through NAT Gateway or Transit Gateway from the DGBank VPCs.

---

### Anser BizSol

| Field | Description |
|---|---|
| **Category** | External / Banking Integration |
| **Service overview** | A middleware platform that connects DGBank to Resona Bank's core banking system, handling account opening workflows, deposit processing, bank transfer processing, and branch-level operations. |
| **Purpose in DGBank** | Serves as the sole integration path between DGBank's AWS environment and Resona Bank's core banking systems. Ensures all financial transactions are recorded in the core ledger. |
| **Position in architecture** | Called by ECS Fargate in the money transfer and batch processing journeys. Sits between DGBank's application layer and Resona Bank's core systems. |

---

### HENNGE (Email)

| Field | Description |
|---|---|
| **Category** | External / Communication |
| **Service overview** | A cloud-based email security and delivery platform used for enterprise email communication. |
| **Purpose in DGBank** | Delivers transactional and operational email to end users and DGBank staff, with enterprise-grade security filtering and compliance controls. |
| **Position in architecture** | Reached from ECS Fargate via NAT Gateway. Integrated with Amazon SES for email delivery. |

---

### Accrete (SMS)

| Field | Description |
|---|---|
| **Category** | External / Communication |
| **Service overview** | An SMS delivery platform used for transactional and notification messaging. |
| **Purpose in DGBank** | Delivers SMS messages to end users, including one-time passwords for Cognito 2FA and transaction alerts. |
| **Position in architecture** | Reached from ECS Fargate via NAT Gateway. Integrated with Amazon SNS. |

---

### formrun

| Field | Description |
|---|---|
| **Category** | External / Business Process |
| **Service overview** | A form creation and management platform used for structured data collection and workflow initiation. |
| **Purpose in DGBank** | Supports account opening and other structured intake processes, feeding data into DGBank's business workflows. |
| **Position in architecture** | Called from the Business System. Feeds Resona Bank's account opening workflow via Anser BizSol. |

---

### DG AI-OCR

| Field | Description |
|---|---|
| **Category** | External / Document Processing |
| **Service overview** | An AI-powered Optical Character Recognition service that extracts structured text and data from uploaded document images. |
| **Purpose in DGBank** | Processes identity documents and other customer-submitted paperwork during account opening and verification. Extracted data feeds into Aurora via Step Functions workflows. |
| **Position in architecture** | Invoked by Step Functions in the document upload journey. Results are stored in S3 and written to the customer record in Aurora. |

---

### Google Drive

| Field | Description |
|---|---|
| **Category** | External / File Storage |
| **Service overview** | A cloud file storage and collaboration platform used for document exchange and storage. |
| **Purpose in DGBank** | Supports file sharing and document exchange between DGBank staff and approved external parties. Integrated with internal business workflows. |
| **Position in architecture** | Reached from the Business System via NAT Gateway. |

---

### LINE (Operational Contact)

| Field | Description |
|---|---|
| **Category** | External / Communication |
| **Service overview** | A messaging platform used for real-time operational communication between teams. |
| **Purpose in DGBank** | Provides the primary escalation and contact channel between Future Tech operators and DGBank stakeholders during incidents and routine operations. |
| **Position in architecture** | Operational communications channel. Not on a transaction path. |

---
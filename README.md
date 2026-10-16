# DGBank AWS Architecture Documentation

**Project:** DGBank Digital Banking Platform
**Region:** AWS Tokyo (`ap-northeast-1`)
**Maintained by:** Future Tech System Operations
**Status:** Living Document — Last Updated [YYYY-MM-DD]

---

## What This Repository Is

This repository is the single source of truth for the DGBank AWS architecture. It explains:

- How the system is structured
- What happens when a customer or staff member performs an action
- What every AWS service does and why it exists in this design
- How Future Tech monitors and operates the system 24/7

It is written for **both technical and non-technical readers**. No prior AWS knowledge is assumed.

---

## Who Should Read What

| If you are... | Start here |
|---|---|
| A manager, stakeholder, or new team member | [01-executive-summary.md](docs/01-executive-summary.md) |
| Trying to understand how a request flows | [03-request-journeys.md](docs/03-request-journeys.md) |
| A new engineer onboarding | [02-architecture-overview.md](docs/02-architecture-overview.md) then all `docs/` files in order |
| Looking for a specific service definition | [`services/`](services/) folder |
| An operator on shift | [`runbooks/`](runbooks/) folder |
| Reviewing risks and open items | [11-risks-and-todos.md](docs/11-risks-and-todos.md) |

---

## Repository Map

### `docs/` — The Narrative
Read in numbered order for a complete understanding.

| File | Covers |
|---|---|
| `00-reader-guide.md` | How to read this documentation and our writing conventions |
| `01-executive-summary.md` | Business context, system purpose, key strengths |
| `02-architecture-overview.md` | The three environments and their boundaries |
| `03-request-journeys.md` | Step-by-step walkthroughs of real user actions |
| `04-network-foundation.md` | VPCs, subnets, Transit Gateway, NAT, connectivity |
| `05-security-and-identity.md` | WAF, Cognito, IAM, KMS, GuardDuty, and the security stack |
| `06-compute-and-application.md` | ECS Fargate, Lambda, Step Functions, ALB |
| `07-data-and-storage.md` | Aurora, ElastiCache, S3, SQS |
| `08-monitoring-and-operations.md` | CloudWatch, EventBridge, and the Future Tech ops model |
| `09-external-integrations.md` | Anser BizSol, HENNGE, Accrete, formrun, DG AI-OCR, Google Drive |
| `10-cicd-and-deployment.md` | GitHub Actions, ECR, deployment pipeline |
| `11-risks-and-todos.md` | Pending action items and open decisions |

### `services/` — The Dictionary
One-line-per-service reference. Every service gets the same four-part explanation:
**What it is → Why DGBank uses it → Where it sits in the flow → Real-world analogy.**

### `diagrams/` — Visual References
All architecture diagrams in English, named by purpose.

### `runbooks/` — Operational Procedures
What to do when something breaks. Written for the Future Tech 24/7 shift team.

---

## Writing Conventions

Every service in the [services/](services/) dictionary follows the same fixed template. This keeps the reference consistent and predictable — a reader who learns to read one service entry can read all of them the same way.

The template is defined in [`services/_template.md`](services/_template.md). Do not edit the template without updating every existing service file.

Every service entry answers four core questions:

1. **What is it, in plain English?** — one sentence, no jargon
2. **Why does DGBank use it?** — the business, security, or operational reason
3. **Where does it sit in the flow?** — which request journey touches it
4. **What everyday thing is it like?** — a real-world analogy

Additional fields (cost model, failure impact, configuration notes) exist to help operators and reviewers, not to overwhelm casual readers.
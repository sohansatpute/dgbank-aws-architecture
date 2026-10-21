# 10 — CI/CD and Deployment

## Purpose

This document describes how code moves from a developer's workstation to running in production: the pipeline, the artefacts, the deployment process, and the safeguards that prevent faulty code from reaching customers.

This is a design-level document. Detailed pipeline configuration lives in the infrastructure repositories.

---

## Deployment Principles

1. **No manual production changes** — all deployments are automated, reproducible, and traceable.
2. **Immutable artefacts** — the container image deployed to production is the exact image that was built and tested. It is never rebuilt on a production machine.
3. **Vulnerability scanning before deployment** — container images are scanned by Amazon Inspector before they are eligible for deployment.
4. **Rollback-first design** — every deployment can be reverted to the previous known-good state.

---

## The Pipeline

The DGBank deployment pipeline consists of four stages:

### Stage 1 — Source Control (GitHub)

Application code is stored in GitHub. Branch protection rules require:
- A pull request for every change to `main`
- At least one peer review approval
- Passing automated tests before merge
- No direct commits to `main`

### Stage 2 — Build and Test (GitHub Actions)

On merge to `main`, or on a release tag, GitHub Actions runs:
- Unit tests
- Linting and static analysis
- Docker image build
- Push of the image to Amazon ECR

### Stage 3 — Image Scan (Amazon ECR + Amazon Inspector)

Once the image is in ECR, Amazon Inspector scans it for known vulnerabilities (CVEs).
- If the image contains vulnerabilities above the configured severity threshold, the pipeline halts.
- Approved images are tagged with the commit SHA and a semantic version for traceability.

### Stage 4 — Deployment (ECS Fargate)

Approved images are deployed to ECS Fargate:
- The ECS service pulls the new image from ECR.
- New tasks are started with the new image.
- The Internal ALB health-checks the new tasks.
- Once healthy, the ALB routes traffic to the new tasks and drains the old ones.
- Old tasks are terminated.

This is a **rolling deployment** by default. A **blue/green deployment** strategy (using CodeDeploy for ECS) can be enabled for higher-risk releases.

---

## Environment Strategy

DGBank maintains separate deployment paths for:
- **Development** — continuous deployment on merge to the development branch
- **Staging** — deployment after promotion from development, with all integrations configured but using test credentials
- **Production** — deployment after promotion from staging, with change approval

Promotion between environments is a deliberate, logged action, not an automatic cascade.

---

## What Deployable Units Exist

| Artefact | Deployed to | Deployment Method |
|---|---|---|
| Backend API container | ECS Fargate | Rolling (default) or Blue/Green |
| Lambda functions | AWS Lambda | Code update via GitHub Actions |
| Step Function definitions | AWS Step Functions | IaC deployment (CDK or CloudFormation) |
| Infrastructure (VPCs, IAM, networking) | AWS accounts | IaC via pipeline |

Application code and infrastructure code are deployed through the same pipeline with the same safeguards.

---

## Rollback

Every deployment has a rollback path:

- **ECS Fargate:** The previous task definition is retained. Rolling back re-deploys the previous image.
- **Lambda:** The previous version is retained and can be re-aliased.
- **Infrastructure:** All infrastructure is managed as code. Rollback means reverting the IaC commit and applying it.
- **Database migrations:** Database schema changes are versioned and designed to be backward-compatible. Destructive migrations require explicit approval and manual verification.

Rollback is a first-class operation, not an afterthought.

---

## Change Traceability

Every change to production is traceable:

- The Git commit SHA of the source code
- The ECR image digest
- The ECS task definition revision
- The IAM role that deployed the change
- The timestamp and approval record
- The CloudTrail entry for the deployment API call

An auditor can reconstruct exactly what was deployed, when, by whom, and from what code.

---

## Secrets in the Pipeline

- The pipeline authenticates to AWS using **OpenID Connect (OIDC)** with GitHub Actions, not long-lived access keys.
- Secrets used by the application at runtime are stored in AWS Secrets Manager. The pipeline does not handle runtime secrets.
- Deployment permissions are scoped to the minimum required for each environment.

---

## Access Control

- Only authorised engineers can merge to `main`.
- Only authorised roles can approve production deployments.
- Production deployment permissions are held by a dedicated IAM role, not by individual users.

---

## Pending CI/CD Items

1. **Define change approval workflow** for production deployments.
2. **Confirm blue/green deployment requirement** for high-risk releases.
3. **Document database migration procedure** and rollback policy.
4. **Confirm OIDC integration** between GitHub Actions and AWS IAM.
5. **Define environment promotion criteria** (what must pass in staging before production).

Full details in `11-risks-and-todos.md`.

---

## Summary

DGBank deploys through a four-stage pipeline: GitHub source control, GitHub Actions build and test, Amazon ECR image scan via Amazon Inspector, and ECS Fargate rolling deployment. Every change is traceable from commit to production. Every deployment has a rollback path. Infrastructure and application code are deployed with the same safeguards.

---
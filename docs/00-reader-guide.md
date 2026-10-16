# 00 — Reader Guide

## Purpose of This Guide

This document explains how to read the rest of the repository, so that a first-time reader — technical or not — can navigate confidently.

---

## The Three Layers of This Documentation

**Layer 1 — The Story (docs/01 to docs/03)**
Explains *what* DGBank is building and *how a request flows through the system*. Start here regardless of your role.

**Layer 2 — The Domains (docs/04 to docs/10)**
Explains *how each part of the system is built*. Read the ones relevant to your role.

**Layer 3 — The Reference (services/, runbooks/)**
Lookup material. Use it when you need to know exactly what a service does or what to do during an incident.

---

## How We Describe a Request

Whenever we describe a flow, we use this format:

- **Scenario:** What the user is trying to do
- **Trigger:** What starts the flow
- **Path:** Step-by-step list of services the request passes through
- **Outcome:** What the user sees
- **Behind the scenes:** What is logged, monitored, or stored

This keeps flows consistent and easy to compare.

---

## How We Describe a Service

Every service in the dictionary uses four fields:

| Field | What it tells you |
|---|---|
| **Category** | Where it belongs in the architecture |
| **Service overview** | What the service is and what it does |
| **Purpose in DGBank** | Why DGBank uses it |
| **Position in architecture** | Which layer it operates in and which request journeys it touches |

---

## Glossary of Terms

| Term | Meaning |
|---|---|
| **VPC** | A private, isolated network inside AWS where DGBank's resources live |
| **Subnet** | A smaller section of a VPC. Public subnets face the internet; private subnets do not |
| **AZ (Availability Zone)** | A physically separate data centre within a region. Using multiple AZs protects against failure |
| **Region** | A geographic area containing multiple AZs. DGBank uses Tokyo |
| **Ingress** | Traffic entering the system |
| **Egress** | Traffic leaving the system |
| **North-south traffic** | Traffic entering or leaving the system from the internet |
| **East-west traffic** | Traffic moving between internal services |
| **Writer / Reader** | Two roles of a database. The Writer accepts changes; the Reader answers queries |
| **Blast radius** | How much damage a single failure or breach can cause |

---

## Reading Order by Role

**Non-technical reader:**
01 → 02 → 03 → 08 → 11

**New engineer:**
00 → 01 → 02 → 03 → 04 → 05 → 06 → 07 → 08 → 09 → 10 → 11

**Operator on shift:**
03 → 08 → runbooks/

**Security reviewer:**
04 → 05 → 09 → 11

**Manager preparing a status update:**
01 → 03 → 11
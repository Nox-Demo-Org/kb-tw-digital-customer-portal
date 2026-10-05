---
okf_version: '0.2'
title: Customer Portal
description: Standard Next.js development and execution via Node.js.
generated:
  at: '2026-10-05T12:42:58Z'
---

# Customer Portal

**customer-portal** (`tidewell.example`) is the customer-facing web application for Tidewell Mutual, built on Next.js 15 and React 19. It provides policyholders with self-service capabilities to view policy details, check payment schedules and instalment fees, submit new claims, and track open claim progress.

### Core Capabilities & Flows
- **Authentication & Sign-in**: Interacts with `customer-identity` (`POST /v1/auth/token`).
- **Policy Management**: Retrieves policy details from `policy-admin` (`GET /v1/policies/{id}`). Policy modifications (e.g., changes to address, cover, or vehicles) are directed offline to the contact centre phone support.
- **Billing & Payments**: Retrieves instalment and payment plan info from `billing-service` (`GET /v1/plans/{policyId}`) to display frequency and instalment fees.
- **Claims Submission & Tracking**: Submits new claims through `claims-intake` (`POST /v1/claims`) and monitors ongoing claim status via `claims-management` (`GET /v1/claims/{id}`). Maps internal claim statuses (`submitted`, `assigned`, `in_review`, `settled`, `paid`) into simplified customer-facing milestones.

### Running the Application
Configured via environment variables for upstream services:
- `CUSTOMER_IDENTITY_URL` (default: `http://customer-identity/v1`)
- `POLICY_ADMIN_URL` (default: `http://policy-admin/v1`)
- `BILLING_URL` (default: `http://billing-service/v1`)
- `CLAIMS_INTAKE_URL` (default: `http://claims-intake/v1`)
- `CLAIMS_URL` (default: `http://claims-management/v1`)

Standard Next.js development and execution via Node.js.

<!-- okf:contents -->

## Contents

- [Concepts and flows](/concepts/index.md) — 3 pages. Flows, lifecycles and cross-cutting mechanisms.
- [Architecture decisions](/decisions/index.md) — 2 pages. One ADR per architecture decision the code or documents make evident.
- [Components and data models](/entities/index.md) — 3 pages. One page per significant component and core data model.
- [Interfaces and references](/summaries/index.md) — 1 page. API, event and module references for the application.
- [Change log](/log.md) — every generation and sync, newest first.

<!-- /okf:contents -->

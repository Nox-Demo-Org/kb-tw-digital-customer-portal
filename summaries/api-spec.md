---
type: Interface Reference
title: API Specification
description: This document details the exposed user interface routes and the upstream backend REST APIs consumed by the customer-portal application.
resource: https://github.com/Nox-Demo-Org/kb-tw-digital-customer-portal/blob/main/summaries/api-spec.md
tags:
- customer-portal
- summaries
sources:
- resource: https://github.com/Nox-Demo-Org/customer-portal/blob/HEAD/lib/api.ts
- resource: https://github.com/Nox-Demo-Org/customer-portal/blob/HEAD/README.md
- resource: https://github.com/Nox-Demo-Org/customer-portal/blob/HEAD/app/billing/page.tsx
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:42:58Z'
---

<!-- anchor: lib/api.ts:L1-L15 -->
<!-- anchor: README.md:L1-L15 -->
<!-- anchor: app/billing/page.tsx:L1-L11 -->

# API Specification

This document details the exposed user interface routes and the upstream backend REST APIs consumed by the **customer-portal** application. For high-level microservice topology and environment variable configurations, see [[concepts/service-integration]].

---

## Exposed Portal Routes

The portal exposes user-facing Next.js App Router pages:

| Route | Source File | Purpose | Upstream Interaction | Related Entities / ADRs |
| --- | --- | --- | --- | --- |
| `GET /billing?policy={policyId}` | `app/billing/page.tsx` | Displays payment frequency and yearly instalment fee. | Calls `getPlan(policyId)` via `billing-service` (`GET /v1/plans/{policyId}`). | [[entities/billing-plan]] |
| `GET /claims/{id}` | `app/claims/[id]/page.tsx` | Renders the customer-facing claim status tracker and reported date. | Calls `getClaim(id)` via `claims-management` (`GET /v1/claims/{id}`). | [[entities/claim]], [[concepts/claim-tracking-flow]], [[decisions/adr-claim-tracker-step-mapping]] |
| `GET /policies` | `app/policies/page.tsx` | Displays policy listings and telephone routing for modifications (`03000000000`). | Retrieves policy details via `policy-admin` (`GET /v1/policies/{id}`). | [[entities/policy]], [[decisions/adr-policy-changes-via-phone]], [[concepts/contact-centre-escalations]] |

---

## Consumed Upstream Endpoints

Upstream REST clients are implemented in `lib/api.ts`. Base URLs are dynamically resolved using environment variables with fallback defaults.

```typescript
// lib/api.ts
const BASE = {
  identity: process.env.CUSTOMER_IDENTITY_URL ?? "http://customer-identity/v1",
  policy: process.env.POLICY_ADMIN_URL ?? "http://policy-admin/v1",
  billing: process.env.BILLING_URL ?? "http://billing-service/v1",
  intake: process.env.CLAIMS_INTAKE_URL ?? "http://claims-intake/v1",
  claims: process.env.CLAIMS_URL ?? "http://claims-management/v1",
};
```

### 1. `customer-identity`
- **Method & Path**: `POST /v1/auth/token`
- **Function**: Used for customer sign-in and token authentication.
- **Base URL Env**: `CUSTOMER_IDENTITY_URL` (default: `http://customer-identity/v1`)

### 2. `policy-admin`
- **Method & Path**: `GET /v1/policies/{id}`
- **Function**: `getPolicy(id: string)`
- **Base URL Env**: `POLICY_ADMIN_URL` (default: `http://policy-admin/v1`)
- **Payload & Model**: Returns policy records for self-service viewing.
- **Note**: The portal does *not* invoke `POST /v1/policies/{id}/changes`; policy updates are directed offline to the contact centre (see [[decisions/adr-policy-changes-via-phone]] and [[concepts/contact-centre-escalations]]).

### 3. `billing-service`
- **Method & Path**: `GET /v1/plans/{policyId}`
- **Function**: `getPlan(policyId: string)`
- **Base URL Env**: `BILLING_URL` (default: `http://billing-service/v1`)
- **Fields Consumed**:
  - `frequency`: string (e.g. payment frequency)
  - `fee_pence`: number (yearly instalment fee in pence, formatted as `£{(fee_pence / 100).toFixed(2)}`)
- **Entity Reference**: [[entities/billing-plan]]

### 4. `claims-intake`
- **Method & Path**: `POST /v1/claims`
- **Function**: `reportClaim(body: unknown)`
- **Base URL Env**: `CLAIMS_INTAKE_URL` (default: `http://claims-intake/v1`)
- **Usage**: Submits a new claim report.

### 5. `claims-management`
- **Method & Path**: `GET /v1/claims/{id}`
- **Function**: `getClaim(id: string): Promise<Claim>`
- **Base URL Env**: `CLAIMS_URL` (default: `http://claims-management/v1`)
- **TypeScript Type**:
  ```typescript
  export type Claim = {
    id: string;
    status: string;
    reportedAt: string;
    handlerName?: string;
    handlerAssignedAt?: string;
  };
  ```
- **Status Transformation**: The portal maps upstream statuses (`submitted`, `assigned`, `in_review`, `settled`, `paid`) to UI steps (`"Submitted"`, `"In review"`, `"Settled"`, `"Paid"`), hiding internal handler assignments (see [[entities/claim]] and [[decisions/adr-claim-tracker-step-mapping]]).

---

## Upstream Integration Summary

| Target Service | Endpoint | HTTP Method | Client Function | Default Base URL |
| --- | --- | --- | --- | --- |
| `customer-identity` | `/v1/auth/token` | `POST` | (Sign in authentication) | `http://customer-identity/v1` |
| `policy-admin` | `/v1/policies/{id}` | `GET` | `getPolicy(id)` | `http://policy-admin/v1` |
| `billing-service` | `/v1/plans/{policyId}` | `GET` | `getPlan(policyId)` | `http://billing-service/v1` |
| `claims-intake` | `/v1/claims` | `POST` | `reportClaim(body)` | `http://claims-intake/v1` |
| `claims-management` | `/v1/claims/{id}` | `GET` | `getClaim(id)` | `http://claims-management/v1` |

For overall portal architecture, see [[index]].

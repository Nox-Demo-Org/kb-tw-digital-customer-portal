---
type: Concept
title: Service Integration
description: The customer-portal application serves as the customer-facing frontend interface for Tidewell Mutual, orchestrating synchronous HTTP communication across multiple domain microservices.
resource: https://github.com/Nox-Demo-Org/kb-tw-digital-customer-portal/blob/main/concepts/service-integration.md
tags:
- customer-portal
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:42:58Z'
---

# Service Integration

The **customer-portal** application serves as the customer-facing frontend interface for Tidewell Mutual, orchestrating synchronous HTTP communication across multiple domain microservices.

---

## Environment Configuration

Upstream service locations are configured via environment variables defined in `lib/api.ts`. If an environment variable is omitted, the application falls back to standard internal hostnames:

| Environment Variable | Target Service | Default URL | Purpose |
| --- | --- | --- | --- |
| `CUSTOMER_IDENTITY_URL` | `customer-identity` | `http://customer-identity/v1` | Customer authentication and token generation |
| `POLICY_ADMIN_URL` | `policy-admin` | `http://policy-admin/v1` | Policy retrieval and customer policy views |
| `BILLING_URL` | `billing-service` | `http://billing-service/v1` | Billing plan details, frequencies, and instalment fees |
| `CLAIMS_INTAKE_URL` | `claims-intake` | `http://claims-intake/v1` | First Notice of Loss (FNOL) claim submission |
| `CLAIMS_URL` | `claims-management` | `http://claims-management/v1` | Tracking existing claim status, dates, and handler details |

---

## Upstream Microservices

The application interacts synchronously via REST over HTTP with five backend systems across the Tidewell platform:

```
                      +-------------------+
                      |  customer-portal  |
                      +---------+---------+
                                |
       +----------------+-------+--------+----------------+
       |                |                |                |
+------v-------+ +------v-------+ +------v-------+ +------v-------+ +------v-------+
|  customer-   | |    policy-   | |   billing-   | |    claims-   | |    claims-   |
|   identity   | |     admin    | |    service   | |    intake    | |  management  |
+--------------+ +--------------+ +--------------+ +--------------+ +--------------+
```

### 1. Customer Identity (`customer-identity`)
- **Domain Team**: Customer Platform
- **Endpoint**: `POST /v1/auth/token`
- **Role**: Authenticates policyholders entering the portal.

### 2. Policy Admin (`policy-admin`)
- **Domain Team**: Policy
- **Endpoint**: `GET /v1/policies/{id}`
- **Role**: Fetches policy attributes for the "My policies" view ([[entities/policy]]). Note that policy updates (address, car, cover) are not exposed for self-service edits and require phone routing to the contact centre ([[decisions/adr-policy-changes-via-phone]]).

### 3. Billing Service (`billing-service`)
- **Domain Team**: Billing & Payments
- **Endpoint**: `GET /v1/plans/{policyId}`
- **Role**: Provides schedule details and instalment fee calculations ([[entities/billing-plan]]). Backend billing lifecycle events such as [[ap:kb-tw-billing-billing-service/summaries/events-spec#billing-instalment-due|billing.instalment.due]] and [[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|payments.collection.succeeded]] are handled downstream asynchronously via Google Cloud Pub/Sub.

### 4. Claims Intake (`claims-intake`)
- **Domain Team**: Claims
- **Endpoint**: `POST /v1/claims`
- **Role**: Submits new insurance claims reported online. Once received by `claims-intake`, downstream processing publishes `claims.claim.reported` for fraud scoring and handler assignment.

### 5. Claims Management (`claims-management`)
- **Domain Team**: Claims
- **Endpoint**: `GET /v1/claims/{id}`
- **Role**: Retrieves open and historical claim progress, reporting dates, and assigned handler metadata ([[entities/claim]], [[concepts/claim-tracking-flow]]).

---

## Client API Implementation

The client methods are centralized in `lib/api.ts`:

```typescript
const BASE = {
  identity: process.env.CUSTOMER_IDENTITY_URL ?? "http://customer-identity/v1",
  policy: process.env.POLICY_ADMIN_URL ?? "http://policy-admin/v1",
  billing: process.env.BILLING_URL ?? "http://billing-service/v1",
  intake: process.env.CLAIMS_INTAKE_URL ?? "http://claims-intake/v1",
  claims: process.env.CLAIMS_URL ?? "http://claims-management/v1",
};

export type Claim = { 
  id: string; 
  status: string; 
  reportedAt: string; 
  handlerName?: string; 
  handlerAssignedAt?: string 
};

export const getClaim = (id: string): Promise<Claim> => 
  fetch(`${BASE.claims}/claims/${id}`).then((r) => r.json());

export const getPlan = (policyId: string) => 
  fetch(`${BASE.billing}/plans/${policyId}`).then((r) => r.json());

export const getPolicy = (id: string) => 
  fetch(`${BASE.policy}/policies/${id}`).then((r) => r.json());

export const reportClaim = (body: unknown) =>
  fetch(`${BASE.intake}/claims`, { 
    method: "POST", 
    body: JSON.stringify(body) 
  }).then((r) => r.json());
```

---

## Architectural Context & Asynchronous Backing

While `customer-portal` acts purely as an HTTP client communicating directly with REST APIs, the wider Tidewell platform uses Google Cloud Pub/Sub for asynchronous event communication between domain services (such as [[ap:kb-tw-billing-billing-service/summaries/api-spec#policy-policy-issued|policy.policy.issued]], `claims.handler.assigned`, and `claims.claim.settled`).

For additional details on route mappings and escalations, refer to:
- [[summaries/api-spec]] for route definitions.
- [[concepts/contact-centre-escalations]] for operational escalation paths.
- [[decisions/adr-claim-tracker-step-mapping]] for downstream claim status presentation logic.

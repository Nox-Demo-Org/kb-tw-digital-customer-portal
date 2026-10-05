---
type: Component
title: Policy
description: The Policy entity representation in the customer-portal provides customers with a read-only self-service view of their insurance policies retrieved from the upstream policy-admin microservice.
resource: https://github.com/Nox-Demo-Org/kb-tw-digital-customer-portal/blob/main/entities/policy.md
tags:
- customer-portal
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/customer-portal/blob/HEAD/app/policies/page.tsx
- resource: https://github.com/Nox-Demo-Org/customer-portal/blob/HEAD/lib/api.ts
- resource: https://github.com/Nox-Demo-Org/customer-portal/blob/HEAD/README.md
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:42:58Z'
---

<!-- anchor: app/policies/page.tsx:L1-L5 -->
<!-- anchor: lib/api.ts:L1-L15 -->
<!-- anchor: README.md:L1-L15 -->

# Policy

The **Policy** entity representation in the `customer-portal` provides customers with a read-only self-service view of their insurance policies retrieved from the upstream `policy-admin` microservice.

## Overview & Self-Service View

Policy information is rendered on the `/policies` route (`app/policies/page.tsx`). The page lists policies for the signed-in policyholder and provides direct guidance for making policy alterations.

### Modification Boundaries
Customers cannot change their policy details (such as address, vehicle/car, or cover level) directly within the self-service web interface. 
- The modification API endpoint `POST /v1/policies/{id}/changes` is not wired into the customer portal.
- Instead, the interface routes customers to contact centre telephone support via `<a href="tel:03000000000">Call us to make a change</a>`.
- Contact centre agents modify policy details directly within `policy-admin`.

For more details on this architectural boundary, see [[decisions/adr-policy-changes-via-phone]] and [[concepts/contact-centre-escalations]].

## Client Integration

Policy data is fetched using client helpers defined in `lib/api.ts`:

```typescript
export const getPolicy = (id: string) => 
  fetch(`${BASE.policy}/policies/${id}`).then((r) => r.json());
```

The upstream base URL is resolved via the `POLICY_ADMIN_URL` environment variable, defaulting to `http://policy-admin/v1` (see [[concepts/service-integration]]).

## Responsibilities

- **Policy Retrieval**: Querying `policy-admin` via `GET /v1/policies/{id}` to display policy details.
- **Route Handling**: Exposing the `GET /policies` page to policyholders.
- **Change Routing**: Directing customer modification requests offline to the contact centre phone line (`03000000000`) rather than offering in-portal self-service mutations.

## Dependencies

- **`policy-admin`**: Upstream service providing `GET /v1/policies/{id}`. Configured via `POLICY_ADMIN_URL`. See [[summaries/api-spec]].
- **Contact Centre**: Serves as the primary operational escalation and modification path for all policy adjustment requests (see [[concepts/contact-centre-escalations]]).

---
type: Component
title: Billing Plan
description: The Billing Plan entity represents a policyholder's payment schedule and instalment fee metadata fetched from the upstream billing-service.
resource: https://github.com/Nox-Demo-Org/kb-tw-digital-customer-portal/blob/main/entities/billing-plan.md
tags:
- customer-portal
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/customer-portal/blob/HEAD/app/billing/page.tsx
- resource: https://github.com/Nox-Demo-Org/customer-portal/blob/HEAD/lib/api.ts
- resource: https://github.com/Nox-Demo-Org/customer-portal/blob/HEAD/README.md
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:42:58Z'
---

<!-- anchor: app/billing/page.tsx:L1-L11 -->
<!-- anchor: lib/api.ts:L1-L15 -->
<!-- anchor: README.md:L1-L15 -->

# Billing Plan

The **Billing Plan** entity represents a policyholder's payment schedule and instalment fee metadata fetched from the upstream `billing-service`. It is consumed by the customer portal to present payment terms and cost breakdowns on the `/billing` route.

## Data Structure

The billing plan payload returned by `getPlan(policyId)` (from endpoint `GET /v1/plans/{policyId}`) contains the following fields:

| Field | Type | Description |
| --- | --- | --- |
| `frequency` | `string` | The recurring payment schedule frequency (e.g., monthly, annually). |
| `fee_pence` | `number` | The total instalment fee charged for the current policy year, denominated in integer pence. |

### Presentation Logic

In `app/billing/page.tsx`, the portal formats the integer pence value into a standard currency string (GBP) for customer display:

```typescript
<p>
  Paying {plan.frequency}. Instalment fee this year: £{(plan.fee_pence / 100).toFixed(2)}
</p>
```

## Responsibilities

- **Payment Term Display**: Supplies the payment frequency for a specific [[entities/policy]] to inform policyholders of their scheduled collection intervals.
- **Instalment Fee Calculation**: Conveys the annual instalment surcharge (`fee_pence`) in pence, which the UI converts to standard pound currency display (`£{(plan.fee_pence / 100).toFixed(2)}`).
- **Billing Route Delivery**: Serves as the primary data model backing the `GET /billing` page rendered by `app/billing/page.tsx`.

## Dependencies

- **Upstream Services**:
  - `billing-service`: Provides `GET /v1/plans/{policyId}` via `BILLING_URL` (default: `http://billing-service/v1`). See [[concepts/service-integration]] and [[summaries/api-spec]].
  - Related billing events tracked in the wider ecosystem include [[ap:kb-tw-billing-billing-service/summaries/events-spec#billing-instalment-due|billing-service (billing.instalment.due)]] and [[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]].
- **API Client**:
  - `getPlan(policyId: string)` in `lib/api.ts`.
- **Related Pages & Components**:
  - `app/billing/page.tsx` (`Payments` component).
- **Related Entities**:
  - [[entities/policy]]: Each billing plan is queried using the associated policy identifier passed via query parameter `?policy={policyId}`.
- **Escalations**:
  - Discrepancies or customer queries regarding payment schedules or instalment charges are directed to `#tw-contact-centre` as detailed in [[concepts/contact-centre-escalations]].

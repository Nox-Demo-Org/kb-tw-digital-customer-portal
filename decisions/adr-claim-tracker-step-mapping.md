---
type: Architecture Decision
title: 'ADR: Claim Tracker Step Mapping & Handler Visibility'
description: 'In the initial implementation of the claim status tracking interface (GET /claims/{id} rendered in app/claims/[id]/page.tsx), a simplified 4-step progress tracker was designed for policyholders: 1.'
resource: https://github.com/Nox-Demo-Org/kb-tw-digital-customer-portal/blob/main/decisions/adr-claim-tracker-step-mapping.md
tags:
- customer-portal
- decisions
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:42:58Z'
---

# ADR: Claim Tracker Step Mapping & Handler Visibility

## Status
Accepted (v1 Implementation; under review for Digital Claims v2)

## Context
When policyholders report an insurance claim via [[index|customer-portal]] (`POST /v1/claims` to `claims-intake`), the claim lifecycle is tracked internally by `claims-management` through multiple discrete statuses: `submitted`, `assigned`, `in_review`, `settled`, and `paid`. 

In the initial implementation of the claim status tracking interface (`GET /claims/{id}` rendered in `app/claims/[id]/page.tsx`), a simplified 4-step progress tracker was designed for policyholders:
1. `Submitted`
2. `In review`
3. `Settled`
4. `Paid`

During this initial build, the internal status `assigned`—which indicates that a claims handler has been assigned to the claim—was not allocated a dedicated milestone step in the customer-facing UI. Additionally, handler details (such as `handlerName` provided by `claims-management` via `GET /v1/claims/{id}`) were not displayed.

## Decision
In `app/claims/[id]/page.tsx`, the portal explicitly maps both `submitted` and `assigned` internal statuses directly to the customer-facing `"Submitted"` milestone:

```typescript
const STEPS = ["Submitted", "In review", "Settled", "Paid"] as const;

const STEP_FOR: Record<string, (typeof STEPS)[number]> = {
  submitted: "Submitted",
  assigned: "Submitted",
  in_review: "In review",
  settled: "Settled",
  paid: "Paid",
};
```

1. **Collapsing Statuses**: The tracker collapses `assigned` into `Submitted`.
2. **Omission of Handler Information**: `handlerName` returned from `claims-management` is not surfaced anywhere in the UI. The page only renders the current active milestone step, claim ID, and reported date formatted in `en-GB` (`"Reported <date>. We'll be in touch."`).

## Consequences

### Positive
- Simplifies the initial user interface down to four linear milestones.
- Decouples early customer UI delivery from handler assignment notification mechanisms.

### Negative & Operational Impact
- **Customer Blind Spot**: While the median time from report to handler assignment is 1.3 working days, the median time to reach `in_review` is 4.6 working days. Because `assigned` maps to `Submitted`, the UI shows no visible progress for over 4 days.
- **Contact Centre Strain**: As documented in the Q3 2026 digital claims review, 41% of online claims result in a "where is my claim?" call within 5 days of submission, costing £6.80 per call. Customers frequently report confusion when a handler calls them directly while the portal continues to show `Submitted` (see [[concepts/contact-centre-escalations|Contact Centre Escalations]] and Jira `TWCLM-2`).
- **Satisfaction Impact**: Overall customer claims satisfaction dropped to 3.4/5, driven by lack of visibility into whether an internal handler has picked up the file.

### Future Evolution
Under the **Digital Claims v2** initiative (Jira `TWCLM-1`, `TWCLM-3`, `TWCLM-5` and Notion discovery), requirements are in discovery to:
- Introduce explicit tracker visibility for handler assignment (including handler first name and date assigned).
- Integrate SMS/text notifications when `claims.handler.assigned` is published via Google Cloud Pub/Sub.
- Update [[entities/claim|Claim Data Models]] and [[concepts/claim-tracking-flow|Claim Tracking Flow]] to eliminate the customer visibility gap.

## Related Documentation
- [[entities/claim|Entities: Claim Model]]
- [[concepts/claim-tracking-flow|Concepts: Claim Tracking Flow]]
- [[concepts/contact-centre-escalations|Concepts: Contact Centre Escalations]]
- [[summaries/api-spec|API Specification]]
- [[index|Customer Portal Architecture]]

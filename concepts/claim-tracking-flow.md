---
type: Concept
title: Claim Tracking Flow
description: The claim tracking lifecycle in customer-portal provides policyholders with a visual status tracker for open insurance claims rendered at /claims/{id}.
resource: https://github.com/Nox-Demo-Org/kb-tw-digital-customer-portal/blob/main/concepts/claim-tracking-flow.md
tags:
- customer-portal
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:42:58Z'
---

# Claim Tracking Flow

The claim tracking lifecycle in **customer-portal** provides policyholders with a visual status tracker for open insurance claims rendered at `/claims/{id}`. This page details how upstream statuses from `claims-management` are translated into customer-facing milestones, the architectural decoupling between internal processing and portal presentation, and the resulting customer experience friction points identified in operational reviews.

---

## Tracking Architecture and Status Mapping

When a policyholder views a claim tracker at `GET /claims/{id}`, the portal fetches the claim details using `getClaim(params.id)` from `claims-management` via `GET /v1/claims/{id}` (see [[summaries/api-spec]] and [[concepts/service-integration]]).

```
  Upstream Status                       Customer Portal Tracker Step
 (claims-management)                           (app/claims/[id]/page.tsx)
 ┌──────────────────┐
 │    submitted     │ ───────────────┐
 └──────────────────┘                │
                                     ▼
 ┌──────────────────┐         ┌─────────────┐
 │     assigned     │ ───────►│  Submitted  │ (aria-current)
 └──────────────────┘         └─────────────┘
                                     │
 ┌──────────────────┐                ▼
 │    in_review     │ ───────►┌─────────────┐
 └──────────────────┘         │  In review  │
                               └─────────────┘
                                     │
 ┌──────────────────┐                ▼
 │     settled      │ ───────►┌─────────────┐
 └──────────────────┘         │   Settled   │
                               └─────────────┘
                                     │
 ┌──────────────────┐                ▼
 │       paid       │ ───────►┌─────────────┐
 └──────────────────┘         │    Paid     │
                               └─────────────┘
```

### Tracker Implementation (`app/claims/[id]/page.tsx`)

The UI renders an ordered list (`<ol>`) representing four discrete customer milestones:

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

- **Current Step Active State**: Determined by evaluating `STEP_FOR[claim.status] ?? "Submitted"`, setting the `aria-current` attribute on the corresponding `<li>` element.
- **Reporting Date Display**: Formatted as `en-GB` date from `claim.reportedAt` (`Reported DD/MM/YYYY. We'll be in touch.`).
- **Handler Data Abstraction**: As documented in [[decisions/adr-claim-tracker-step-mapping]], the internal status `assigned` is collapsed into the `Submitted` stage, and `claim.handlerName` is explicitly omitted from the customer-facing view. For the underlying claim schema, see [[entities/claim]].

---

## End-to-End Claim Lifecycle

Claim progression spans multiple asynchronous and synchronous touchpoints across the Tidewell platform:

1. **Intake**: The customer submits a claim via `POST /v1/claims` targeting `claims-intake`. The service issues the `claims.claim.reported` event over Google Cloud Pub/Sub.
2. **Fraud Evaluation**: `fraud-scoring` consumes `claims.claim.reported`. Scores exceeding `0.8` trigger a `fraud.score.flagged` event.
3. **Handler Assignment**: `claims-management` assigns a claims handler to the record (internal status transitions from `submitted` to `assigned`) and publishes `claims.handler.assigned`.
4. **Triage & Review**: The claim transitions to `in_review` once formal investigation begins.
5. **Settlement & Payout**: Upon reaching `settled`, `claims-management` issues `claims.claim.settled` and calls `POST /v1/payouts` against `payments-gateway`. `payments-gateway` subsequently emits `payments.payout.sent` when funds clear (updating the status to `paid`).

---

## Operational Impact and Customer Experience Gaps

Data from the **Q3 2026 Digital Claims Journey Review** revealed substantial customer friction stemming from the tracker's status abstraction:

| Metric | Recorded Figure | Operational Context |
| :--- | :--- | :--- |
| **Online Claim Share** | 64% | Up from 51% in the previous year. |
| **"Where is my claim?" Inquiries** | 41% of online claims | Inbound calls received within 5 days of submission. |
| **Contact Centre Cost** | £6.80 per call | Substantial operational expense driven by status inquiry call volume. |
| **Assignment Duration** | 1.3 working days (median) | Time taken from intake to handler assignment (`assigned`). |
| **Review Duration** | 4.6 working days (median) | Time before status updates from `Submitted` to `In review` on the portal. |
| **Claims CSAT** | 3.4 / 5.0 | Primary detractor cited: *"not knowing if anyone has picked it up"*. |

### Key Failure Modes
- **Status Stagnation & Telephony Desynchronisation**: Customers whose claims have been picked up and who have already received direct phone contact from a handler (e.g., ticket `TWCLM-2`) find the portal tracker still displaying `Submitted`. This creates customer confusion and triggers follow-up verification calls to the contact centre (see [[concepts/contact-centre-escalations]]).
- **Information Latency**: Because `assigned` maps to `Submitted`, policyholders experience an average blind spot of 4.6 working days before seeing any tracker movement.

---

## Digital Claims v2 Roadmap

To address the 41% inquiry rate and bridge tracker information gaps, several initiatives are tracked under Jira epic `TWCLM-1` and the Digital Channels discovery backlog:

1. **Handler Transparency**: Update the tracker step pipeline to display when a handler is assigned, exposing the handler's first name and assignment date.
2. **SMS Notification Pipeline (`TWCLM-3`)**: Consume `claims.handler.assigned` (implemented in `TWCLM-5`) via `notifications-hub` to send an automated SMS when handler triage occurs.
3. **Excess Visibility (`TWCLM-4`)**: Render the policy excess amount directly on the tracking view.
4. **Self-Service Evidence Uploads**: Allow customers to upload photographic evidence directly to open claims via the portal interface.

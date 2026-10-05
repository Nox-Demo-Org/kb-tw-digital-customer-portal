---
type: Concept
title: Contact Centre Escalations
description: In the customer-portal architecture, several customer lifecycle events and service operations are not fully automated through self-service endpoints.
resource: https://github.com/Nox-Demo-Org/kb-tw-digital-customer-portal/blob/main/concepts/contact-centre-escalations.md
tags:
- customer-portal
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:42:58Z'
---

# Contact Centre Escalations

In the [[index|customer-portal]] architecture, several customer lifecycle events and service operations are not fully automated through self-service endpoints. Instead, they require customer handoffs to the telephone contact centre (`03000000000`) or generate inbound contact centre call volumes due to digital experience gaps.

---

## Escalation Triggers and Operational Touchpoints

### 1. Policy Modifications and Mid-Term Adjustments
While customers can view their active policies via `GET /policies` (sourced from `policy-admin`), the portal does not wire up self-service change endpoints (e.g. `POST /v1/policies/{id}/changes`). 

* **Mechanism**: The policy list interface explicitly directs customers to call phone support (`tel:03000000000`) for adjustments (such as address changes or cover modifications). See [[decisions/adr-policy-changes-via-phone]].
* **Downstream Discrepancies**: Address updates taken manually by the contact centre update `policy-admin`, but `billing-service` currently captures the address only during initial plan creation ([[ap:kb-tw-billing-billing-service/summaries/api-spec#policy-policy-issued|billing-service (policy.policy.issued)]]) and does not ingest subsequent updates. This results in billing correspondence (e.g., missed-payment notices) continuing to go to legacy addresses.

### 2. Claim Status Inquiries ("Where is my claim?")
Customer tracking under `GET /claims/{id}` abstracts internal states from `claims-management` into simplified UI steps. 

* **Tracker Gap**: The portal tracker collapses the internal `assigned` status into the first step ("Submitted") and does not surface the handler name or assignment date (see [[decisions/adr-claim-tracker-step-mapping]] and [[concepts/claim-tracking-flow]]).
* **Contact Centre Impact**:
  * While the median time from report to handler assignment is **1.3 working days**, the portal tracker remains visually stuck at "Submitted" until reaching `in_review` (a median of **4.6 working days**).
  * According to the Q3 2026 Digital Claims Journey Review, **41% of online claims** generate a "Where is my claim?" phone call within 5 days of submission.
  * Inbound calls occur even when a handler has already phoned the customer, because the portal UI still reads "Submitted".
  * Each inquiry call costs **£6.80** in contact centre operational time.

```
Customer Submits Claim (POST /v1/claims)
   │
   ▼
Portal Status: "Submitted"
   │
   ├─► claims-management assigns handler (median: 1.3 days)
   │   └─► Portal STILL displays "Submitted" (no handler info)
   │
   ├─► 41% of customers call Contact Centre within 5 days (£6.80/call)
   │
   ▼
claims-management moves to "in_review" (median: 4.6 days)
   │
   ▼
Portal Status: "In Review"
```

### 3. Instalment Fee Disputes and Manual Refunds
Policyholders who pay via monthly payment plans are billed an instalment fee per schedule via `billing-service` (see [[entities/billing-plan]]).

* **Operational Friction**:
  * Contact centre agents handle approximately **300 fee refund requests per month** by hand.
  * Each refund handling session averages **12 minutes** of contact centre agent time on the line.
  * Long-standing policyholders (e.g., 8–15 years) frequently call to dispute paying additional monthly instalment fees, driving elevated churn rates (long-standing monthly customers cancel at approximately twice the rate following a fee rise).

---

## Contact Centre & Business Metrics Summary

| Area | Root Cause in Digital Portal / Backend | Operational Impact |
| :--- | :--- | :--- |
| **Claim Inquiries** | Assignment milestone and handler details hidden from tracker (`GET /claims/{id}`) | 41% of online claims call within 5 days; £6.80 per call |
| **Instalment Fees** | Fixed monthly instalment fees charged across all customer tenures | ~300 manual refunds/month; 12 min average call duration |
| **Policy Adjustments** | `POST /v1/policies/{id}/changes` not exposed in UI; routed to `03000000000` | Manual handling of all mid-term policy changes; cross-service address sync issues with billing |

---

## Related Topics

- [[entities/policy]] - Self-service policy view and phone change routing
- [[entities/claim]] - Claim models and status mapping details
- [[concepts/claim-tracking-flow]] - End-to-end claim reporting and tracking flow
- [[decisions/adr-claim-tracker-step-mapping]] - Architectural decision to hide handler assignment details
- [[decisions/adr-policy-changes-via-phone]] - Architectural decision to handle policy updates offline

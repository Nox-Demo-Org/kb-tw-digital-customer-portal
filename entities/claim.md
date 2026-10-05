---
type: Component
title: Claim
description: The Claim entity represents an insurance claim within the customer-portal application.
resource: https://github.com/Nox-Demo-Org/kb-tw-digital-customer-portal/blob/main/entities/claim.md
tags:
- customer-portal
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/customer-portal/blob/HEAD/app/claims/[id]/page.tsx
- resource: https://github.com/Nox-Demo-Org/customer-portal/blob/HEAD/lib/api.ts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:42:58Z'
---

<!-- anchor: app/claims/[id]/page.tsx:L1-L30 -->
<!-- anchor: lib/api.ts:L1-L15 -->

# Claim

The **Claim** entity represents an insurance claim within the `customer-portal` application. It encapsulates the claim's identifier, processing status, submission date, and optional handler assignment metadata retrieved from upstream claims services.

---

## Responsibilities

The Claim model and its associated API utilities and tracking components fulfill several core functions:
- **Intake & Creation**: Submits new insurance claim payloads via `reportClaim` to the `claims-intake` service (`POST /v1/claims`).
- **State Retrieval**: Fetches claim status and handler metadata via `getClaim` from the `claims-management` service (`GET /v1/claims/{id}`).
- **Status Abstraction & Milestone Mapping**: Maps granular upstream backend statuses (`submitted`, `assigned`, `in_review`, `settled`, `paid`) into simplified user-facing tracking milestones displayed on the claim tracking page (`/claims/[id]`). See [[concepts/claim-tracking-flow]] and [[decisions/adr-claim-tracker-step-mapping]].

---

## Data Model

Defined in `lib/api.ts`, the TypeScript interface `Claim` defines the data received from the upstream `claims-management` service:

```typescript
export type Claim = {
  id: string;
  status: string;
  reportedAt: string;
  handlerName?: string;
  handlerAssignedAt?: string;
};
```

### Field Definitions

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Unique claim identifier (e.g., `CLM-88213`). |
| `status` | `string` | Raw upstream lifecycle status (`submitted`, `assigned`, `in_review`, `settled`, `paid`). |
| `reportedAt` | `string` | ISO 8601 date string indicating when the claim was originally reported. |
| `handlerName` | `string` (optional) | Name of the assigned claims handler. Received from backend but omitted from tracker display. |
| `handlerAssignedAt` | `string` (optional) | Timestamp when the handler was assigned. |

---

## Status Mapping & Customer Milestones

In `app/claims/[id]/page.tsx`, the application normalises the raw claim status into four customer-facing progress steps:

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

### Mapping Rules
- **`submitted` & `assigned` $\rightarrow$ "Submitted"**: When a claim is `assigned` in the backend, the portal does not display an independent milestone; it remains mapped to `"Submitted"`. `handlerName` and `handlerAssignedAt` are not rendered in the user interface, prompting customer inquiries documented in [[concepts/contact-centre-escalations]] and [[decisions/adr-claim-tracker-step-mapping]].
- **`in_review` $\rightarrow$ "In review"**: Active triage and evaluation by claims handlers.
- **`settled` $\rightarrow$ "Settled"**: Agreement reached on settlement terms.
- **`paid` $\rightarrow$ "Paid"**: Payout finalised.
- **Fallback**: Any unrecognized status defaults to `"Submitted"` via `STEP_FOR[claim.status] ?? "Submitted"`.

---

## Dependencies

### Upstream Services
- **`claims-intake`** (`CLAIMS_INTAKE_URL`, default `http://claims-intake/v1`): Receives new claim submissions via `reportClaim()` at `POST /v1/claims`. See [[concepts/service-integration]] and [[summaries/api-spec]].
- **`claims-management`** (`CLAIMS_URL`, default `http://claims-management/v1`): Provides claim records and status data via `getClaim()` at `GET /v1/claims/{id}`.

### Downstream UI Consumers
- **`app/claims/[id]/page.tsx`**: Server component that renders the claim tracker view (`<ClaimTracker>`), parsing `params.id`, formatting `reportedAt` using `toLocaleDateString("en-GB")`, and highlighting the current step with `aria-current`.

---
mission: NOX-12
title: 'Handler visibility on the claim tracker'
role: engineering
status: approved
version: 2
author: unknown
ai_drafted: false
approved_at: 2026-10-18T10:42:42Z
---

# Engineering design: Handler visibility on the claim tracker

## Applications changing

- **`customer-portal` (`tw-digital/customer-portal`)** — Owned by the **Digital Channels team**.
  - **Why it must change**: The customer-facing claim tracker page (`app/claims/[id]/page.tsx`) explicitly collapses the internal `assigned` status into the `"Submitted"` progress step and suppresses handler metadata (`handlerName`, `handlerAssignedAt`) [[kb:customer-portal/decisions/adr-claim-tracker-step-mapping]], [[media:ec737e1c-b9a3-4877-8a01-95476cbe35eb]]. It must be modified to advance the progress indicator to `"In review"` when an internal handler is assigned, and render the handler's first name, team, and assignment date.
- **Notable applications that must not change**:
  - **`claims-management`** (Claims team): Already exposes `GET /v1/claims/{id}` providing `id`, `status`, `reportedAt`, `handlerName`, and `handlerAssignedAt` [[kb:customer-portal/summaries/api-spec]], [[kb:customer-portal/entities/claim]]. Its data schema, status transition engine, and triage assignment routines remain untouched.
  - **`claims-intake`** (Claims team): Responsible strictly for First Notice of Loss claim ingestion (`POST /v1/claims`) [[kb:customer-portal/concepts/service-integration]]. Untouched.
  - **`notifications-hub`** (Platform / Customer Communications team): Automated SMS notifications on `claims.handler.assigned` (TWCLM-3 / TWCLM-5) are out of scope for this mission [[kb:customer-portal/concepts/claim-tracking-flow]], [[media:ec737e1c-b9a3-4877-8a01-95476cbe35eb]]. Untouched.
  - **`policy-admin`** (Policy team) & **`billing-service`** (Billing & Payments team): Unrelated domain services [[kb:customer-portal/concepts/service-integration]]. Untouched.

## Approach

The implementation will follow the existing React Server Component pattern in Next.js 15 App Router (`app/claims/[id]/page.tsx`) [[kb:customer-portal/index]], consuming upstream data via `getClaim` in `lib/api.ts` [[kb:customer-portal/concepts/service-integration]].

1. **Milestone mapping update in `app/claims/[id]/page.tsx`**:
   - Keep the four existing customer milestones: `STEPS = ["Submitted", "In review", "Settled", "Paid"] as const` [[kb:customer-portal/concepts/claim-tracking-flow]].
   - Update `STEP_FOR` so that `assigned` maps to `"In review"` rather than `"Submitted"`:
     ```typescript
     const STEP_FOR: Record<string, (typeof STEPS)[number]> = {
       submitted: "Submitted",
       assigned: "In review",
       in_review: "In review",
       settled: "Settled",
       paid: "Paid",
     };
     ```
   - This immediately advances the visual milestone (`aria-current`) once internal triage allocates a handler, eliminating the 4.6-day stagnation period [[kb:customer-portal/concepts/contact-centre-escalations]], [[media:d1540d7c-27ed-4c39-9dc2-67414c025c75]].

2. **Handler metadata presentation & privacy controls**:
   - Upstream `claims-management` returns `handlerName` (e.g. `"Priya Raman (Home, South West)"`) and `handlerAssignedAt` (ISO-8601 string) [[kb:customer-portal/entities/claim]], [[media:d1540d7c-27ed-4c39-9dc2-67414c025c75]].
   - Format the handler display string to isolate first name and department/team (e.g., `"Priya (Home, South West)"` or `"Priya · Home Claims"`), strictly preventing disclosure of personal contact details, surnames where policy dictates, or direct contact methods per AC-7.
   - If `claim.status === "assigned"` (or downstream) but `handlerName` is null or empty, render the fallback copy: `"Assigned to a claims specialist"`.
   - Format `handlerAssignedAt` using `Intl.DateTimeFormat('en-GB')` or `toLocaleDateString("en-GB")` to output the standard UK format `DD/MM/YYYY` [[kb:customer-portal/concepts/claim-tracking-flow]].
   - For newly reported claims with `claim.status === "submitted"`, maintain the current display: highlight `"Submitted"`, show `"Reported <date>. We'll be in touch."`, and omit handler elements entirely [[kb:customer-portal/decisions/adr-claim-tracker-step-mapping]].
   - For downstream states (`in_review`, `settled`, `paid`), retain the handler details on the claim summary if handler metadata is present.

3. **Data fetching caching configuration**:
   - Ensure `getClaim` in `lib/api.ts` does not cache claim tracking responses indefinitely (`cache: 'no-store'` or revalidation on page load), ensuring customer refreshes immediately reflect internal reassignments [[kb:customer-portal/concepts/claim-tracking-flow]].

## Contracts affected

| Contract | Owner app | Consumers | Status | Notes |
| :--- | :--- | :--- | :--- | :--- |
| `GET /claims/{id}` | `customer-portal` | Policyholders (browsers) | **Additive** | User-facing route in `app/claims/[id]/page.tsx`. HTML representation updated to show advanced milestone `"In review"` and handler details when claim is assigned [[kb:customer-portal/summaries/api-spec]]. |
| `GET /v1/claims/{id}` | `claims-management` | `customer-portal` | **Unchanged** | Backend REST contract already returns `id`, `status`, `reportedAt`, `handlerName`, and `handlerAssignedAt` [[kb:customer-portal/entities/claim]]. No contract changes required. |
| `claims.handler.assigned` | `claims-management` | `notifications-hub` | **Unchanged** | Pub/Sub domain event remains untouched [[kb:customer-portal/concepts/service-integration]], [[media:ec737e1c-b9a3-4877-8a01-95476cbe35eb]]. |

## Must not break

- **Unassigned claim rendering**: Newly submitted claims (`status: "submitted"`) must continue to highlight milestone `"Submitted"` with reported date (`"Reported DD/MM/YYYY. We'll be in touch."`) without rendering empty handler containers or throwing runtime errors [[kb:customer-portal/concepts/claim-tracking-flow]].
- **Downstream claim status rendering**: Completed claims (`status: "settled"` or `"paid"`) must highlight their respective final milestone steps and retain handler history [[kb:customer-portal/decisions/adr-claim-tracker-step-mapping]].
- **Authentication and unrelated routes**: Customer session verification (`POST /v1/auth/token`), billing views (`GET /billing`), and policy views (`GET /policies`) must remain completely untouched [[kb:customer-portal/summaries/api-spec]].
- **Handler privacy guardrail**: No direct handler contact details (direct phone numbers, personal email addresses) may be exposed in the UI or client markup [[kb:customer-portal/decisions/adr-claim-tracker-step-mapping]].
- **Upstream error handling**: Upstream API connectivity failures on `getClaim` must continue to bubble to standard Next.js error boundaries rather than rendering broken partial layouts.

## Architecture, guardrails and standards

- **ADR supersession**:
  - Formally supersedes the status-collapsing decision in [[kb:customer-portal/decisions/adr-claim-tracker-step-mapping]], which previously mapped internal `assigned` to `"Submitted"` and hid `handlerName`.
  - Maintains compliance with [[kb:customer-portal/decisions/adr-policy-changes-via-phone]] by avoiding unnecessary interactive self-service forms and keeping the tracking page clean and informational.
- **Layering and modularity**:
  - Preserves the Next.js 15 App Router Server Component architecture [[kb:customer-portal/index]]. UI rendering logic resides in `app/claims/[id]/page.tsx`, and HTTP client communication remains centralized in `lib/api.ts` [[kb:customer-portal/concepts/service-integration]].
- **Accessibility & HTML semantics**:
  - The visual progress bar must use an ordered list (`<ol>`) with `aria-current="step"` applied to the active step element [[kb:customer-portal/concepts/claim-tracking-flow]].
- **Locale & date formatting**:
  - Consistent use of `en-GB` date formatting (`DD/MM/YYYY`) across both submission date and handler assignment date [[kb:customer-portal/concepts/claim-tracking-flow]].

## Test strategy

| Level | What it proves | AC or contract covered |
| :--- | :--- | :--- |
| **Unit** | `STEP_FOR` correctly maps `assigned` to `"In review"`, `submitted` to `"Submitted"`, and retains downstream milestones (`settled`, `paid`). | AC-1, AC-4, AC-5 |
| **Unit** | Handler name formatting utility sanitizes upstream handler string into first name and team, strips direct phone numbers/emails, and returns fallback copy `"Assigned to a claims specialist"` when name is null or empty. | AC-2, AC-7, Edge case (Assigned with missing details) |
| **Unit** | Assignment date formatting correctly renders ISO-8601 timestamps into UK format `DD/MM/YYYY`. | AC-3 |
| **Integration** | Server component `ClaimTracker` renders expected HTML tree for an unassigned claim: highlights "Submitted", displays reported date, omits handler section. | AC-4, Edge case (Unassigned claim) |
| **Integration** | Server component `ClaimTracker` renders expected HTML tree for an assigned claim: highlights "In review", renders handler first name and team, renders formatted assignment date. | AC-1, AC-2, AC-3, [[media:d1540d7c-27ed-4c39-9dc2-67414c025c75]] |
| **Integration** | Server component `ClaimTracker` renders updated handler name upon simulated reassignment without caching stale details. | AC-6, Edge case (Handler reassignment) |
| **Contract** | Consumer contract test against `claims-management` verifying `GET /v1/claims/{id}` schema matches `Claim` interface (`id`, `status`, `reportedAt`, `handlerName`, `handlerAssignedAt`). | Contract `GET /v1/claims/{id}` [[kb:customer-portal/summaries/api-spec]] |
| **End-to-End** | Full browser workflow on `/claims/[id]`: customer navigates to assigned claim (e.g. `CLM-88213`), verifies active milestone is "In review", handler details and date are displayed, and layout wraps cleanly on 375px mobile viewport. | AC-1, AC-2, AC-3, Edge case (Mobile responsiveness) |

## Rollout and rollback

- **Rollout flags & sequencing**:
  - Upstream `claims-management` already supplies all required handler fields on `GET /v1/claims/{id}` [[kb:customer-portal/entities/claim]]. No cross-application deployment sequence or backend prerequisite is required.
  - The feature can be gated via an optional environment variable `FEATURE_HANDLER_VISIBILITY=true` in `customer-portal` if staged canary rollout is desired; otherwise, deploy as a standard frontend release.
- **Data migrations & backfills**:
  - None. `customer-portal` is a stateless Next.js application [[kb:customer-portal/index]]. Claim state and handler history are persisted in upstream domain databases.
- **Rollback procedure**:
  - In case of unexpected UI degradation or formatting defects, revert the `customer-portal` container deployment to the previous image tag.
  - Reversion immediately restores the previous `STEP_FOR` mapping (`assigned` -> `"Submitted"`) and suppresses handler display, requiring zero database rollbacks or upstream coordination.

## Risks

| Risk | Likelihood | Mitigation |
| :--- | :--- | :--- |
| **Handler PII leakage**: Upstream `handlerName` contains full surname or contact details that violate privacy rules. | Medium | Implement defensive name parsing in the UI component to extract only the first name and team name, stripping phone numbers, email addresses, or unexpected characters (AC-7). |
| **Null/missing handler data**: Claim enters `assigned` status in backend before `handlerName` is populated. | Low | Handle nullish `handlerName` gracefully by displaying fallback text `"Assigned to a claims specialist"` and omitting assignment date rather than throwing a render error. |
| **Next.js cache stagnation**: Server component caches `GET /v1/claims/{id}` response, showing stale handler after reassignment. | Low | Configure `fetch` in `lib/api.ts` with `cache: "no-store"` or explicit revalidation to guarantee real-time claim status evaluation on every page request (AC-6). |
| **Mobile viewport layout clipping**: Additional handler copy overflows or displaces progress step labels on mobile screens. | Low | Wrap handler details in a dedicated summary card below the progress bar (`<ol>`) using responsive flex/grid layouts verified down to 320px viewport widths. |

## Verification checklist

- [x] Scope matches the design: only `customer-portal` (`tw-digital/customer-portal`) has changed; `claims-management`, `claims-intake`, and `notifications-hub` remain untouched.
- [x] Contract `GET /v1/claims/{id}` consumed from `claims-management` is verified unchanged and backward-compatible.
- [x] Contract `GET /claims/{id}` additive UI changes render handler first name, team, and assignment date without breaking existing consumers.
- [x] The architecture adheres to Next.js 15 App Router Server Component patterns with API communication localized in `lib/api.ts`.
- [x] Architectural decision [[kb:customer-portal/decisions/adr-claim-tracker-step-mapping]] is superseded to document the new `assigned` -> `"In review"` milestone mapping.
- [x] Privacy guardrail is honoured: direct telephone numbers and email addresses are never displayed in the tracker UI or DOM (AC-7).
- [x] The test strategy is delivered at every named level (unit, integration, contract, end-to-end).
- [x] Mobile responsiveness is verified on small viewports (320px–375px) without horizontal clipping.
- [x] Rollback plan (container image revert) has been verified and is ready if needed.

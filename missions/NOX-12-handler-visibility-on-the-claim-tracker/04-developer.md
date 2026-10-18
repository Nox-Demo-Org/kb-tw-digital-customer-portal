---
mission: NOX-12
title: 'Handler visibility on the claim tracker'
role: developer
status: approved
version: 1
author: unknown
ai_drafted: false
approved_at: 2026-10-18T10:47:40Z
---

# Build spec: Handler visibility on the claim tracker

## Files and services touched

### `customer-portal`
- `lib/api.ts`: Configure request caching options in `getClaim` to bypass static response caching (`cache: "no-store"`) for real-time reassignment updates [[kb:customer-portal/concepts/claim-tracking-flow]].
- `lib/format.ts`: Create utility functions `formatHandlerDisplay` and `formatUkDate` for handler name sanitization, privacy filtering, and UK date formatting [[kb:customer-portal/concepts/claim-tracking-flow]].
- `app/claims/[id]/page.tsx`: Update `STEP_FOR` mapping to advance `assigned` claims to `"In review"`, and render the assigned handler name, team, and assignment date [[kb:customer-portal/decisions/adr-claim-tracker-step-mapping]], [[media:d1540d7c-27ed-4c39-9dc2-67414c025c75]].
- `__tests__/claims/page.test.tsx`: Integration tests for server component rendering across all claim statuses.
- `__tests__/lib/format.test.ts`: Unit tests for handler string parsing, privacy masking, fallback strings, and date formatting.

### Applications untouched (verified invariant)
- `claims-management`: Backend API `GET /v1/claims/{id}` and Pub/Sub event `claims.handler.assigned` remain unchanged [[kb:customer-portal/summaries/api-spec]].
- `claims-intake`: First notice of loss endpoint `POST /v1/claims` remains unchanged [[kb:customer-portal/concepts/service-integration]].
- `notifications-hub`: Event subscription rules remain unchanged [[kb:notifications-hub/entities/subscription-registry]].
- `policy-admin`: Policy routes and models remain unchanged [[kb:customer-portal/entities/policy]].
- `billing-service`: Billing routes and models remain unchanged [[kb:customer-portal/entities/billing-plan]].

---

## What to reuse

- `Claim` type in `lib/api.ts`: Reuse `{ id: string; status: string; reportedAt: string; handlerName?: string; handlerAssignedAt?: string }` [[kb:customer-portal/entities/claim]].
- `STEPS` milestone tuple in `app/claims/[id]/page.tsx`: Reuse `const STEPS = ["Submitted", "In review", "Settled", "Paid"] as const` [[kb:customer-portal/concepts/claim-tracking-flow]].
- Progress ordered list markup in `app/claims/[id]/page.tsx`: Reuse `<ol>` and `<li>` elements with `aria-current={s === current}` [[kb:customer-portal/concepts/claim-tracking-flow]].
- Base URL dictionary `BASE` in `lib/api.ts`: Reuse `BASE.claims` (`process.env.CLAIMS_URL ?? "http://claims-management/v1"`) [[kb:customer-portal/concepts/service-integration]].
- UK date locale formatting pattern: Reuse `toLocaleDateString("en-GB")` / `Intl.DateTimeFormat("en-GB")` [[kb:customer-portal/decisions/adr-claim-tracker-step-mapping]].

---

## Tasks

1. **Configure dynamic data fetching in `lib/api.ts`**
   - Update `getClaim(id: string)` to pass `{ cache: "no-store" }` in the `fetch` options:
     ```typescript
     export const getClaim = (id: string): Promise<Claim> =>
       fetch(`${BASE.claims}/claims/${id}`, { cache: "no-store" }).then((r) => r.json());
     ```
   - Prevents Next.js Server Component fetch memoization from serving stale handler metadata when reassignment occurs (AC-6) [[kb:customer-portal/concepts/claim-tracking-flow]].

2. **Add handler string and date formatting utilities in `lib/format.ts`**
   - Create `lib/format.ts`.
   - Implement `formatUkDate(isoDate?: string): string`:
     - Return empty string if `isoDate` is undefined or null.
     - Parse ISO string and return formatted date using `new Date(isoDate).toLocaleDateString("en-GB")` (AC-3).
   - Implement `formatHandlerDisplay(rawName?: string | null): string`:
     - If `rawName` is null, undefined, or whitespace-only, return `"Assigned to a claims specialist"` (Edge case: missing handler details).
     - Remove any direct phone numbers (e.g. `/\+?[0-9\s-]{7,}/g`) and email addresses (e.g. `/\S+@\S+\.\S+/g`) per privacy rules (AC-7).
     - Parse name and department format: when upstream provides `"First Last (Team, Region)"`, extract first name and team to return `"First (Team, Region)"` or `"First · Team, Region"`. If no parenthesis exists, extract first name only.
     - Trim surrounding whitespace and ensure no trailing delimiters.

3. **Update milestone mapping `STEP_FOR` in `app/claims/[id]/page.tsx`**
   - Modify the `STEP_FOR` map in `app/claims/[id]/page.tsx`:
     ```typescript
     const STEP_FOR: Record<string, (typeof STEPS)[number]> = {
       submitted: "Submitted",
       assigned: "In review",
       in_review: "In review",
       settled: "Settled",
       paid: "Paid",
     };
     ```
   - Advances active milestone from `"Submitted"` to `"In review"` whenever `claim.status === "assigned"` (AC-1) [[kb:customer-portal/decisions/adr-claim-tracker-step-mapping]], [[media:d1540d7c-27ed-4c39-9dc2-67414c025c75]].

4. **Render handler card and metadata in `app/claims/[id]/page.tsx`**
   - Import `formatHandlerDisplay` and `formatUkDate` from `@/lib/format`.
   - In `ClaimTracker`:
     - Maintain existing milestone `<ol>` list.
     - For claims where `claim.status === "submitted"`, render reported paragraph:
       `<p>Reported {formatUkDate(claim.reportedAt)}. We'll be in touch.</p>`
       Do not render any handler container or handler labels (AC-4).
     - For claims where `claim.status !== "submitted"` (e.g. `assigned`, `in_review`, `settled`, `paid`):
       - If `claim.handlerName` or `claim.status === "assigned"` / `in_review`, render a handler section:
         - Handler title/name: `<p>Claims handler: {formatHandlerDisplay(claim.handlerName)}</p>` (AC-2, AC-5, AC-7).
         - Assignment date (if `claim.handlerAssignedAt` exists): `<p>Assigned: {formatUkDate(claim.handlerAssignedAt)}</p>` (AC-3).
         - Retain reported date subtitle: `<p>Reported {formatUkDate(claim.reportedAt)}</p>`.
   - Wrap handler information in responsive block containers to avoid mobile clipping (Edge case: mobile responsiveness).

5. **Create unit and integration test suites**
   - Create `__tests__/lib/format.test.ts` covering `formatHandlerDisplay` and `formatUkDate`.
   - Create `__tests__/claims/page.test.tsx` verifying server component rendering for `submitted`, `assigned`, `in_review`, `settled`, and `paid` claims.

---

## Test plan

### Task 1 & 2: Formatting utility and fetch cache (`__tests__/lib/format.test.ts`)
- **Case 1 (UK date formatting)**: `formatUkDate("2026-09-30T10:15:00Z")` returns `"30/09/2026"`. Proves AC-3.
- **Case 2 (Handler first name and team extraction)**: `formatHandlerDisplay("Priya Raman (Home, South West)")` returns `"Priya (Home, South West)"`. Proves AC-2, AC-7.
- **Case 3 (Direct contact detail redaction)**: `formatHandlerDisplay("David Miller 07123456789 david@tidewell.example (Contents)")` returns `"David (Contents)"` and contains neither phone number nor email. Proves AC-7.
- **Case 4 (Missing handler fallback)**: `formatHandlerDisplay(null)` and `formatHandlerDisplay("")` return `"Assigned to a claims specialist"`. Proves Edge case: Assigned status with missing handler details.
- **Command**: `npm test -- __tests__/lib/format.test.ts`

### Task 3 & 4: Progress milestone and component rendering (`__tests__/claims/page.test.tsx`)
- **Case 1 (Unassigned claim milestone)**: Render `ClaimTracker` with `status: "submitted"`, `reportedAt: "2026-09-29T08:00:00Z"`. Verify `<li aria-current="true">Submitted</li>`, contains `"Reported 29/09/2026. We'll be in touch."`, and handler section is absent from the DOM. Proves AC-4, Edge case: Unassigned claim.
- **Case 2 (Assigned claim milestone advance)**: Render `ClaimTracker` with `status: "assigned"`, `handlerName: "Priya Raman (Home, South West)"`, `handlerAssignedAt: "2026-09-30T14:22:00Z"`. Verify `<li aria-current="true">In review</li>`, `<li aria-current="false">Submitted</li>`, handler text `"Priya (Home, South West)"`, and assignment date `"30/09/2026"`. Proves AC-1, AC-2, AC-3, [[media:ec737e1c-b9a3-4877-8a01-95476cbe35eb]].
- **Case 3 (Assigned claim with missing handler details)**: Render `ClaimTracker` with `status: "assigned"`, `handlerName: undefined`. Verify `<li aria-current="true">In review</li>` and fallback copy `"Assigned to a claims specialist"`. Proves Edge case: Assigned status with missing handler details.
- **Case 4 (Downstream milestone handler retention)**: Render `ClaimTracker` with `status: "paid"`, `handlerName: "David Miller (Contents)"`. Verify `<li aria-current="true">Paid</li>` and handler text `"David (Contents)"` remains present. Proves AC-5.
- **Case 5 (Reassignment rendering)**: Re-render component with new `handlerName: "Marcus Jones (Home)"` and verify updated handler name renders without stale DOM nodes. Proves AC-6.
- **Command**: `npm test -- __tests__/claims/page.test.tsx`

### End-to-end and lint verification
- Run typecheck and full test suite:
  ```bash
  npm run lint
  npx tsc --noEmit
  npm test
  ```

---

## Rollout

- **Database migrations**: None. `customer-portal` is a stateless Next.js presentation application [[kb:customer-portal/index]].
- **Feature flag**: Deploy directly via standard frontend CI/CD pipeline. No feature flag required as upstream `claims-management` already serves `handlerName` and `handlerAssignedAt` on `GET /v1/claims/{id}` [[kb:customer-portal/entities/claim]].
- **Deployment steps**:
  1. Build Docker container image for `tw-digital/customer-portal`.
  2. Deploy container to staging environment; run automated smoke tests on `/claims/CLM-88213`.
  3. Deploy container to production cluster.
- **Rollback steps**:
  1. In the event of frontend formatting regressions or rendering exceptions, redeploy previous container image tag.
  2. Rollback requires no backend intervention, schema rollback, or cross-service sync.

---

## Verification checklist

- [ ] Task 1 done: `getClaim` in `lib/api.ts` configured with `{ cache: "no-store" }`.
- [ ] Task 2 done: `lib/format.ts` created with `formatHandlerDisplay` and `formatUkDate`.
- [ ] Task 3 done: `STEP_FOR.assigned` updated to `"In review"` in `app/claims/[id]/page.tsx`.
- [ ] Task 4 done: Handler name, team, and assignment date rendered in `app/claims/[id]/page.tsx` for non-submitted claims, with unassigned claims displaying submission copy only.
- [ ] Task 5 done: Unit and integration tests written and passing in `__tests__/lib/format.test.ts` and `__tests__/claims/page.test.tsx`.
- [ ] Named tests cover AC-1 through AC-7 and edge cases.
- [ ] Reused components (`STEPS`, `BASE`, `Claim` interface, `<ol>` markup) not duplicated.
- [ ] Contracts `GET /v1/claims/{id}` and Pub/Sub event `claims.handler.assigned` in upstream services remain untouched.
- [ ] Handler privacy rule verified: no phone numbers or personal email addresses exposed in markup.
- [ ] Rollout steps verified and rollback image identified.

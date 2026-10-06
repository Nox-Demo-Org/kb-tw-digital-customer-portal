---
mission: NOX-3
title: 'Online self-service policy changes'
role: engineering
status: ai_drafted
version: 1
author: NoX
ai_drafted: true
---

# Engineering design: Online self-service policy changes

## Applications changing
- **`customer-portal` (Digital Channels team)**: Must change. The portal currently displays read-only policy data on `/policies` (`app/policies/page.tsx`) and routes all change requests to telephone support (`03000000000`) per [[kb:customer-portal/decisions/adr-policy-changes-via-phone]]. It must be updated to provide interactive self-service adjustment forms, invoke `POST /v1/policies/{id}/changes` via `lib/api.ts`, handle field validation, and render updated policy states immediately.
- **`policy-admin` (Policy team)**: Must **not** change. The service already exposes `POST /v1/policies/{id}/changes` in `PolicyController` (`com.tidewell.policy.api.PolicyController`) accepting `ChangeRequest` (`type`, `effectiveDate`, `details`) and returning `PolicyView` [[kb:policy-admin/concepts/mid-term-changes]]. The endpoint is consumed as-is.
- **`billing-service` (Billing & Payments team)**: Must **not** change in this mission. While address updates in `policy-admin` do not automatically cascade to `billing-service` for legacy plans [[kb:customer-portal/concepts/contact-centre-escalations]], billing correspondence synchronization is tracked under separate billing lifecycle initiatives.
- **`customer-identity`, `claims-intake`, `claims-management`**: Must **not** change. Authentication, FNOL intake, and claim tracking workflows remain untouched.

## Approach
1. **Supersede Phone Escalation Boundary**: Update the architectural pattern in [[kb:customer-portal/decisions/adr-policy-changes-via-phone]] to wire `POST /v1/policies/{id}/changes` into `customer-portal`.
2. **API Client Integration (`lib/api.ts`)**:
   - Add `submitPolicyChange(id: string, payload: PolicyChangeRequest): Promise<PolicyView>` communicating with `POLICY_ADMIN_URL` (`http://policy-admin/v1`) via `fetch(`${BASE.policy}/policies/${id}/changes`, { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(payload) })` [[kb:customer-portal/concepts/service-integration]].
   - Define TypeScript interfaces matching `policy-admin` DTOs:
     - `PolicyChangeRequest`: `{ type: string; effectiveDate: string; details: Record<string, string>; }`
     - `PolicyView`: `{ id: string; customerId: string; product: string; status: string; startDate: string; renewalDate: string; annualPremiumPence: number; cover: CoverLine[]; }`
     - `CoverLine`: `{ peril: string; limitPence: number; excessPence: number; }`
3. **Policy Page Workflow (`app/policies/page.tsx`)**:
   - For active policies (`status === "ACTIVE"`), replace the static `<a href="tel:03000000000">Call us to make a change</a>` link with an interactive adjustment component.
   - For cancelled, lapsed, or inactive policies, disable adjustment triggers and present an explanatory notice.
   - Implement adjustment modal/form supporting change categories (`address`, `car`/`vehicle`, `cover`) and effective date selection.
   - Provide client-side validation (e.g., non-empty address lines, valid UK postcode format, vehicle registration syntax).
   - On successful `200 OK` response from `policy-admin`, immediately update local React state with the returned `PolicyView` (including re-priced `annualPremiumPence` and updated `cover` lines) and display an on-screen confirmation notice [[kb:policy-admin/entities/policy-controller]].
   - On referral or service errors, display contextual feedback without navigating away or corrupting page state.

## Contracts affected

| Contract | Owner app | Consumers | Unchanged / additive / breaking |
| :--- | :--- | :--- | :--- |
| `POST /v1/policies/{id}/changes` | `policy-admin` | `customer-portal` | **Unchanged** (existing backend endpoint consumed by customer-portal for the first time) |
| `GET /v1/policies/{id}` | `policy-admin` | `customer-portal` | **Unchanged** |
| `GET /policies` (User route) | `customer-portal` | End user (Policyholder) | **Additive** (adds change forms and dynamic view updates to existing read-only page) |

## Must not break
- Existing policy retrieval via `GET /v1/policies/{id}` and existing `/policies` view rendering.
- Real-time rating flow and premium re-calculation in `policy-admin` via `RatingService.Price` [[kb:policy-admin/entities/policy-controller]].
- Integer-pence monetary representation across all models (`annualPremiumPence`, `limitPence`, `excessPence`) [[kb:policy-admin/summaries/api-spec]].
- Independent portal features on `/billing` (`GET /v1/plans/{policyId}`) and `/claims` (`GET /v1/claims/{id}`, `POST /v1/claims`).

## Architecture, guardrails and standards
- **Decoupled Service Ownership (ADR-0003)**: `customer-portal` interacts with policy domain data strictly through `policy-admin` HTTP REST endpoints configured via `POLICY_ADMIN_URL` [[kb:policy-admin/decisions/adr-0003-services-own-their-data]]. No direct database reads or shared persistence.
- **ADR Evolution**: Replaces the phone-routing constraint in [[kb:customer-portal/decisions/adr-policy-changes-via-phone]] with structured self-service API integration.
- **Frontend Architecture Standards**: Built within Next.js 15 App Router and React 19. Type definitions must strictly mirror `PolicyController` Java records.
- **Monetary Precision Guardrail**: Currency amounts must remain 64-bit integer pence; the portal must only format pence into decimal strings (e.g. `£XX.XX`) at presentation layer.

## Test strategy

| Level | What it proves | AC or contract covered |
| :--- | :--- | :--- |
| **Unit** | Client-side form validation (postcode format, vehicle registration, required fields) and pence-to-currency formatting | AC-2, AC-6 |
| **Integration** | `lib/api.ts` (`submitPolicyChange`) serializes `ChangeRequest`, handles `200 OK`, `400 Bad Request`, `422 Underwriting Referral`, and `500` server/network timeouts | AC-3, AC-6, Edge cases |
| **Contract** | Pact contract test between `customer-portal` (consumer) and `policy-admin` (provider) for `POST /v1/policies/{id}/changes` verifying request/response schema alignment | Contract: `POST /v1/policies/{id}/changes`, AC-3, AC-4 |
| **End-to-end** | Authenticated policyholder navigates to `/policies`, submits an address change, receives immediate confirmation, and verifies updated address and cover details on screen without phone prompt | AC-1, AC-2, AC-3, AC-4, AC-5 |

## Rollout and rollback
- **Feature Flagging**: Introduce client/server feature flag `NEXT_PUBLIC_FEATURE_POLICY_SELF_SERVICE_CHANGES` (default: `false`). When disabled, `/policies` continues rendering the existing telephone support fallback (`tel:03000000000`).
- **Rollout Sequence**:
  1. Verify `policy-admin` `POST /v1/policies/{id}/changes` endpoint availability in staging environment.
  2. Deploy `customer-portal` build containing new change components with flag set to `false`.
  3. Enable `NEXT_PUBLIC_FEATURE_POLICY_SELF_SERVICE_CHANGES=true` for internal test accounts and staging verification.
  4. Enable feature flag in production and monitor error rates on `POST /v1/policies/{id}/changes`.
- **Data Migration**: None. No schema modifications or data backfills are required.
- **Rollback**: Set `NEXT_PUBLIC_FEATURE_POLICY_SELF_SERVICE_CHANGES=false` in application environment variables to immediately revert UI back to telephone routing without code redeployment.

## Risks

| Risk | Likelihood | Mitigation |
| :--- | :--- | :--- |
| **Rating service latency or timeout**: Re-pricing via `RatingService.Price` during adjustment submission could cause HTTP request timeouts. | Low | Configure a 5-second fetch timeout with a retry prompt on failure. Display explicit failure banner advising that no changes were applied. |
| **Complex risk referral rejection**: Modifications requiring manual underwriter approval may return a referral status rather than an active policy view. | Low | Capture underwriting referral responses and display an informational status with reference number rather than a generic error. |
| **Downstream billing address desynchronisation**: Address updates applied in `policy-admin` do not automatically propagate to `billing-service` for legacy schedules [[kb:customer-portal/concepts/contact-centre-escalations]]. | High | Document boundary limitation in release notes; policyholder correspondence for policy schedules reflects immediately, and billing sync is tracked under platform backlog. |

## Verification checklist
- [ ] Scope matches the design (only `customer-portal` modified; `policy-admin`, `billing-service`, `claims-intake`, and `customer-identity` unchanged).
- [ ] `POST /v1/policies/{id}/changes` contract is consumed without breaking changes or modifications to backend signature.
- [ ] Layering rules and ADR-0003 are honoured via clean REST calls through `POLICY_ADMIN_URL`.
- [ ] All monetary fields are preserved as integer pence until presentation formatting.
- [ ] Test strategy is fully delivered across unit, integration, consumer-provider contract, and end-to-end levels.
- [ ] Feature flag `NEXT_PUBLIC_FEATURE_POLICY_SELF_SERVICE_CHANGES` allows immediate zero-downtime rollback to contact centre phone prompt.

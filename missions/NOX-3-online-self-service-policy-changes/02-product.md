---
mission: NOX-3
title: 'Online self-service policy changes'
role: product
status: approved
version: 1
author: dev
ai_drafted: false
approved_at: 2026-10-06T02:41:45Z
---

# Product spec: Online self-service policy changes

## Goal
Enable policyholders to submit mid-term policy adjustments (such as address updates, vehicle changes, and cover alterations) directly within the customer portal, providing instant confirmation and updating policy records without requiring a call to the contact centre.

## User stories
- **US-1 (Address update)**: As a policyholder, I want to update my home or mailing address on my active policy online, so that my policy records remain accurate without waiting in a phone queue.
- **US-2 (Vehicle change)**: As a motor insurance policyholder, I want to update my registered vehicle details online, so that my cover transfers to my new vehicle immediately.
- **US-3 (Cover adjustment)**: As a policyholder, I want to adjust my cover options and policy add-ons online, so that my policy reflects my changing insurance needs without calling customer support.

## Acceptance criteria
- **AC-1 (Change policy entry point)**: Given an authenticated policyholder viewing an active policy on the policy details screen (`/policies`), When the policy details load, Then the customer is presented with an action to "Change policy" or "Make an adjustment" rather than solely being prompted to call telephone support [[kb:customer-portal/decisions/adr-policy-changes-via-phone]].
- **AC-2 (Self-service change flow selection)**: Given a policyholder initiating a policy change, When they choose to make an amendment, Then they can select from supported change categories (address change, vehicle details change, or cover level adjustment) and specify the effective date for the amendment.
- **AC-3 (Address modification submission)**: Given a policyholder submitting a new residential address, When they enter the new address details and confirm submission, Then the updated address is saved to the policy record and an immediate on-screen confirmation with an adjustment summary is displayed.
- **AC-4 (Immediate policy view update)**: Given a successfully confirmed policy change, When the policyholder returns to or refreshes their policy view (`/policies`), Then the updated policy details and cover lines reflect the new information immediately without manual agent intervention [[kb:policy-admin/concepts/mid-term-changes]].
- **AC-5 (Removal of forced phone prompt for supported changes)**: Given an active policy eligible for self-service adjustment, When the policyholder views their policy options, Then they are not blocked or instructed that contact centre calling (`03000000000`) is mandatory for standard amendments [[kb:customer-portal/concepts/contact-centre-escalations]].
- **AC-6 (Validation and error handling)**: Given a policyholder entering incomplete or invalid adjustment details (such as a missing postcode or invalid registration format), When they attempt to submit, Then inline validation highlights the invalid fields and prevents submission until corrected.

## Edge cases
- **Cancelled or expired policy**: If a customer views a policy that is cancelled, lapsed, or expired, the self-service amendment option is disabled, and an informational banner explains that adjustments cannot be made on inactive policies.
- **Future-dated effective date**: When a customer selects an effective date in the future, the submission is confirmed, and the portal displays the pending effective date alongside the currently active cover.
- **Underwriting referral or unrated risk *(from the map)***: If a policy amendment introduces a risk profile requiring manual underwriter review or exceeds automated rating limits, the portal displays a clear status indicating the change is under review with a reference number rather than failing silently.
- **Billing address synchronisation *(from the map)***: When an address update is submitted, policy records update immediately; downstream billing notices must also reflect the updated correspondence address [[kb:customer-portal/concepts/contact-centre-escalations]].
- **Network or service timeout during adjustment**: If the backend adjustment service is temporarily unreachable during submission, the portal displays a resilient error message advising the customer that the change could not be completed and no charges or changes were applied, offering a retry option.

## Out of scope
- Full policy cancellation self-service (cancellations remain handled via standard renewal and cancellation workflows).
- Submitting mid-term claim reports through the policy adjustment screen (claims continue through the dedicated digital claims flow [[kb:customer-portal/concepts/claim-tracking-flow]]).
- Direct self-service adjustments for commercial lines or non-standard bespoke underwriting policies not supported by automated rating.

## Success metric
- **Call volume reduction**: 100% of routine policy adjustments currently routed to phone support (`03000000000`) [[kb:customer-portal/decisions/adr-policy-changes-via-phone]] → at least 65% completed self-service online *(suggestion)*, measured via contact centre call categorization and portal analytics 30 days post-launch.
- **Task completion time**: Average time for a customer to complete a policy amendment drops from 12+ minutes (phone wait and agent handling) to under 3 minutes self-service *(suggestion)*.

## Priority
**P1**: High impact on operational costs and customer satisfaction. Currently, 100% of policy updates require manual phone agent interaction, creating long wait times, operational bottlenecks, and address desynchronisation risks.

## Verification checklist
- [ ] AC-1: Action to change/amend policy is available on `/policies` instead of solely showing contact centre phone prompt.
- [ ] AC-2: Policyholder can select amendment type (address, vehicle, cover) and effective date.
- [ ] AC-3: Submitting an address change displays an immediate confirmation with adjustment details.
- [ ] AC-4: Policy view updates immediately after successful submission with new details and cover lines.
- [ ] AC-5: Phone routing text is replaced with direct online change workflows for supported adjustments.
- [ ] AC-6: Form validation prevents submission of invalid or incomplete data with clear inline error messages.
- [ ] Edge case: Inactive, cancelled, or expired policies correctly disable change workflows.
- [ ] Edge case: Future-dated effective dates show clear pending status.
- [ ] Edge case: Underwriting referral flows surface appropriate status and reference number.
- [ ] Edge case: Address adjustments synchronise correctly across policy and billing communications.
- [ ] Edge case: Network failure during submission displays friendly retry prompt without corrupting policy state.
- [ ] Read success metrics: Contact centre amendment call volume drop and portal self-service completion rate at 30 days post-launch.

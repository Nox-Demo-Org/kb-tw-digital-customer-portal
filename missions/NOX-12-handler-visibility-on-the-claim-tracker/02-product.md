---
mission: NOX-12
title: 'Handler visibility on the claim tracker'
role: product
status: approved
version: 2
author: unknown
ai_drafted: false
approved_at: 2026-10-18T10:39:51Z
---

# Product spec: Handler visibility on the claim tracker

## Goal
Provide policyholders with immediate visibility into their assigned claims specialist and claim progression beyond "Submitted" on the claim tracking screen as soon as a claim is allocated internally, eliminating the 4.6-day silence period that currently prompts 41% of claimants to call the contact centre [[kb:customer-portal/decisions/adr-claim-tracker-step-mapping]], [[kb:customer-portal/concepts/contact-centre-escalations]].

## User stories
- **As a policyholder tracking an open claim**, I want to see the claim progress beyond "Submitted" as soon as an internal handler is assigned, so that I know my file is actively being worked on without having to telephone customer support [[kb:customer-portal/decisions/adr-claim-tracker-step-mapping]].
- **As a policyholder tracking an open claim**, I want to see the name of my assigned claims handler and the date they picked up my file, so that I know who is accountable for my claim and can verify their identity when they contact me [[kb:customer-portal/concepts/claim-tracking-flow]].
- **As a contact centre agent**, I want the customer portal claim tracker to accurately mirror handler allocation, so that customers who have already spoken with a claims handler do not call in confused by an out-of-date status [[kb:customer-portal/concepts/contact-centre-escalations]].

## Acceptance criteria

- **AC-1 (Progress milestone advancement on assignment)**: Given a policyholder has submitted an insurance claim that has been internally allocated to a claims handler, When the policyholder views the claim tracking screen in customer-portal, Then the visual progress indicator shows the claim has advanced to "In review" rather than remaining on "Submitted" [[kb:customer-portal/decisions/adr-claim-tracker-step-mapping]], [[media:ec737e1c-b9a3-4877-8a01-95476cbe35eb]], [[media:d1540d7c-27ed-4c39-9dc2-67414c025c75]].
- **AC-2 (Display of handler name and team)**: Given a claim has been allocated to a claims specialist, When the policyholder views the claim tracking screen, Then the screen displays the assigned handler's first name and team alongside the claim status information [[kb:customer-portal/decisions/adr-claim-tracker-step-mapping]], [[media:d1540d7c-27ed-4c39-9dc2-67414c025c75]].
- **AC-3 (Display of assignment date)**: Given a claim has an assigned handler with a recorded assignment timestamp, When the policyholder views the claim tracking screen, Then the screen displays the date the handler was assigned in UK date format (`DD/MM/YYYY`) [[kb:customer-portal/concepts/claim-tracking-flow]].
- **AC-4 (Unassigned claim milestone state)**: Given a newly submitted claim has not yet been allocated to a handler, When the policyholder views the claim tracking screen, Then the progress tracker highlights "Submitted" as the current milestone, displays the original reported date, and does not show any handler details [[kb:customer-portal/concepts/claim-tracking-flow]].
- **AC-5 (Handler visibility across downstream milestones)**: Given a claim has progressed further in its lifecycle to "In review", "Settled", or "Paid" with an assigned handler, When the policyholder views the claim tracking screen, Then the active progress step reflects the current claim milestone and the assigned handler's name remains visible [[kb:customer-portal/concepts/claim-tracking-flow]].
- **AC-6 (Handler reassignment update)**: Given a claim is reassigned internally to a new handler, When the policyholder views or refreshes the claim tracking screen, Then the display updates to show the new handler's name and latest assignment date.
- **AC-7 (Handler privacy and direct contact details)**: Given a claim has an assigned claims handler, When the policyholder views the claim tracking screen, Then the screen displays the handler's first name and team only, and never displays direct contact details (such as a direct telephone number or personal email address).

### Open questions for the product owner
1. **Progress milestone taxonomy**: Should handler allocation advance the current 4-step progress tracker directly to "In review", or should a dedicated 5th milestone step ("Assigned") be added? *(suggestion: Advance the current 4-step tracker to "In review" upon handler allocation to keep the customer journey clean and linear while immediately eliminating the stuck "Submitted" perception [[kb:customer-portal/decisions/adr-claim-tracker-step-mapping]], [[media:d1540d7c-27ed-4c39-9dc2-67414c025c75]]).*

## Edge cases
- **Unassigned claim during initial intake *(from the map)***: A claim has been submitted but claims-management has not yet assigned a handler (median duration 1.3 working days [[kb:customer-portal/concepts/contact-centre-escalations]]). The tracker must display the "Submitted" milestone with the submission date and the reassuring copy "Reported DD/MM/YYYY. We'll be in touch." without rendering empty handler containers [[kb:customer-portal/concepts/claim-tracking-flow]].
- **Telephony desynchronisation after handler phone contact *(from the map)***: A handler contacts the customer by telephone shortly after assignment (e.g. bug TWCLM-2 [[kb:customer-portal/concepts/claim-tracking-flow]]). When the customer logs into customer-portal afterwards, the tracker must show the handler's name and the advanced milestone, reassuring the customer that the phone contact was legitimate [[kb:customer-portal/decisions/adr-claim-tracker-step-mapping]], [[media:d1540d7c-27ed-4c39-9dc2-67414c025c75]].
- **Assigned status with missing handler details**: If upstream data marks a claim as assigned but the handler name is blank or null, the tracker must advance past "Submitted" to "In review" and display a generic message such as "Assigned to a claims specialist" rather than displaying an empty string or failing to render.
- **Handler reassignment**: When a claim is transferred between handlers internally, the customer-facing tracker must reflect the current handler and new assignment timestamp on the next page view without caching stale handler details.
- **Settled or completed claims**: For claims that have reached "Settled" or "Paid", handler information must remain visible on the claim summary for customer records.
- **Mobile responsiveness**: On narrow mobile screens, handler details and assignment dates must wrap cleanly beneath the progress milestone list without horizontal clipping or obscuring step labels.
- **Upstream service unavailability**: If claim details cannot be retrieved from claims-management, the portal must display the standard friendly error page rather than an empty tracker.

## Out of scope
- Automated SMS or text messaging when a handler is assigned (tracked under Jira TWCLM-3 and TWCLM-5 for notifications-hub [[kb:customer-portal/concepts/claim-tracking-flow]]).
- Direct in-portal messaging, live chat, or email composition to the assigned handler.
- Customer document and photographic evidence uploads on the tracking screen (scheduled under Digital Claims v2 discovery [[kb:customer-portal/concepts/claim-tracking-flow]]).
- Policy excess visibility on the claim tracker (tracked under Jira TWCLM-4 [[kb:customer-portal/concepts/claim-tracking-flow]]).
- Modifying backend handler allocation logic, workload routing, or triage algorithms in claims-management.

## Success metric
- **"Where is my claim?" call volume**: Baseline is 41% of online claimants calling the contact centre within 5 days of submission [[kb:customer-portal/concepts/contact-centre-escalations]] → Target: under 20% of online claims *(suggestion)*.
- **Operational support cost reduction**: Baseline contact centre cost is £6.80 per status inquiry call [[kb:customer-portal/concepts/contact-centre-escalations]] → Target: at least a 50% decrease in overall status inquiry call expenditure *(suggestion)*.
- **Claims customer satisfaction (CSAT)**: Baseline is 3.4 out of 5.0 [[kb:customer-portal/concepts/claim-tracking-flow]] → Target: 4.2 out of 5.0 *(suggestion)*.
- **Data source & cadence**: Telephony call disposition logs ("Where is my claim?" inquiry category) and monthly claims CSAT survey reports from the contact centre and portal analytics. Read at 30 days and 60 days post-deployment *(suggestion)*.

## Priority
**P1 (High)**

### Why
- 64% of claims are now submitted online, up from 51% last year [[kb:customer-portal/concepts/claim-tracking-flow]].
- The current 4.6-day tracker stagnation drives 41% of online claimants to call the contact centre, costing £6.80 per call in avoidable overhead [[kb:customer-portal/concepts/contact-centre-escalations]].
- Lack of visibility into whether a claim has been picked up is the single largest detractor dragging claims satisfaction down to 3.4 out of 5 [[kb:customer-portal/concepts/claim-tracking-flow]].
- Handlers calling customers directly while the portal remains on "Submitted" creates customer confusion and unnecessary verification calls [[kb:customer-portal/decisions/adr-claim-tracker-step-mapping]], [[media:ec737e1c-b9a3-4877-8a01-95476cbe35eb]].

### Cost of doing nothing
Continued monthly expenditure of thousands of pounds on avoidable status inquiry calls, reduced contact centre capacity for complex and emergency claims, and persistent low customer satisfaction ratings.

### Stakeholder communications
- **Contact Centre Operations**: Notify supervisors and frontline agents of the new handler visibility and updated tracker milestones so scripts can reflect that customers now see their handler's details online.
- **Claims Operations**: Brief claims handlers that their name and assignment date will be visible to policyholders immediately upon assignment.
- **Customer Support & Communications**: Update help articles and customer FAQs explaining the claim tracking milestones on the website.

## Verification checklist

- [ ] AC-1: Verify that an internally assigned claim displays the active milestone advanced to "In review" rather than "Submitted".
- [ ] AC-2: Verify that the assigned handler's first name and team are clearly displayed on the claim tracking screen.
- [ ] AC-3: Verify that the handler assignment date appears in UK date format (`DD/MM/YYYY`) on the claim tracking screen.
- [ ] AC-4: Verify that an unassigned submitted claim remains on the "Submitted" milestone with the reported date and shows no handler information.
- [ ] AC-5: Verify that claims in subsequent states ("In review", "Settled", "Paid") continue to display the assigned handler's name alongside their current milestone.
- [ ] AC-6: Verify that reassigning a claim internally immediately updates the handler name and assignment date on the next portal view.
- [ ] AC-7: Verify that the claim tracking screen displays only the handler's first name and team, and never displays direct contact details (such as direct telephone number or personal email address).
- [ ] Edge case (Unassigned claim during initial intake): Verify that claims in submitted status render without empty handler containers or formatting errors.
- [ ] Edge case (Telephony desynchronisation): Verify that a customer whose claim was assigned and who received phone contact sees the handler's name and advanced status upon logging into the portal.
- [ ] Edge case (Assigned status with missing handler details): Verify that a fallback message ("Assigned to a claims specialist") appears if an assigned claim has no handler name.
- [ ] Edge case (Handler reassignment): Verify that stale handler data is not cached when a claim changes handlers.
- [ ] Edge case (Settled or completed claims): Verify that historical handler information remains visible once a claim reaches settled or paid status.
- [ ] Edge case (Mobile responsiveness): Verify that handler details and progress milestones wrap correctly on mobile screens without truncation.
- [ ] Edge case (Upstream service unavailability): Verify that upstream service retrieval failures trigger standard error handling rather than an incomplete tracker screen.
- [ ] Success metric reading: Review contact centre inquiry call volume ("Where is my claim?") and CSAT scores at 30 and 60 days post-launch against the 41% call baseline and 3.4/5.0 CSAT baseline.

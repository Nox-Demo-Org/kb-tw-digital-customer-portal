---
mission: NOX-12
title: 'Handler visibility on the claim tracker'
role: business
status: draft
version: 1
author: Maya Shah
ai_drafted: false
---

# Business requirement: Handler visibility on the claim tracker

## The request
"Customers keep calling to ask where their claim is. Can they just see who's handling it?"

## Problem
When a customer files a claim on our website, our claims team normally assigns a claims handler to the case within an average of 1.3 working days. However, the online claim tracker does not show this progress. Instead, the tracker stays stuck on "Submitted" for an average of 4.6 working days until formal assessment begins, giving no indication that anyone has picked up the file.

This 4-to-5-day silence leaves customers feeling ignored. Today, 41% of policyholders who file online call our contact centre within five days just to ask "Where is my claim?". Each status inquiry call costs £6.80 in staff time and ties up phone lines for customers with urgent emergencies. Even when a handler reaches out to a customer directly by phone, the website still reads "Submitted" when the customer logs back in, prompting confused follow-up calls to verify whether the case is genuine and moving forward. Customer satisfaction across claims has dropped to 3.4 out of 5 as a direct result of this blind spot.

## Who is affected
- **Policyholders (customers)**: Roughly 41% of customers who file online experience unnecessary worry during the first week of their claim because the website makes it look like nothing has happened.
- **Contact centre agents**: Staff spend hours each day handling avoidable status calls and reassuring anxious policyholders instead of helping customers in urgent distress.
- **Claims handlers**: Handlers deal with confused or frustrated customers who assume their claim has been sitting unattended on a pile since submission.

## What should change
- When a claim is allocated to an internal handler, the online claim tracker must immediately advance beyond "Submitted" so the customer sees real progress.
- The claim tracker must display the assigned handler's name and team so policyholders know exactly who is responsible for their claim.
- Policyholders should be able to log in at any time, see that their claim is actively with a named team member, and feel confident without needing to telephone the contact centre.

## What "done" looks like
- The percentage of online claimants who call the contact centre asking "Where is my claim?" within five days drops substantially from the current 41% baseline.
- Contact centre reporting shows a clear decline in status-inquiry call volumes, lowering avoidable operational support costs.
- Claims customer satisfaction ratings recover from 3.4 out of 5 as customers gain clear visibility into who is managing their file.
- Any business manager logging into a customer's online claim view after handler allocation can immediately see the updated status and the handler's name.

## Examples
### Example 1: Home water damage claim
- **Before**: Sarah reports a water leak on Tuesday, 29 September 2026. Priya Raman in the Home Claims team is assigned to the file on Wednesday, 30 September. Sarah checks the website on Thursday, 1 October, sees the status is still "Submitted", assumes nobody has looked at it, and calls the contact centre on Friday morning. The call takes 8 minutes and costs £6.80 simply to tell Sarah that Priya has her file.
- **After**: Sarah checks the website on Wednesday afternoon, 30 September. The tracker has moved forward to show her claim is being handled by Priya Raman (Home Claims). Knowing someone is on the case, Sarah waits for Priya's scheduled follow-up without calling the contact centre.

### Example 2: Handler calls customer directly
- **Before**: David in Contents Claims is assigned to Mark's theft claim on 12 October and calls Mark to introduce himself. Later that evening, Mark logs into the website to check what documents he needs. The tracker still says "Submitted" with no mention of David or any assignment. Wondering whether the phone call was legitimate or if the system lost his details, Mark rings customer support the following morning to verify.
- **After**: Mark logs in after speaking with David and sees his claim tracker showing David Miller (Contents Claims) as his handler. Reassured that the system reflects reality, Mark uploads his receipts without placing a verification call.

## Verification checklist
- [ ] Log in to the customer portal with a test claim that has been allocated to a claims specialist, and confirm the progress tracker shows that the claim has progressed beyond "Submitted".
- [ ] Confirm that the assigned claims specialist's name and department are clearly displayed on the customer's claim page.
- [ ] Review the monthly contact centre inquiry report and verify that the share of "Where is my claim?" status calls within five days of submission decreases from the 41% baseline.
- [ ] Confirm that the original request is satisfied: "Customers keep calling to ask where their claim is. Can they just see who's handling it?"

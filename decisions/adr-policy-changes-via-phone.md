---
type: Architecture Decision
title: 'ADR: Route Policy Changes to Contact Centre'
description: Policyholders frequently need to update critical policy details, such as their address, insured vehicle ("car"), or cover levels.
resource: https://github.com/Nox-Demo-Org/kb-tw-digital-customer-portal/blob/main/decisions/adr-policy-changes-via-phone.md
tags:
- customer-portal
- decisions
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:42:58Z'
---

# ADR: Route Policy Changes to Contact Centre

## Status
Accepted

## Context
In the [[entities/policy|customer portal]], customers can view their active insurance policies rendered on the `/policies` route (`app/policies/page.tsx`), which retrieves policy data from the upstream `policy-admin` service via `GET /v1/policies/{id}` (see [[concepts/service-integration]]).

Policyholders frequently need to update critical policy details, such as their address, insured vehicle ("car"), or cover levels. While the upstream `policy-admin` service supports policy modifications (such as `POST /v1/policies/{id}/changes`), implementing self-service mid-term adjustments within `customer-portal` requires handling complex validation rules, dynamic underwriting reassessments, and potential premium recalculations.

## Decision
We decided not to implement or wire up self-service policy modification endpoints (such as `POST /v1/policies/{id}/changes`) within `customer-portal`.

Instead:
- The policy view (`app/policies/page.tsx`) only provides read-only policy details.
- All requests to modify policy details (address, car, or cover) are explicitly routed offline to the contact centre via a telephone link (`<a href="tel:03000000000">Call us to make a change</a>`).
- Contact centre staff handle policy updates directly in `policy-admin`.

## Consequences
- **Reduced Frontend Complexity**: The portal avoids implementing and maintaining complex multi-step forms, validation logic, and error handling for mid-term adjustments across diverse policy types.
- **Limited Self-Service Capability**: Policyholders cannot update address, vehicle, or cover details directly online, creating customer friction for routine account updates.
- **Increased Contact Centre Load**: Routine policy modification requests generate phone calls to the contact centre (`#tw-contact-centre`), requiring agent intervention as documented in [[concepts/contact-centre-escalations]].
- **Minimal API Surface**: The portal integration with `policy-admin` remains limited to read-only queries (`GET /v1/policies/{id}`), as outlined in the [[summaries/api-spec]].

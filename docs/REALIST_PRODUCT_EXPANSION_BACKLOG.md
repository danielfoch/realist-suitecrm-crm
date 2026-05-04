# Realist.ca Product Expansion Backlog

These are product requirements that extend Realist from a deal-analysis site into an execution platform: analyze, compare, inspect, finance, and hand off deals.

## 1. Book/send a home inspector

### Product job

Let a user move from analysis to diligence without leaving Realist.ca.

User story:

> As an investor analyzing a property, I want to book or send an inspector so I can quickly validate the deal before wasting time or losing the opportunity.

### User-facing flow

1. User is on a listing/deal-analysis page.
2. CTA appears when appropriate: `Send an inspector` / `Book inspection`.
3. User picks:
   - standard home inspection
   - investor/rental inspection
   - plex/multi-unit inspection
   - pre-offer walkthrough/check
4. Checkout cart opens with a default price, e.g. `$500`.
5. User enters property access details, preferred dates/times, contact info, and notes.
6. Payment is authorized/captured depending on fulfillment model.
7. CRM creates inspection request, task/opportunity, and timeline event.
8. Inspector/contractor receives assignment or request to accept.
9. User sees status: requested, assigned, scheduled, completed, report uploaded.

### Backend objects

| Object | Purpose |
|---|---|
| `inspection_request` | User order/request tied to listing, contact, property, payment, and status. |
| `inspector_profile` | Contractor/certified inspector account profile. |
| `inspection_assignment` | Which inspector is assigned, schedule, acceptance, completion. |
| `inspection_report` | Uploaded report, photos, summary, flags. |
| `checkout_order` | Payment/cart object and provider transaction ids. |

### Inspector onboarding

Add an onboarding path for service providers. During signup, user can choose account type:

- Investor/buyer
- Realtor/agent
- Mortgage/financing partner
- Contractor
- Certified home inspector

Inspector/contractor onboarding fields:

- name/company
- service area
- certification/license details if applicable
- insurance/E&O info
- service types
- availability
- pricing/default fee
- payout/payment details
- phone/email
- accepted terms
- verification status

Status model:

```text
applied -> needs_review -> approved -> active -> suspended
```

### CRM integration

Events:

- `inspection.checkout_started`
- `inspection.order_created`
- `inspection.payment_authorized`
- `inspection.requested`
- `inspection.assigned`
- `inspection.completed`
- `inspector.signup_started`
- `inspector.approved`

CRM actions:

- Create/update contact.
- Create inspection opportunity/task.
- Assign owner/ops queue.
- Add note to listing/deal timeline.
- Draft message to user/inspector where permitted.

### MVP constraint

Start with checkout + request capture + manual assignment. Do not overbuild marketplace dispatch before demand exists.

## 2. Deal-analysis flywheel

### Product job

Make analyzing one deal naturally lead to analyzing the next one.

User story:

> As an investor, I want to apply the same rent/vacancy/expense/financing assumptions to similar listings so I can screen many deals quickly.

### Core loop

```text
Analyze deal
  -> save assumptions
  -> show similar listings
  -> one-click "apply assumptions"
  -> compare outputs
  -> save shortlist
  -> repeat
```

### Required features

- `Apply assumptions to another deal` button.
- Similar-listing recommendations based on:
  - market/neighbourhood
  - property type
  - price band
  - bedrooms/bathrooms
  - estimated unit count
  - rent profile
  - days on market / price change
  - investor score similarity
- Scenario versioning:
  - preserve original assumptions
  - cloned scenario references source scenario
  - user can edit assumptions after cloning
- Comparison table:
  - cap rate
  - cash flow
  - cash required
  - rent estimate
  - investor score
  - risk flags
- Shortlist action:
  - save top deals
  - ask for financing help
  - ask for inspection
  - ask for agent review

### Events

- `analysis.assumptions_saved`
- `analysis.assumptions_applied_to_listing`
- `analysis.similar_listing_clicked`
- `analysis.comparison_created`
- `deal.shortlisted`

### CRM integration

Repeated analysis is a strong behavioural signal, but still not permission for generic nurture. Next-best-action can create:

- financing handoff task
- deal review task
- saved-search alert
- leaderboard progress update
- inspection CTA if user shows property-level intent

## 3. Better plex detection

### Product job

Find duplexes/triplexes/fourplexes even when listings do not cleanly label them as `duplex`, `triplex`, `4plex`, or `multi-family`.

### Detection hierarchy

1. Explicit structured property type/tags:
   - duplex
   - triplex
   - fourplex/4plex
   - multiplex
   - multi-family
   - legal basement apartment
   - secondary suite
2. Unit/count fields if available:
   - number of units
   - number of kitchens
   - number of meters
   - number of entrances
3. Listing text/NLP cues:
   - "two kitchens"
   - "3 kitchens"
   - "separate entrance"
   - "basement apartment"
   - "in-law suite"
   - "legal duplex"
   - "non-conforming duplex"
   - "upper/lower units"
   - "separately metered"
   - "vacant possession"
4. Image/vision cues later:
   - multiple mailboxes
   - separate entrances
   - multiple kitchens in photos

### Kitchen-count rule

If explicit plex tags are missing, default to number of kitchens as the strongest proxy for unit count.

Examples:

- `kitchens >= 2` -> likely duplex / house with secondary suite.
- `kitchens >= 3` -> likely triplex or larger.
- `kitchens >= 4` -> likely fourplex/multiplex.

Always label this as `estimated_unit_count` or `possible_plex`, not guaranteed legal unit count.

### Scoring model

Add fields:

| Field | Meaning |
|---|---|
| `explicit_plex_type` | Structured property type if available. |
| `kitchen_count` | Number of kitchens from listing data/photos/text. |
| `estimated_unit_count` | Best guess from tags + kitchens + text. |
| `plex_confidence_score` | 0-100 confidence. |
| `plex_detection_reason` | Human-readable explanation. |
| `legal_status_known` | yes/no/unknown. |
| `needs_manual_review` | true when ambiguity is high. |

### UI behaviour

Show badges:

- `Possible duplex`
- `Possible triplex`
- `2 kitchens detected`
- `Separate entrance mentioned`
- `Legal status unknown`

Do not misrepresent legality. The wording should be careful:

> "Possible 2-unit setup based on 2 kitchens and separate entrance language. Verify legal status before underwriting."

### Events

- `plex_candidate.viewed`
- `plex_filter.used`
- `plex_detection_feedback.submitted`

### CRM/product value

This creates a differentiated investor search feature: Realist finds hidden plex candidates that normal listing filters miss.

## Implementation priority

1. Add shared event names and backend schemas.
2. Add deal-analysis flywheel UI because it directly increases engagement and account value.
3. Add plex detection heuristic from existing listing fields/text.
4. Add inspection checkout as request-capture/manual fulfillment first.
5. Add inspector onboarding once first request flow exists.
6. Later: automate inspector dispatch, payouts, availability, and uploaded reports.

# Realist CRM Architecture

This fork is the base for a Realist.ca-owned CRM that can replace GoHighLevel without recreating GHL's lock-in. SuiteCRM remains the local source of truth. Email, SMS, calendar, podcast feeds, and Realist.ca webhooks are replaceable rails around it.

## Product job

Realist CRM should turn Realist.ca behaviour into useful, reasoned follow-up:

- Keep users returning to Realist.ca when their own behaviour shows intent.
- Move listeners/readers/viewers toward the next useful action: listen to the right podcast episode, read the right market page, ask for help finding a deal, or ask for help financing a deal.
- Send weekly leaderboard emails from Realist.ca webhook payloads so users can see their activity, compare progress, and return to the site.
- Give OpenClaw safe, auditable actions so agents can help without spraying the CRM.

Non-goal: generic drip nurture. Every outreach must have a behavioural reason.

## Core principles

1. **SuiteCRM is source of truth** for contacts, leads, accounts, opportunities, tasks, consent, suppression, owner assignment, and audit history.
2. **Realist.ca owns behavioural events** and sends normalized webhook payloads into the CRM layer.
3. **OpenClaw proposes actions, the CRM enforces rules.** Agent output is never a direct send.
4. **Rails are replaceable.** Twilio/SendGrid/Gmail/Google Calendar/GHL can be swapped behind adapters.
5. **Reason-first outreach.** A message exists because of a concrete event: saved search, deal calculator use, leaderboard milestone, podcast topic viewed, financing page return, etc.
6. **Dry-run before write.** Agents can preview effects before a confirmed mutation.
7. **Idempotency everywhere.** Replayed webhooks or duplicate agent calls must not duplicate contacts, notes, emails, SMS, or tasks.

## Target system map

```text
Realist.ca
  webhooks: user.created, search.saved, calculator.used, podcast.played,
            deal.favourited, financing.intent, leaderboard.weekly.ready
      |
      v
Realist Agent API (this repo: /realist-agent-api scaffold)
  - webhook intake
  - idempotency validation
  - event normalization
  - action policy checks
  - dry-run/confirmed-write split
  - audit log
      |
      v
SuiteCRM 7.15.1
  Contacts / Leads / Accounts / Opportunities / Tasks / Calls / Meetings / Notes
  custom fields + custom modules added over time
      |
      v
Adapters
  SMS provider | email provider | calendar provider | Realist.ca links | podcast feed
```

## Entity model

Use existing SuiteCRM modules first. Add custom modules only when the concept cannot be represented cleanly.

### Existing SuiteCRM modules

| Concept | SuiteCRM module | Notes |
|---|---|---|
| Realist user | Contacts or Leads | Contact when known person; Lead when incomplete/anonymous-to-known transition. |
| Investor/buyer profile | Contacts custom fields | Budget, target markets, asset type, financing stage, time horizon. |
| Deal/help request | Opportunities | Buying, financing, listing, JV, mortgage/referral opportunities. |
| Follow-up job | Tasks | Human handoff, callback, review saved deal, financing intro. |
| Call/meeting | Calls / Meetings | Calendar adapter should create/update these. |
| Behaviour notes | Notes | Human-readable event summaries tied to contact/lead. |
| Owner assignment | Assigned user | CRM owner remains authoritative. |

### Custom modules/records to add later

| Entity | Purpose | Minimum fields |
|---|---|---|
| `realist_behavior_event` | Immutable event log from Realist.ca | `event_id`, `event_type`, `contact_id`, `occurred_at`, `payload_hash`, `reason_summary`. |
| `realist_agent_action` | Agent-safe action request ledger | `action_id`, `idempotency_key`, `actor`, `dry_run`, `status`, `policy_result`, `target_record`. |
| `realist_outreach_decision` | Why a message/task was allowed or blocked | `decision_id`, `behaviour_reason`, `cooldown_result`, `suppression_result`, `recommended_channel`. |
| `realist_leaderboard_snapshot` | Weekly stats email source | `week_start`, `week_end`, `score`, `rank`, `stats_json`, `cta_url`. |
| `realist_content_recommendation` | Podcast/article/deal recommendation | `content_type`, `content_id`, `reason`, `url`, `expires_at`. |

## Webhook events from Realist.ca

All events must include an immutable `event_id`, timestamp, user/contact identity, and source URL when applicable.

```json
{
  "event_id": "evt_01J...",
  "event_type": "leaderboard.weekly.ready",
  "occurred_at": "2026-05-03T13:00:00Z",
  "realist_user_id": "usr_123",
  "email": "investor@example.com",
  "idempotency_key": "realist:leaderboard.weekly.ready:usr_123:2026-W18",
  "reason": "Weekly investor leaderboard is ready after 4 calculator runs and 2 saved deals.",
  "payload": {
    "week": "2026-W18",
    "rank": 42,
    "stats": {
      "calculator_runs": 4,
      "saved_deals": 2,
      "podcast_minutes": 37
    },
    "return_url": "https://realist.ca/leaderboard?week=2026-W18"
  }
}
```

### Initial event catalog

| Event | Behavioural reason | Default CRM action |
|---|---|---|
| `user.created` | New known Realist.ca account | Search-before-create lead/contact, add source, create first note. |
| `profile.updated` | User gave budget/market/strategy | Update custom fields, re-score lead, maybe create task. |
| `search.saved` | Explicit market/property intent | Save criteria, recommend relevant listings/content if cooldown allows. |
| `calculator.used` | Deal analysis activity | Add note; if repeated/high-quality, prompt for help underwriting or financing. |
| `deal.favourited` | Property-level intent | Create opportunity candidate or task for human review. |
| `podcast.played` | Topic interest | Recommend a matching Realist.ca resource only if useful and fresh. |
| `financing.intent` | Mortgage/private lending/refi signal | Create financing opportunity/task; route to approved partner workflow. |
| `leaderboard.weekly.ready` | Weekly engagement summary available | Queue leaderboard email with stats and return CTA. |
| `help.requested` | Direct request for help | Immediate task/opportunity; can bypass normal nurture cooldown, not consent/suppression. |

## Workflows

### 1. Search before create

1. Intake identifies by `realist_user_id`, email, phone, then normalized name.
2. If exact match exists, update that record.
3. If probable duplicates exist, return `needs_human_review` instead of creating.
4. If no match, create Lead/Contact only on confirmed write.
5. Store idempotency key and audit entry.

### 2. Behaviour-led nurture

1. Webhook arrives with concrete behavioural reason.
2. CRM checks consent, suppression, DND, recent human conversation, channel cooldown, duplicate idempotency key, and owner state.
3. Planner creates one of: note, task, content recommendation, opportunity, draft outbound, or no-op.
4. If outbound is proposed, message copy must include a specific reason and useful CTA.
5. Human/agent confirms write or send where required.

Bad: “Just checking in.”
Good: “You ran the multiplex calculator twice in Hamilton this week. I pulled together the two next things to check before you waste time on showings.”

### 3. Weekly leaderboard email

Trigger: `leaderboard.weekly.ready` from Realist.ca.

1. Upsert contact by Realist user identity.
2. Store `realist_leaderboard_snapshot`.
3. Build email draft from stats, rank, streaks, and one return CTA.
4. Enforce email consent, suppression, and weekly idempotency.
5. Queue/send through current email rail.
6. Log the exact payload, rendered subject, rendered body hash, send rail, and provider message id.

Leaderboard email should be short:

- Your week in Realist
- 2-4 stats
- One insight from behaviour
- One CTA back to Realist.ca

### 4. Help finding / financing deals

Triggers: repeated calculator use, saved deals, financing intent, explicit help request.

Actions:

- Create/update Opportunity with type `find_deal`, `finance_deal`, or `analyze_deal`.
- Assign to CRM owner or routing queue.
- Create task with behaviour context and next best question.
- Optionally draft SMS/email for approval.

## Agent action endpoints

The scaffold in `realist-agent-api/` defines the shape. Implementation should eventually expose these routes behind SuiteCRM auth or a separate internal service with SuiteCRM API credentials.

| Endpoint | Method | Purpose |
|---|---:|---|
| `/realist-agent/actions/validate` | POST | Validate an action envelope and return policy result. |
| `/realist-agent/actions/dry-run` | POST | Resolve target records and show proposed creates/updates/messages. |
| `/realist-agent/actions/confirm` | POST | Execute a previously dry-run action with same idempotency key. |
| `/realist-agent/webhooks/realist` | POST | Receive Realist.ca behavioural webhooks. |
| `/realist-agent/audit/{idempotency_key}` | GET | Return the audit trail for one key. |

## Action policy checklist

Every mutation checks:

- Actor identity and allowed action type.
- `dry_run=true` unless request is a confirmation of a stored dry run.
- Idempotency key has not already produced a write.
- Search-before-create result is clean.
- Consent and suppression rules.
- Behavioural reason is present and maps to a recent event.
- Outbound cooldown and active-human-conversation guard.
- Payload hash matches the dry-run preview at confirmation time.

## Migration plan from GHL

### Phase 0: Inventory

- Export GHL contacts, tags, custom fields, conversations, opportunities, automations, calendars, forms, and source mappings.
- Classify fields into: keep, merge, delete, replace with Realist event.
- Identify high-risk automations that currently send external messages.

### Phase 1: Mirror mode

- Realist.ca webhooks write to SuiteCRM in dry-run/log-only mode.
- Compare GHL contact counts, tag/state mapping, opportunities, and latest notes.
- No outbound sends from SuiteCRM yet.

### Phase 2: Source-of-truth cutover

- SuiteCRM becomes primary contact/opportunity/task record.
- GHL remains a rail if needed for SMS/calling/calendar.
- All outbound decisions originate from SuiteCRM/OpenClaw policy, not GHL workflows.

### Phase 3: Replace rails

- Swap GHL email/SMS/calendar with direct providers.
- Keep provider message ids in SuiteCRM audit records.
- Run reconciliation reports until missing-send/missing-log rate is near zero.

### Phase 4: Retire GHL automations

- Disable generic drips first.
- Retire only after SuiteCRM event logs, action logs, and dashboards prove parity.

## First implementation slice

1. Keep this architecture doc under version control.
2. Add action envelope schema and examples in `realist-agent-api/`.
3. Build webhook intake in log-only mode.
4. Add custom fields for Realist identity and behavioural summary.
5. Implement dry-run search-before-create for contacts/leads.
6. Implement leaderboard webhook -> email draft, not send.
7. Add audit table/module before enabling confirmed writes.

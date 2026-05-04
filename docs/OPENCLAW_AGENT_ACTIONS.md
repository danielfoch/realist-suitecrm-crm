# OpenClaw Agent Actions for Realist CRM

OpenClaw agents may help operate the CRM, but they do not get unrestricted write access. The CRM must enforce a small action contract: search first, dry-run first, confirmed writes only, idempotency keys, and audit logs.

## Rules

1. **Search before create.** `create_contact`, `create_lead`, and `create_opportunity` must include prior search evidence.
2. **Dry-run first.** Mutations default to `dry_run=true`. Confirmed writes require the same `idempotency_key` and an existing approved dry-run record.
3. **Idempotency key required.** No key, no write.
4. **Audit everything.** Store actor, source event, target records, policy checks, payload hash, diff preview, confirmation, and execution result.
5. **Behavioural reason required for outreach.** Outbound copy must tie to a recent Realist.ca event or explicit user request.
6. **No generic nurture.** If the reason is “follow up because time passed,” block it unless tied to an active open task created by a behavioural event.
7. **Provider rails are not authority.** SMS/email/calendar providers log delivery; SuiteCRM owns the state.

## Action envelope

```json
{
  "schema_version": "realist.agent_action.v1",
  "action_type": "contact.upsert",
  "dry_run": true,
  "idempotency_key": "realist:user.created:usr_123",
  "actor": {
    "type": "openclaw_agent",
    "id": "clyde",
    "display_name": "Clyde"
  },
  "source": {
    "system": "realist.ca",
    "event_id": "evt_01J...",
    "event_type": "user.created",
    "occurred_at": "2026-05-03T13:00:00Z"
  },
  "behaviour_reason": "User created a Realist.ca account after viewing Hamilton multiplex content.",
  "target": {
    "module": "Contacts",
    "lookup": {
      "realist_user_id": "usr_123",
      "email": "investor@example.com"
    }
  },
  "payload": {
    "fields": {
      "first_name": "Alex",
      "last_name": "Investor",
      "email1": "investor@example.com",
      "lead_source": "Realist.ca"
    }
  },
  "policy": {
    "requires_confirmation": true,
    "allow_outbound": false
  }
}
```

## Supported first actions

| Action type | Description | Dry-run result |
|---|---|---|
| `contact.search` | Search contacts/leads by Realist user id, email, phone, and name. | Matching records and duplicate risk. |
| `contact.upsert` | Create/update contact or lead from Realist identity. | Field diff or proposed create. |
| `note.create` | Add behaviour summary note to lead/contact/opportunity. | Target resolution and rendered note. |
| `task.create` | Create human task from behavioural signal. | Assignee, due date, subject/body. |
| `opportunity.upsert` | Create/update help-finding/help-financing opportunity. | Matching opportunity or proposed create. |
| `outbound.draft` | Draft SMS/email only. | Rendered copy, behavioural reason, policy checks. |
| `leaderboard_email.draft` | Draft weekly leaderboard email from webhook stats. | Subject/body preview, stats, CTA, consent result. |

## Policy checks

Every `dry-run` and `confirm` returns a policy block:

```json
{
  "allowed": false,
  "status": "blocked",
  "checks": [
    { "name": "idempotency_key_present", "passed": true },
    { "name": "search_before_create", "passed": true },
    { "name": "behaviour_reason_present", "passed": true },
    { "name": "email_consent", "passed": false, "message": "No email consent on contact." },
    { "name": "cooldown", "passed": true }
  ]
}
```

## Examples

### Contact upsert from new Realist.ca user

See `realist-agent-api/examples/contact-upsert.dry-run.json`.

Expected behaviour:

1. Search SuiteCRM by `realist_user_id`, email, phone.
2. If unique match exists, preview field updates.
3. If no match exists, preview contact creation.
4. If duplicate risk exists, return `needs_human_review` and do not create.

### Weekly leaderboard email draft

See `realist-agent-api/examples/leaderboard-email.dry-run.json`.

Expected behaviour:

1. Validate `leaderboard.weekly.ready` source event.
2. Upsert/store leaderboard snapshot in dry-run output.
3. Check consent/suppression/cooldown.
4. Draft email only; sending is a separate confirmed action or human approval.

### Deal help task

A repeated calculator-use event can create a task:

```json
{
  "schema_version": "realist.agent_action.v1",
  "action_type": "task.create",
  "dry_run": true,
  "idempotency_key": "realist:calculator.used:usr_123:2026-05-03:underwrite-help-task",
  "actor": { "type": "openclaw_agent", "id": "clyde" },
  "source": {
    "system": "realist.ca",
    "event_id": "evt_calc_456",
    "event_type": "calculator.used",
    "occurred_at": "2026-05-03T15:20:00Z"
  },
  "behaviour_reason": "User ran the multiplex deal calculator three times in 24 hours for Hamilton properties and requested financing assumptions.",
  "target": {
    "module": "Contacts",
    "lookup": { "realist_user_id": "usr_123", "email": "investor@example.com" }
  },
  "payload": {
    "task": {
      "subject": "Review Hamilton multiplex calculator activity",
      "body": "Ask what property they are underwriting and whether they want financing help.",
      "priority": "High",
      "due_in_hours": 24
    }
  },
  "policy": { "requires_confirmation": true, "allow_outbound": false }
}
```

## Audit log minimum fields

- `action_id`
- `schema_version`
- `action_type`
- `dry_run`
- `confirmed_from_action_id`
- `idempotency_key`
- `actor_type`, `actor_id`
- `source_system`, `source_event_id`, `source_event_type`
- `behaviour_reason`
- `target_module`, `target_record_id`
- `payload_hash`
- `policy_status`
- `policy_checks_json`
- `diff_preview_json`
- `execution_status`
- `provider_message_id` where relevant
- `created_at`, `confirmed_at`, `executed_at`

## Confirmation flow

```text
OpenClaw -> POST /realist-agent/actions/dry-run
CRM      -> stores dry-run audit + returns preview
OpenClaw -> asks for confirmation when required
OpenClaw -> POST /realist-agent/actions/confirm with same idempotency_key
CRM      -> verifies payload hash + policy + confirmation
CRM      -> writes to SuiteCRM + records execution audit
```

If the payload changes after dry-run, confirmation must fail with `payload_changed_after_dry_run`.

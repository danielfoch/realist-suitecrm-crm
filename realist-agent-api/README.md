# Realist Agent API Scaffold

This is a safe, non-runtime scaffold for the Realist.ca/OpenClaw layer. It intentionally does not register SuiteCRM entry points yet. The first implementation should be log-only/dry-run until the audit table/module and policy checks are in place.

## Intended responsibilities

- Receive Realist.ca behavioural webhooks.
- Normalize events into SuiteCRM contact/lead/opportunity/task actions.
- Enforce OpenClaw-safe action rules.
- Store idempotent audit records.
- Produce dry-run previews before any confirmed write.
- Draft leaderboard/content/deal-help outreach only when a behavioural reason exists.

## Proposed routes

```text
POST /realist-agent/webhooks/realist
POST /realist-agent/actions/validate
POST /realist-agent/actions/dry-run
POST /realist-agent/actions/confirm
GET  /realist-agent/audit/{idempotency_key}
```

## Files

- `schemas/action-envelope.schema.json` - JSON schema for first action envelope.
- `examples/contact-upsert.dry-run.json` - new Realist.ca user -> contact upsert preview.
- `examples/leaderboard-email.dry-run.json` - weekly leaderboard payload -> email draft preview.

## Implementation notes

- Start with a separate internal controller/service or SuiteCRM custom entry point only after auth and audit storage are decided.
- Do not wire this into public SuiteCRM routes without HMAC webhook verification and CRM-user permission checks.
- Confirmed writes should reference a stored dry-run by `idempotency_key` and `payload_hash`.
- Use existing SuiteCRM modules first; custom modules can come after the first dry-run workflow proves useful.

# Realist Leaderboard Rewards

Realist should be able to reward the top analyst on the leaderboard every month, e.g. `$100`, through Stripe.

This is a retention and behaviour loop, not a gambling/promo gimmick. The prize should reward useful underwriting activity and quality participation, not spammy clicks.

## Product job

Make users want to analyze another deal because the leaderboard has status, progress, and a real monthly reward.

User story:

> As a Realist user, I want my deal-analysis activity to count toward a monthly leaderboard so I can compete, improve, and potentially earn a reward for being the top analyst.

## Monthly reward flow

```text
User analyzes/saves/compares deals
  -> events feed leaderboard score
  -> monthly leaderboard closes
  -> eligibility/fraud checks run
  -> winner is selected
  -> payout is queued via Stripe
  -> admin review/approval gate for MVP
  -> Stripe transfer/payout sent
  -> CRM logs reward + user notified
```

## MVP rule

Do **not** auto-send money with no review on day one.

MVP should:

1. Calculate monthly winner.
2. Run eligibility and anti-abuse checks.
3. Create a `leaderboard_reward` record with status `pending_review`.
4. Notify/admin-task Dan or ops for approval.
5. Only then send through Stripe.

After the system proves clean, it can move to auto-payout with exception review.

## Reward model

| Field | Description |
|---|---|
| `reward_id` | Unique reward record id. |
| `period` | Monthly period, e.g. `2026-05`. |
| `leaderboard_id` | Source leaderboard. |
| `winner_user_id` | Winning Realist user. |
| `amount_cents` | Default `10000`. |
| `currency` | `cad` by default unless changed. |
| `status` | `pending_review`, `approved`, `paid`, `blocked`, `failed`. |
| `stripe_account_id` | Connected account/customer payout target where applicable. |
| `stripe_transfer_id` | Stripe transfer/payment id after execution. |
| `eligibility_result` | Pass/fail details. |
| `fraud_result` | Anti-abuse details. |
| `approved_by` | Admin/operator id. |
| `paid_at` | Timestamp. |

## Eligibility rules

A winner must have:

- Verified account email.
- Accepted reward terms.
- Valid payout destination or Stripe onboarding completed.
- No suppression/fraud flag.
- Minimum quality threshold, not just activity volume.
- At least one meaningful saved/compared/analyzed deal in the month.
- No self-dealing/fake duplicate account pattern.

## Scoring/anti-gaming rules

Leaderboard points should reward quality, not raw button mashing.

Good signals:

- completed calculator run with reasonable assumptions
- saved deal
- compared deal
- applied assumptions to similar deal
- corrected/refined assumptions
- created saved search
- returned to evaluate updated deal
- submitted useful feedback on plex detection or rent assumptions

Weak/spam signals:

- repeated identical calculator runs
- high-volume page refreshes
- duplicate accounts/IP/device patterns
- obviously impossible assumptions
- no saved/compared/meaningful outputs

Add daily caps and diminishing returns.

## Stripe implementation options

Preferred long-term: **Stripe Connect**.

- Users/analysts who can receive money complete Stripe Connect onboarding.
- Realist stores `stripe_account_id`.
- Monthly winner gets a transfer after approval.

MVP alternatives:

- Stripe coupon/account credit if cash payout is too much operational overhead.
- Manual e-transfer task generated from CRM while Stripe Connect is being wired.

Do not store bank details directly in Realist/SuiteCRM.

## Events

| Event | Purpose |
|---|---|
| `leaderboard.monthly.closed` | Monthly leaderboard finalized. |
| `leaderboard.reward.calculated` | Candidate winner/reward computed. |
| `leaderboard.reward.pending_review` | Admin approval task created. |
| `leaderboard.reward.approved` | Reward approved for payout. |
| `leaderboard.reward.paid` | Stripe payout/transfer succeeded. |
| `leaderboard.reward.blocked` | Reward blocked by eligibility/fraud rule. |
| `stripe.connect_onboarding.started` | User started payout onboarding. |
| `stripe.connect_onboarding.completed` | User can receive payouts. |

## CRM integration

SuiteCRM should store the reward state and audit trail.

Actions:

- Create `leaderboard_reward` record.
- Add note to winner contact timeline.
- Create admin task for reward review.
- Log Stripe ids and payout state.
- Notify user only after approval/payment state is known.

## User-facing copy

Use careful language:

- "Top eligible analyst each month gets $100."
- "Subject to eligibility and anti-abuse review."
- "Quality analysis matters; spammy activity does not count."

Avoid:

- guaranteed prize language before eligibility checks
- lottery/gambling framing
- implying every analysis has cash value

## Build order

1. Add leaderboard period model and monthly close job.
2. Add scoring quality/caps/diminishing returns.
3. Add `leaderboard_reward` schema/table or equivalent backend object.
4. Add admin review screen/task.
5. Add Stripe Connect onboarding field/flow.
6. Add payout execution behind admin approval.
7. Add user notification and public/monthly winner display if desired.

# Realist.ca Build Plan

Realist.ca should become the signal layer for the CRM, not just a listings/search front end.

## North star

Realist helps Canadian investors find, analyze, and act on mispriced real estate opportunities. The site should make the underwriting engine visible: cap rates, cash flow, rent estimates, investor score, risk, and next best actions.

## Homepage slice: AI deal radar

Build the homepage hero as an interactive underwriting radar:

- Dark premium interface with a map/radar panel.
- Hoverable market zones or properties.
- Metrics that rapidly calculate while the cursor moves, then settle into a conclusion.
- Metrics: estimated cap rate, monthly cash flow, rent estimate, price/rent ratio, investor score, risk level.
- Conclusion examples:
  - "Hamilton duplexes are screening better than Toronto condos this week."
  - "Oshawa entry-level rentals show stronger cash-flow signal under current assumptions."
- CTAs:
  - Start analyzing deals
  - See this week's leaderboard
  - Save a target market

Use mock data first. Do not wait for real GIS, IDX, or rent APIs to make the product feel alive.

## CRM/event integration requirement

Every meaningful product interaction should eventually emit a normalized event to the Realist CRM backend.

Initial event catalog:

| Event | Trigger | CRM use |
|---|---|---|
| `homepage.market_hovered` | User hovers/scans a market. | Session personalization only; no outreach. |
| `homepage.cta_clicked` | User clicks hero CTA. | Attribution/source note after signup. |
| `deal_metric.viewed` | User opens/inspects deal metrics. | Interest signal for market/content recommendation. |
| `calculator.started` | User starts underwriting. | Behaviour note; return-visit personalization. |
| `calculator.completed` | User completes underwriting. | Strong intent; possible help/financing CTA. |
| `saved_search.created` | User saves criteria. | Explicit target-market intent; possible deal alert/task. |
| `leaderboard.viewed` | User checks ranking/progress. | Retention loop and weekly leaderboard email. |
| `financing.intent` | User checks financing/help CTA. | Hot handoff task/opportunity. |

## Backend alignment

The SuiteCRM fork owns durable state. Realist.ca emits signals. OpenClaw chooses safe next actions. Email/SMS/calendar are replaceable rails.

```text
Realist.ca interactions
  -> normalized events
  -> Realist Agent API
  -> SuiteCRM event/action/audit records
  -> next-best-action engine
  -> draft/send/task/handoff through replaceable rails
```

## Build order

1. Ship the homepage radar with mock data.
2. Add event emission stubs behind a single client function, e.g. `trackRealistEvent(eventType, payload)`.
3. Add backend webhook intake in log-only mode.
4. Add contact timeline and next-best-action endpoint.
5. Wire leaderboard email draft from CRM backend.
6. Replace mock metrics with live Realist deal/search/rent data.
7. Add account/session personalization from prior user behaviour.

## Cross-platform app direction

Realist.ca should become a single account-backed platform across web, iOS, and Android. The mobile apps should use the existing site/product vision as the guide, not become a separate product.

See `docs/REALIST_CROSS_PLATFORM_APP_PLAN.md` for the full app strategy.

Key rule: all saved searches, saved deals, calculator runs, leaderboard state, alerts, preferences, and help/financing requests must live in shared backend state so users can move between web, iOS, and Android seamlessly.

Build web/mobile around the same event model:

```text
web / iOS / Android interaction
  -> shared Realist API + auth
  -> normalized Realist event
  -> Realist Agent API
  -> SuiteCRM source-of-truth records
  -> next-best-action / draft / task / handoff
```

## Non-negotiables

- No generic nurture.
- No fake guaranteed returns.
- Use "estimated", "screening better", and "based on assumptions" language.
- Every outbound CRM action needs a behavioural reason.
- Build for API/agent operation first; UI is the visible layer, not the system of record.

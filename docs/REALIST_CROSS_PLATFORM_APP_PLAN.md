# Realist.ca Cross-Platform App Plan

Realist.ca should become one product across web, iOS, and Android: the same account, saved searches, saved deals, underwriting history, leaderboard, alerts, and CRM-driven next best actions everywhere.

## Product thesis

The app should not be a separate mobile product. It should be a native-feeling shell around the same Realist platform and backend state.

Users should be able to:

- Start underwriting a deal on web.
- Save it to their account.
- Open the iOS/Android app later and see the same saved deal, assumptions, notes, leaderboard progress, and alerts.
- Get push/email/SMS prompts only when there is a behavioural reason.
- Ask for help finding, financing, or analyzing a deal, and have that request land in the SuiteCRM-backed CRM as a task/opportunity.

## Platform strategy

Use one shared product/backend with multiple clients.

```text
Realist Web App
Realist iOS App
Realist Android App
        |
        v
Shared Realist API + Auth + Event Tracking
        |
        v
Realist Agent API / OpenClaw action layer
        |
        v
SuiteCRM fork as source of truth
        |
        v
Replaceable rails: email | SMS | push | calendar | analytics
```

## Recommended app approach

### Phase 1: Web-first PWA foundation

Before native app work, make the web platform mobile-grade:

- Responsive/mobile-first deal search and analyzer.
- Login/account flows that work cleanly on mobile Safari/Chrome.
- Saved searches, saved deals, leaderboard, calculator history, and profile saved server-side.
- Installable PWA shell where practical.
- Push-notification abstraction prepared, even if native push comes later.

This keeps the first version shippable from Replit/web without waiting on App Store/TestFlight complexity.

### Phase 2: Shared-code mobile app

Use a shared-code mobile framework unless there is a hard reason not to.

Preferred candidates:

| Option | Use if | Tradeoff |
|---|---|---|
| React Native + Expo | Existing Realist frontend is React/TypeScript or close to it. | Fastest path to iOS + Android from one codebase. |
| Capacitor/Ionic wrapper | Existing web app is strong and mostly needs native shell, push, deep links. | Less native feel; fastest reuse of web UI. |
| Native Swift/Kotlin | Need heavy native map/performance features immediately. | Slowest and highest maintenance; avoid first. |

Recommended default: **React Native + Expo** if the current Realist frontend is React/TypeScript. If the current web app is already strong and responsive, start with **Capacitor** as the quickest wrapper, then graduate to React Native screens where needed.

Xcode on the Mac mini is useful for iOS simulator, TestFlight builds, signing, and native debugging. Android builds can run from the same machine with Android Studio/SDK installed later.

## Shared account/state requirements

Account state must live on the backend, not device-local storage.

### Required shared objects

| Object | Must sync across web/iOS/Android |
|---|---|
| User profile | name, email, phone, role, investor/realtor type, consent settings |
| Investor profile | budget, target markets, property types, financing status, time horizon |
| Saved searches | filters, markets, alerts, notification preferences |
| Saved deals | property id, assumptions, notes, status, score, timestamps |
| Calculator runs | assumptions, outputs, scenario versions, source listing |
| Leaderboard | weekly stats, rank, streaks, return CTA |
| Alerts | deal alerts, market alerts, leaderboard alerts, financing/help prompts |
| CRM handoffs | help requests, financing requests, agent assignment, task status |
| Content history | podcast/article/video engagement where useful for recommendations |
| Consent/suppression | email/SMS/push permissions, DND, unsubscribe state |

### Sync rules

- Backend is source of truth.
- Clients cache for speed/offline reading only.
- Every write uses an authenticated API request.
- Writes include idempotency keys where duplicates matter.
- Conflict resolution defaults to latest-write-wins for low-risk preferences and explicit versioning for calculator/deal assumptions.
- Saved deal assumptions should keep version history so a user can revisit old underwriting outputs.
- Anonymous sessions may collect local events, but cross-platform saved state starts only after account creation/login.

## Auth requirements

Minimum:

- Email/password or magic-link login.
- OAuth/social login only if it simplifies conversion, not as a dependency.
- Refresh-token/session strategy that works for web and native clients.
- Device registration table for native push tokens.
- Account deletion/export path for compliance.

Nice later:

- Passkeys.
- Apple login for iOS App Store friendliness.
- Google login for Android/web convenience.

## Event tracking and CRM integration

Mobile and web clients should emit the same event names. The backend should not care which client sent the signal.

Event envelope:

```json
{
  "event_id": "evt_01J...",
  "event_type": "saved_search.created",
  "occurred_at": "2026-05-04T12:00:00Z",
  "realist_user_id": "usr_123",
  "platform": "ios",
  "session_id": "sess_456",
  "idempotency_key": "realist:saved_search.created:usr_123:search_789",
  "payload": {}
}
```

Initial cross-platform events:

| Event | Web | iOS | Android | CRM use |
|---|---:|---:|---:|---|
| `homepage.market_hovered` | yes | optional | optional | Personalization only. |
| `deal_metric.viewed` | yes | yes | yes | Market/deal interest. |
| `calculator.started` | yes | yes | yes | Underwriting intent. |
| `calculator.completed` | yes | yes | yes | Strong investor signal. |
| `saved_search.created` | yes | yes | yes | Deal alert / follow-up. |
| `saved_deal.created` | yes | yes | yes | Property-level intent. |
| `leaderboard.viewed` | yes | yes | yes | Retention loop. |
| `push_notification.opened` | no | yes | yes | Attribution and timing quality. |
| `financing.intent` | yes | yes | yes | Hot handoff task/opportunity. |
| `help.requested` | yes | yes | yes | Immediate CRM task/opportunity. |

## Native app MVP screens

Do not clone the entire website first. Build the habit loop.

1. **Home / Deal Radar**
   - AI deal-radar visual.
   - This week's best markets/deal signals.
   - Resume last analysis.

2. **Search / Map**
   - Mobile-friendly property/market search.
   - Saved filters.
   - Deal-score cards.

3. **Deal Analyzer**
   - Cap rate, cash flow, rent estimate, financing assumptions.
   - Save scenario.
   - Compare scenarios.

4. **Saved**
   - Saved searches.
   - Saved deals.
   - Saved calculator runs.

5. **Leaderboard**
   - Weekly score/rank/streak.
   - Return CTA.
   - Progress loop.

6. **Help / Handoff**
   - Ask for help finding a deal.
   - Ask for financing help.
   - Ask for deal review.
   - Creates CRM task/opportunity.

7. **Profile / Preferences**
   - Investor profile.
   - Notification preferences.
   - Consent/unsubscribe controls.

## Push notification rules

Push is powerful, so treat it like SMS: behaviour-first only.

Allowed:

- Saved search match.
- Leaderboard ready.
- Significant watched-deal change.
- Saved deal assumption/rent/price change.
- Human replied to help/financing request.

Blocked:

- Generic nurture.
- Generic market spam.
- Repeated reminders without new signal.

## Build order

### Slice 0: Inspect current Realist web stack

- Identify frontend framework, routing, auth, API shape, database, and deployment flow.
- Confirm whether React Native/Expo, Capacitor, or another path fits best.
- Inventory existing account/saved-search/saved-deal/calculator data.

### Slice 1: Shared backend contract

- Define shared API routes for profile, saved searches, saved deals, calculator runs, leaderboard, and event tracking.
- Add OpenAPI/JSON schema docs.
- Add auth/session requirements.
- Add platform field to events: `web`, `ios`, `android`.

### Slice 2: Web mobile-grade foundation

- Make existing web flows responsive and account-backed.
- Add `trackRealistEvent()` client wrapper.
- Ensure saved state syncs server-side.

### Slice 3: App shell proof-of-concept

- Create native app workspace using chosen approach.
- Implement login, home/deal radar, saved deals, and profile.
- Connect to staging API.
- Run on iOS simulator through Xcode.

### Slice 4: Cross-platform saved-state parity

- User saves search/deal on web; verify it appears in iOS/Android.
- User edits assumptions on iOS; verify web shows same scenario/version.
- User completes calculator on Android; verify CRM receives same event envelope.

### Slice 5: Notifications and deep links

- Register devices.
- Add deep links to saved deal, leaderboard, help request, and calculator scenario.
- Add push notifications behind same behavioural policy as email/SMS.

### Slice 6: TestFlight/internal Android build

- Configure signing.
- Ship internal iOS TestFlight build.
- Ship Android internal testing build.
- Add release checklist.

## Repository structure options

If this SuiteCRM fork remains backend-only, keep app code in the Realist web/platform repo and keep only plans/contracts here.

If this becomes the umbrella repo, use:

```text
/apps/web
/apps/mobile
/apps/crm-admin-or-suitecrm
/packages/api-client
/packages/event-schemas
/packages/ui
/packages/config
```

Recommended near-term: do **not** move SuiteCRM into a monorepo yet. Keep this repo as CRM/backend plan and contracts. Build the mobile app from the actual Realist web repo once inspected.

## Codex/Replit prompt for later

Use this after inspecting the actual Realist.ca repo:

```text
You are working in the Realist.ca codebase.

Goal: create a cross-platform build foundation so Realist works as web, iOS, and Android using one shared backend account/state model.

First, inspect the app stack: frontend framework, backend/API, auth, database, routing, build commands, and existing saved-search/saved-deal/calculator/account models. Do not assume the stack.

Then produce and implement the smallest safe slice:
1. Add or document shared API contracts for:
   - user profile
   - investor profile
   - saved searches
   - saved deals
   - calculator runs/scenarios
   - leaderboard
   - event tracking
2. Add a single client event wrapper: trackRealistEvent(eventType, payload), including platform, session id, user id if available, event id, occurred_at, and idempotency key where relevant.
3. Ensure saved searches/deals/calculator runs are account-backed, not only local/browser state.
4. Add mobile-first responsive improvements to the core web flows that will become app screens.
5. If the repo is React/TypeScript, recommend whether Expo React Native or Capacitor is the fastest path and create a docs/mobile-app-plan.md with exact next steps.

Acceptance criteria:
- Existing web app still builds/runs.
- Account-backed saved state is clearly documented or implemented.
- Event tracking wrapper exists or is stubbed in one place.
- No app-store/native signing work yet.
- No heavy rewrite.
```

## Definition of done for the app strategy

The app is not done when it opens on a phone. It is done when a user can move between web, iOS, and Android without losing context, and every meaningful behaviour can feed the CRM safely.

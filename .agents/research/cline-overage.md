# Cline: overage-like mechanisms (credit balance after plan limit, spend limits)

Source: github.com/cline/cline at commit `68b24a92b71ee98c3e8ea4aca7a7a1c0f97a43b8` (same checkout as `cline.md`).
Paths are relative to the repo root. Builds on `.agents/research/cline.md`; the general quota/balance research is not
repeated here.

## Summary

- Cline has NO automatic overage. ClinePass (subscription) and Cline usage-billing (prepaid credits) are two separate
  providers that the user selects explicitly. When a ClinePass limit is hit, the client only offers a manual one-click
  "switch provider to usage-based billing" action. Nothing falls back on its own, and no state field says "plan
  exhausted, now drawing on credits".
  - Docs: "ClinePass is a separate provider from Cline (usage-billing). You can use both independently"
    (`docs/getting-started/clinepass.mdx`, Note under "Why ClinePass").
  - Code: the VS Code `ClinePassLimitError` card asks "Would you like to switch to Usage-Based billing and retry with
    the Cline provider?" and only rewrites `planModeApiProvider`/`actModeApiProvider` to `"cline"` on click, then tells the
    user to retry (`apps/vscode/webview-ui/src/components/chat/ClinePassLimitError.tsx:12-17,28-43,57-69`). The CLI text
    is the same advice: "Switch to Cline usage-based billing and retry with the Cline provider ... /model ... --provider
    cline" (`apps/cli/src/utils/cline-pass-errors.ts:24-33`).
  - The free-model daily limit follows the same pattern: a "switch to paid model" button that commits a
    `cline`-provider model (`ClineFreeModelLimitError.tsx:73-123`).
- The client knows which billing path is in use only from the selected provider id (`cline` = credits, `cline-pass` =
  subscription): the TUI hides cost for `cline-pass` (`apps/cli/src/tui/components/status-bar.tsx:47-49,105`), and
  `ErrorRow` shows the credit card only for `providerId === "cline"` (comment: "Deliberately narrower than the shared
  isClineManagedProvider (which also matches cline-pass): only usage-billing errors get the credit and login prompts",
  `ErrorRow.tsx:~44-47`). There is no server-sent "billing source" field.
- `SPEND_LIMIT_EXCEEDED` is a base limit (a hard cap on credit spend, enforced by the org), not an overage cap on a plan.
  It sits in front of the credit balance, is windowed (`daily`/`monthly`), and is only observable after being hit.
  `insufficient_credits` is the separate "balance is empty" signal.
- No proactive state is exposed for either: only error payloads after a request fails. No "allowed/active" signal exists.

## Fields

### `SPEND_LIMIT_EXCEEDED` (HTTP 429, Cline gateway)

Backend body shape inferred from the e2e mock server, the reshaping code, and stories (no backend source in this repo).

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `code` | string, constant `"SPEND_LIMIT_EXCEEDED"` | no (discriminator; also read from `details.code`) | Error discriminator; checked before the generic 429 rate-limit check | `apps/vscode/src/services/error/ClineError.ts:188-192`; `apps/vscode/src/sdk/message-translator.ts:2857` |
| `limit_scope` | string; only `"user"` seen | yes (passed through, never consumed by UI) | Whose limit tripped (per-user value seen; other values such as org/team UNVERIFIED) | `message-translator.ts:2864`; `ErrorRow.stories.tsx:167,189`; `src/test/e2e/fixtures/server/index.ts:467` |
| `budget_period` | string, `"daily"` or `"monthly"` seen | yes | Window the cap applies to; also used to build the label | `message-translator.ts:2865`; `SpendLimitError.tsx:27-28,40-41,86` |
| `limit_usd` | number, whole USD (e.g. `20.0`, `100.0`) | yes | Configured cap in dollars, not micro-dollars (UI does `limitUsd.toFixed(2)` with `$`) | `message-translator.ts:2866`; `SpendLimitError.tsx:29-30,41,113` |
| `spent_usd` | number, USD (e.g. `20.5`, `103.22`) | yes | Spend in the current window; can exceed the limit | `message-translator.ts:2867`; `ErrorRow.stories.tsx:170,192` |
| `resets_at` | string ISO 8601 or `null` | yes | When the window resets; `null` in the monthly story | `message-translator.ts:2868`; `SpendLimitError.tsx:10-24,31-32`; `ErrorRow.stories.tsx:193` |
| `message` | string | yes | Human text, e.g. "Your daily spend limit of $20.00 has been reached." Used as the display fallback | `message-translator.ts:2869`; `SpendLimitError.tsx:40-41` |

- "Minimal" variant: only `code` + `message` (`ErrorRow.stories.tsx:~203-218`). Plain-text-only variants (JSON
  stripped by the SDK) are detected by substring `spend_limit_exceeded` or `spend limit` and synthesized with just
  `code` and `message` (`message-translator.ts:2816-2826`; test `message-translator.test.ts:1646-1666`).
- Envelope on the wire in the mock: `{error: {code, limit_scope, budget_period, limit_usd, spent_usd, resets_at,
  message}}` with HTTP 429 (`src/test/e2e/fixtures/server/index.ts:461-476`). After VS Code reshaping:
  `{message, code, providerId, details: {code, limit_scope, budget_period, limit_usd, spent_usd, resets_at, message}}`
  (`message-translator.ts:2858-2871`). Status 429 appears as `status` in the stories.

### `insufficient_credits` (HTTP 402, Cline gateway)

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `code` | string `"insufficient_credits"` | no | Discriminator | `message-translator.ts:2841`; `ClineError.ts:184` |
| `current_balance` | number | required to classify as `Balance` (`ClineError.ts:184`); plain-text fallback sets it from a `$x.xx` match or `0` (`message-translator.ts:2803-2813`) | Remaining credit | `message-translator.ts:2841-2851` |
| `total_spent` | number | yes | Lifetime spend (display) | `message-translator.ts:2847`; `CreditLimitError.tsx:7-8,60-61` |
| `total_promotions` | number | yes | Promotional credits granted (display) | `message-translator.ts:2848`; `CreditLimitError.tsx:62-65` |
| `buy_credits_url` | string | yes | Top-up link; default is a dashboard URL for user vs org | `message-translator.ts:2850`; `CreditLimitError.tsx:15-18,29-31` |
| `message` | string | yes | e.g. "Not enough credits available" (mock) / "You have run out of credits." (default) | `message-translator.ts:2849`; `index.ts:480-486` |

Units: UNCLEAR for the error body. `CreditLimitError.tsx:46` carries the comment "balance is stored in microcredits" but
prints `currentBalance.toFixed(2)` without dividing, so either the 402 body is already in dollars or the UI is wrong.
The REST `/balance` endpoint is micro-dollars (`balance / 1_000_000`, `apps/cli/src/utils/output.ts:222-227`,
see `cline.md`). `limit_usd`/`spent_usd` are explicitly USD.

### ClinePass and free-model limits (no structured payload)

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| ClinePass limit | free text, "You have reached your ... ClinePass limit ... please try again later." | n/a | Subscription cap hit; which window (5h/weekly/monthly) is not machine readable | `sdk/packages/llms/src/providers/errors.ts:10-12,134-141`; `ClineError.ts:222-227` |
| Free-model limit | free text containing "free limit reached on model" and "try again in <duration>" | reset duration scraped by substring | Daily free-model cap | `errors.ts:13-14,143-164`; `cline-pass-errors.ts:50-60` |
| Not subscribed | text "the user is not subscribed to required model plan" | n/a | Entitlement missing, an "overage disabled/not allowed" style state | `errors.ts:3-8,~120-130`; `ClineError.ts:214-218` |
| Org + individual plan | text "organization accounts cannot use individual model inference subscriptions" | n/a | ClinePass not usable under an org account | `errors.ts:7-8`; `ClineError.ts:205-211` |
| `INFERENCE_CAP_ERROR` | code, free text | n/a | Maps to `QuotaExceeded` | `ClineError.ts:228-230` |

ClinePass window set is documented only in prose: three limits, 5-hour rolling, weekly (calendar week), monthly
(calendar month); usage is viewable only on the web dashboard (`docs/getting-started/clinepass.mdx`, "Usage").

### Limit-increase request

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `POST /api/v1/users/me/budget/request` | no body | n/a | Asks the org admin to raise the spend limit | `apps/vscode/src/services/account/ClineAccountService.ts:267-281`; `apps/vscode/src/sdk/account-service.ts:246-257` |
| `submitLimitIncreaseRequest` RPC / `SubmitLimitIncreaseResponse {success: bool}` | proto | n/a | Webview to extension wrapper | `apps/vscode/proto/cline/account.proto:58-59,132-135`; `submitLimitIncreaseRequest.ts:13-26` |

## State and coverage semantics

- Spend limit as base vs overage cap: it is a base limit on pay-as-you-go spend. It caps total dollars charged per
  window (user or org budget) regardless of the credit balance. The client text is "Limits set by your organization."
  (`SpendLimitError.tsx:~118-122`) and the code comment calls it an "org-enforced budget cap" (`ClineError.ts:188`).
  It does not bound a ClinePass overage, because ClinePass has no overage path.
- Interaction order on the server (from the mock only, UNVERIFIED for the real backend): the spend limit check runs
  first, then the zero-balance check ("Spend limit check takes priority", `src/test/e2e/fixtures/server/index.ts:459-461,
  478-487`). The mock only returns 402 when the user has no organization and balance <= 0; org users are not blocked by
  balance in the mock.
- Which state is observable:
  - Blocked by spend limit: 429 + `SPEND_LIMIT_EXCEEDED` (after the fact).
  - Blocked by empty credits: 402 + `insufficient_credits`.
  - Blocked by plan limit: text error for `cline-pass` (no code).
  - No "allowed" or "active" signals. "Overage in use" is implied only by the provider id being `cline`.
- Reasons (the closest thing to a reason enum): the `ClineErrorType` classes `Balance`, `SpendLimit`,
  `OrgClinePassRestriction`, `Entitlement`, `ClineFreeModelLimit`, `ClinePassLimit`, `ClineFreePromotionEnded`, `Auth`,
  `QuotaExceeded`, `RateLimit` (`ClineError.ts:11-21`, classification order `:178-255`). These are client-side
  string/code classifiers, not a wire enum.
- Who sets the limit: the organization (admin). UI copy: "Limits set by your organization"; `limit_scope: "user"` suggests
  per-member budgets, possibly org-level scope values exist (UNVERIFIED). The user can neither set nor change it in
  Cline; they can only "Request Increase", which POSTs to `/api/v1/users/me/budget/request` (admin is notified,
  UNVERIFIED how; comment says "the backend records the request", `ClineAccountService.ts:270`).
- Consent: there is no consent prompt to start spending credits. The only consent-like flows are (a) the explicit user
  click to switch provider after a ClinePass or free-model limit and (b) the "Request Increase" button with a 5-minute
  client-side cooldown stored in `localStorage` key `cline:spendLimitRequestCooldown`
  (`SpendLimitError.tsx:5-6,64-83`; states `idle | sending | sent`).
- Personal vs org credits: balance shown is the org's when an org is active (`displayedBalance`, `cline.md`); buy-credits
  URL differs per scope (`CreditLimitError.tsx:15-18,29-31`).

## Mapping to the draft overage entity

| Draft field | Cline native | Notes |
|---|---|---|
| `id` | none. Could be derived from provider id (`cline`) or fixed `credits` / `spend_limit` | No server id |
| `state` allowed/active/disabled | not exposed proactively. Implied: provider `cline` selected = credits active; `cline-pass` = overage not available (separate provider) | Only error-time signals |
| `state.reason` | `ClineErrorType` classes above; closest: `SPEND_LIMIT_EXCEEDED` (limit reached), `insufficient_credits` (out of funds), "not subscribed" (not entitled), "org cannot use individual plans" (disabled by org policy) | Client-derived, no wire enum |
| `meter.used` | `spent_usd` (windowed, spend limit); `total_spent` (lifetime, 402) | |
| `meter.limit` | `limit_usd` | |
| `meter.remaining` | `current_balance` (credits, 402 body and `/balance`); for spend limit derive `limit_usd - spent_usd` | Not sent directly |
| `meter.unit/currency` | USD implicit in field names (`*_usd`); balance REST in micro-dollars; 402 `current_balance` unit unclear | No currency code field |
| `meter.share` | none | |
| `meter.reset` | `resets_at` (ISO, nullable); `budget_period` (`daily`/`monthly`) | Reset can be null for monthly |
| covered base limits | none. ClinePass limits (5h/weekly/monthly) are NOT covered by credits automatically; switching provider is manual. Spend limit instead caps the credit path | Cline would map to "no overage" for ClinePass, and "spend limit + balance" for the `cline` provider |

Takeaway: Cline's spend limit fits better as a base (or "cap") limit on the usage-billing provider than as an overage entity.
An RFD overage entity with `covers: []` or absent, plus `state: disabled` for ClinePass, would be the honest mapping.
Cline's data suggests keeping `meter.limit` optional and allowing the meter to describe a cap (limit + window + reset)
separately from a balance (remaining, no window).

## Gaps and unverified items

- No backend source in this repo: the real wire shape, full `limit_scope`/`budget_period` value sets, and whether
  org-scoped (not just `user`) spend limits exist are UNVERIFIED. Shapes come from the e2e mock (`index.ts:461-487`),
  storybook fixtures, and the reshaping code.
- Unit of `current_balance` in the 402 body is unresolved (see above). `limit_usd`/`spent_usd` are dollars per field name
  and UI formatting.
- Server-side ordering of spend-limit versus credit checks for real accounts is only evidenced by the mock.
- Whether ClinePass users who also hold credits ever silently draw on credits: nothing in the client does it; backend
  behavior UNVERIFIED (docs say the providers are independent).
- Docs/API errors page lists only `402` (credits) and generic `429` (`docs/api/errors.mdx:39,42`); neither mentions
  `SPEND_LIMIT_EXCEEDED`. CHANGELOG 3.78.0 only says 'Add a dedicated "Spend Limit Reached" error UI when spend caps are
  hit' (`CHANGELOG.md:772`).
- None of this reaches ACP clients today (see `cline.md`).

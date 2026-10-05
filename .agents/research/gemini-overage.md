# Gemini CLI: overage (Google One AI credits)

Scope: only the overage flow, where Google One AI credits are spent after a quota is exhausted. This note builds on
`gemini-native.md` (buckets, errors, tier) and `gemini-acp.md` (ACP boundary) and does not repeat them.

Source: https://github.com/google-gemini/gemini-cli at commit `fb972b2f87fe7d5b06d37eac711490162d98de2c` (2026-10-02),
the same checkout as the other Gemini notes. Paths are relative to the repo root. Permalink form:
`https://github.com/google-gemini/gemini-cli/blob/fb972b2f87fe7d5b06d37eac711490162d98de2c/<path>#L<line>`.

Abbreviations: `B` = `packages/core/src/billing/billing.ts`, `CA` = `packages/core/src/code_assist`,
`CFG` = `packages/core/src/config/config.ts`, `CFH` = `packages/cli/src/ui/hooks/creditsFlowHandler.ts`,
`UQF` = `packages/cli/src/ui/hooks/useQuotaAndFallback.ts`, `GQE` = `packages/core/src/utils/googleQuotaErrors.ts`.

## Summary

- **The credit is one scalar balance with no meter around it.** The wire type is `Credits { creditType, creditAmount }`.
  `creditType` is `'CREDIT_TYPE_UNSPECIFIED' | 'GOOGLE_ONE_AI'` and `creditAmount` is an int64 string
  (`CA/types.ts:46-56`). Nothing on the wire gives a limit, a reset time, a period, an expiry, or a unit or currency.
  The UI shows the bare number as "N AI Credits" or "Google AI Credits: N".
- **Three carriers.** The balance comes from `paidTier.availableCredits[]` in `loadCodeAssist`, which the client polls
  at startup and on `/stats`. Each generate response pushes `remainingCredits[]` (the new balance) and
  `consumedCredits[]` (the amount spent by that response) (`CA/converter.ts:77-82`, `CA/server.ts:163-179`, `:296-313`).
- **Opt-in on each request.** A request uses credits only if it carries `enabled_credit_types: ['GOOGLE_ONE_AI']`.
  The wire key is snake_case (`CA/converter.ts:38`, `:141`). The client adds it only when all of these hold:
  - the call is a streaming chat call;
  - the effective `overageStrategy` is `always`;
  - the cached G1 balance is at least `MIN_CREDIT_BALANCE = 50`;
  - the model is in `OVERAGE_ELIGIBLE_MODELS`, which holds `gemini-3-pro-preview`, `gemini-3.1-pro-preview`, and
    `gemini-3-flash-preview`.

  Sources: `CA/server.ts:99-125`, `B:33-37`, `B:137-154`, `packages/core/src/config/models.ts:59-65`.
- **Trigger.** Any `TerminalQuotaError` on an eligible model starts the flow, for a user whose tier has
  `availableCredits`. The one exception is reason `INSUFFICIENT_G1_CREDITS_BALANCE` (`UQF:111-146`). The table below
  shows what happens next.

  | Strategy and balance        | What happens                                      | Source                    |
  | --------------------------- | ------------------------------------------------- | ------------------------- |
  | `ask`, balance >= 50        | Overage menu                                      | `CFH:65-95`               |
  | `ask` or `always`, < 50     | "Empty wallet" menu                               | `CFH:65-95`               |
  | `always`, balance >= 50     | Credits were already sent, so the client falls through to the normal Pro quota dialog | `CFH:77-82` |
  | `never`                     | Normal Pro quota dialog, no credits menus         | `CFH:65-95`               |
- **Consent.** In the menu, "Use AI Credits" switches the strategy at runtime to `always` and retries the request
  (`CFH:153-158`). The choice is not saved. It lasts until the next new user prompt, when `resetBillingTurnState`
  restores the value from settings, which defaults to `ask` (`CFG:2012-2016`, `useGeminiStream.ts:1710-1717`). The
  only persistent consent is the setting `billing.overageStrategy`.
- **Credits are not tied to a quota bucket.** No field links credits to a bucket or quota ID. The only link is the
  client-side set of eligible models.

## Fields

### Wire fields (Code Assist `v1internal`)

| Name | Type | Optional? | Meaning | Source |
| ---- | ---- | --------- | ------- | ------ |
| `Credits.creditType` | `'CREDIT_TYPE_UNSPECIFIED' \| 'GOOGLE_ONE_AI'` | required in the TS type | Credit kind. The client counts only `GOOGLE_ONE_AI` entries (`G1_CREDIT_TYPE`) | `CA/types.ts:46`, `:53-54`; `B:27` |
| `Credits.creditAmount` | string (int64) | required in the type, but read defensively (`?? '0'`, NaN treated as 0) | A count of credits with no unit field. It is shown as a bare integer, "N AI Credits". The client sums all G1 entries | `CA/types.ts:55`; `B:130-134` |
| `GeminiUserTier.availableCredits` (on `LoadCodeAssistResponse.paidTier`) | `Credits[]` | optional | Balance. **Its presence marks the user as eligible for G1 credits.** It is absent for users who are not eligible | `CA/types.ts:87`, `:103-104`; `B:115-128`; `UQF:116-123` |
| `CAGenerateContentRequest.enabled_credit_types` | `string[]` | optional, omitted when `undefined` | Per-request opt-in to spend credits. The value is `['GOOGLE_ONE_AI']` | `CA/converter.ts:33-39`, `:129-143`; `CA/server.ts:113-125` |
| `CaGenerateContentResponse.consumedCredits` | `Credits[]` | optional | Credits spent by this response or stream chunk. The client sums them over the stream, for telemetry only | `CA/converter.ts:80`; `CA/server.ts:163-169`, `:184-194` |
| `CaGenerateContentResponse.remainingCredits` | `Credits[]` | optional | Balance after this response. It replaces the cached G1 entries | `CA/converter.ts:81`; `CA/server.ts:170-179`, `:234-255` |
| `ErrorInfo.reason = INSUFFICIENT_G1_CREDITS_BALANCE` | string enum value | only in errors | The server rejected credit use because the balance is too low. The error is always terminal, whatever its domain | `GQE:361-369`; `GQE:48-50` |

### Client-side constants, settings, and state

| Name | Type | Optional? | Meaning | Source |
| ---- | ---- | --------- | ------- | ------ |
| `billing.overageStrategy` (setting) | `'ask' \| 'always' \| 'never'`, default `ask` | optional | Persistent policy for spending credits once a quota is exhausted. It is shown in the settings dialog | `packages/cli/src/config/settingsSchema.ts:993-1019`; `B:18-24`; `docs/cli/settings.md:99`; `docs/reference/configuration.md:509-512` |
| `Config.billing.overageStrategy` | same | - | Effective value at runtime. It starts from settings, is set to `always` when the user accepts in the menu, and is reset to the settings value on each new user prompt | `CFG:1405-1407`, `:2012-2016`, `:2933-2944` |
| `MIN_CREDIT_BALANCE` | const `50` | - | Credits are used automatically, and the overage menu is shown, only at or above this balance. Below it the "empty wallet" menu is shown. The value is not on the wire | `B:137-188` |
| `OVERAGE_ELIGIBLE_MODELS` | `Set<string>` | - | `gemini-3-pro-preview`, `gemini-3.1-pro-preview`, `gemini-3-flash-preview`. `PREVIEW_GEMINI_FLASH_MODEL` can be overridden at runtime | `B:33-46`; `models.ts:59-65`, `:86`, `:99` |
| `getG1CreditBalance(tier)` | `number \| null` | - | Returns `null` when the user is not eligible (no `availableCredits`, or no G1 entry). Otherwise returns the sum of the G1 entries, which can be `0` | `B:110-135` |
| `creditsNotificationShown` | boolean | - | Makes sure "Using AI Credits for this request." is shown only once per user prompt | `CFG:858`, `:2012-2035`; `CA/server.ts:108-111` |
| `FallbackIntent 'retry_with_credits'` | string literal | - | Retry the same model. The retried request then carries credits because the strategy is now `always` | `packages/core/src/fallback/types.ts:20`; `packages/core/src/fallback/handler.ts:177-178` |
| `OverageMenuIntent` | `'use_credits' \| 'use_fallback' \| 'manage' \| 'stop'` | - | Choices in the overage menu | `packages/cli/src/ui/contexts/UIStateContext.tsx:56-69` |
| `EmptyWalletIntent` | `'get_credits' \| 'use_fallback' \| 'stop'` | - | Choices in the empty wallet menu | `UIStateContext.tsx:72-81` |
| Telemetry `credits_used` | `{model, credits_consumed, credits_remaining}` | - | Sent after a stream when `totalConsumed > 0`. Other billing events: `overage_menu_shown {model, credit_balance, overage_strategy}`, `overage_option_selected {model, selected_option, credit_balance}`, `empty_wallet_menu_shown {model}`, `credit_purchase_click {source, model}` | `packages/core/src/telemetry/billingEvents.ts:25-201` |

## State and coverage semantics

### Trigger

The credits flow runs from inside the TUI fallback handler. Retry calls `onPersistent429` (`retry.ts:397-418`), which
calls `handleFallback`, which calls the handler registered by `UQF:250`. The handler enters the credits flow only when
all of these hold (`UQF:111-146`):

1. The error is a `TerminalQuotaError`. That covers daily quota (`PerDay`/`Daily`), `QUOTA_EXHAUSTED`,
   `RATE_LIMIT_EXCEEDED` with a delay over 300 s or with no delay, `limit: 0`, and `MODEL_CAPACITY_*`. See the
   classification in `gemini-native.md`.
2. `!error.isInsufficientCredits`.
3. `paidTier?.availableCredits` is truthy.
4. `isOverageEligibleModel(failedModel)`.

Two cases skip the flow before any handler runs. Retryable per-minute errors are retried first. A fallback whose
policy action is `silent` switches the model without calling the handler (`handler.ts:83-98`).

`MODEL_CAPACITY_*` errors are also `TerminalQuotaError`, and `UQF` checks `isCapacityExceeded` only after the credits
branch (`UQF:96-106`, `:120-149`). So on an eligible model, a capacity error also opens the overage menu. Whether
credits help with capacity errors on the server is UNVERIFIED.

### States the client can tell apart

| Draft state | How Gemini CLI knows it | Source |
| ----------- | ----------------------- | ------ |
| Not applicable or not eligible | `getG1CreditBalance()` returns `null`, meaning no `availableCredits` or no G1 entry. API-key and Vertex auth have no Code Assist server, so they never have credits | `B:118-128`; `CFH:70-73`; `gemini-native.md` (per auth type) |
| Disabled by the user | `overageStrategy === 'never'`. No credits are sent and no credits menus appear | `B:145-188`; `UQF:148` |
| Allowed, consent needed | `ask` and balance >= 50. The menu appears when a quota error occurs | `B:162-171` |
| Allowed, but balance too low (client-side) | `ask` or `always`, and `0 <= balance < 50`. The client shows the "empty wallet" menu and never sends `enabled_credit_types` | `B:179-188`; `CFH:89-92` |
| Enabled, meaning credits are opted in | `always` (from settings or after consent) and balance >= 50 and the model is eligible. The request carries `enabled_credit_types`, and the TUI shows the info message "Using AI Credits for this request." once per prompt | `CA/server.ts:99-125` |
| Credits actually being spent | Only after the fact: `consumedCredits` greater than 0 in a response. The client sends `enabled_credit_types` on every eligible request while the strategy is `always`, including before any quota error. Whether the server bills credits only after the base quota is exhausted is UNVERIFIED | `CA/server.ts:113-125`, `:163-194` |
| Insufficient (server-side) | The error reason is `INSUFFICIENT_G1_CREDITS_BALANCE`. The flow is skipped and the normal quota dialog appears | `GQE:361-369`; `UQF:114-124` |
| Enabled but did not help | `always`, balance >= 50, and a terminal quota error anyway. The client falls through to the Pro quota dialog, which offers the Flash fallback | `CFH:77-82` |

The client has no explicit disabled reason other than these. It has no field for admin-disabled credits, a region, or
an expired balance.

### Coverage

- Credits are tied to **models, not buckets**. The wire has no bucket, quota ID, or window link. The client gates
  credits by the hard-coded set of three preview models (`B:33-37`, `CA/server.ts:105`). For a Pro model, the menu
  title says "all Pro models" (`UQF:107-109`, `CFH:135`).
- Only the streaming chat path (`geminiChat.ts:1082`, `generateContentStream`) can carry credits. Non-streaming
  `generateContent`, which utility and base LLM calls use, always passes `undefined` (`CA/server.ts:205-216`). It does
  still apply `remainingCredits` if the server sends it (`:234-236`).
- Which daily or per-minute limits the credits stand in for is decided by the server and is UNVERIFIED.

### Consent and dialogs (TUI only)

- **Overage menu** (`packages/cli/src/ui/components/OverageMenuDialog.tsx:44-110`):
  - Header text:
    - "Usage limit reached for {model}."
    - "Access resets at {time}." The time comes from `RetryInfo`.
    - "/stats model for usage details"
    - "/model to switch models."
    - "/auth to switch to API key."
  - Balance line: "You have **N** AI Credits available."
  - Options:
    - "Use AI Credits - Continue this request (Overage)"
    - "Manage - View balance and purchase more credits". This opens `one.google.com/ai/activity` through an
      AccountChooser link with UTM parameters (`B:95-108`, `CFH:163-178`).
    - "Switch to {fallback}", shown only if a fallback exists.
    - "Stop - Abort request"
- **Empty wallet menu** (`packages/cli/src/ui/components/EmptyWalletDialog.tsx:40-108`):
  - Same header as the overage menu.
  - Body text: "To continue using this model now, purchase more AI Credits." and "Newly purchased AI credits may take
    a few minutes to update."
  - Options:
    - "Get AI Credits - Open browser to purchase credits". This opens `one.google.com/ai/credits`, then says "Run
      /stats to check your balance" (`CFH:222-258`).
    - "Switch to {fallback}"
    - "Stop - Abort request"
- **Persistence.**
  - `use_credits` sets `Config.billing.overageStrategy = 'always'` in memory only (`CFH:154-158`, `CFG:2937-2944`).
  - On the next new user prompt (not a continuation), `resetBillingTurnState(settings.billing.overageStrategy)` puts
    the settings value back (`useGeminiStream.ts:1710-1717`). So consent covers the current prompt and its tool-loop
    continuations, and the user is asked again on the next prompt that hits the quota.
  - To persist consent, the user sets `billing.overageStrategy: "always"` in settings.
- **Polling and refresh.**
  - The balance is fetched with `loadCodeAssist` at setup (`CA/setup.ts:217-236`).
  - It is re-polled with `mode: 'HEALTH_CHECK'` only from `/stats` (`CA/server.ts:296-313`,
    `packages/cli/src/ui/commands/statsCommand.ts:65-69`).
  - It is replaced by every pushed `remainingCredits`.
  - The UI holds the same `paidTier` object, which gets mutated, so the menus read the latest cached balance
    (`CFG:1707-1709`, `packages/cli/src/ui/AppContainer.tsx:884-890`).

### Display

`StatsDisplay` can render the row "Google AI Credits: N" (`StatsDisplay.tsx:323-333`), but `HistoryItemDisplay` does
not pass `creditBalance` to it (`HistoryItemDisplay.tsx:155-161`). Also, `statsCommand` reads the balance before it
awaits `refreshAvailableCredits()` (`statsCommand.ts:33-34` vs `:65-69`). In this commit, therefore, the balance appears
only in the overage menu.

### ACP and headless modes

`gemini-acp.md` already notes that the menus are unreachable in these modes. This pass adds three points:

- `billing.overageStrategy` from settings still reaches core (`packages/cli/src/config/config.ts:1101`), and
  `CodeAssistServer` reads it for every request.
- `resetBillingTurnState` is called only from the TUI (`useGeminiStream.ts:1715`).
- ACP subscribes only to `ApprovalModeChanged` (`packages/cli/src/acp/acpSession.ts:83-84`).

So with `always` in settings, an ACP session sends `enabled_credit_types` without telling the user. The "Using AI
Credits" notice is never forwarded to the ACP client, and nothing about credits is reported over ACP.

## Mapping to the draft overage entity

| Draft field | Gemini CLI source | Notes |
| ----------- | ----------------- | ----- |
| `id` | `creditType` (`GOOGLE_ONE_AI`) | Stable identifier for the credit kind. One overage entity per credit type |
| `state` | Derived from `overageStrategy`, the balance compared with 50, the model's eligibility, and the last error reason | `allowed` = `ask` and >= 50. `active` = `always` and >= 50 and eligible model, meaning opted in on the request; it is a separate question whether credits were actually consumed. `disabled` comes with one of the reasons in the next row |
| `state.reason` (disabled) | `user_disabled` (`never`), `insufficient_balance` (below 50 on the client, or `INSUFFICIENT_G1_CREDITS_BALANCE` from the server), `not_eligible` (no `availableCredits`), `model_not_eligible` | These reason names are proposed. Gemini has no wire field for a reason |
| `meter.remaining` | `remainingCredits[].creditAmount`, or `availableCredits[].creditAmount` | Integer count. In Gemini the balance is the only meter field |
| `meter.used` | `consumedCredits[].creditAmount` | Per response, not cumulative. A session or period total would have to be summed by the agent |
| `meter.limit` | none | There is no cap or allotment. Omit it |
| `meter.unit` / currency | none on the wire | The UI says "AI Credits". The draft should allow a free-form unit label such as `"credits"` with no currency |
| `meter.share` | none | It cannot be derived because there is no limit |
| `meter.reset` | none | Credits have no reset or period on the wire. A purchased balance just decreases. Whether credits expire is UNVERIFIED |
| `covers` (base limits) | Model IDs from `OVERAGE_ELIGIBLE_MODELS` | Gemini links credits to models, not to quota IDs. The draft should let `covers` reference model-scoped limits, or fall back to a list of models |
| Consent mode (not in the draft) | `overageStrategy` `ask`/`always`/`never` | Worth modeling as a policy next to the state. `ask` implies an interactive prompt each time a quota error occurs. The prompt's choice lasts for the current user prompt only |
| Threshold (not in the draft) | `MIN_CREDIT_BALANCE = 50` | A client-side minimum balance to enable credits. Could be an optional `meter.minimum` or stay agent-internal |

## Gaps and unverified items

- There is no public documentation of the Code Assist credits fields. `docs/resources/quota-and-pricing.md` does not
  mention AI credits, and the setting docs only describe `overageStrategy`. Units, pricing, how credits convert to
  requests or tokens, and whether credits expire are all UNVERIFIED.
- UNVERIFIED: whether the server bills credits only after the base quota is exhausted, or on any request that carries
  `enabled_credit_types`. The client sends the field on every eligible request while the strategy is `always`.
- Why the threshold is 50 is not explained. Some code comments say "credits > 0" and "credits === 0" (`CFH:84`, `:89`),
  which no longer match the `>= 50` and `< 50` logic.
- The `CreditType` enum may grow on the server. The client keeps entries that are not G1 but ignores them
  (`CA/server.ts:246-254`).
- When the overage menu offers credits after a capacity error (`MODEL_CAPACITY_*`), whether the retry succeeds is
  UNVERIFIED.
- Not examined: the a2a-server and the VS Code companion. Both `package.json` files mention billing or overage
  settings and were not read.

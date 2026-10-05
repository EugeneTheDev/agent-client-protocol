# Codex (native): overage-like mechanisms (credits, spend control, reset credits)

Scope: the parts of Codex's quota model that behave like overage, meaning usage that continues or is capped after the
included windows run out. This note builds on `codex-native.md` (base windows, delivery, polling) and
`codex-acp-adapter.md`, and does not repeat them.

Source: `github.com/openai/codex` at commit `823ea830c0fd418b09ff02d36cad9a1fff66465b`, the same commit as
`codex-native.md`. Paths are relative to `codex-rs/`. Public docs are cited by URL.

## Summary

- **Codex has one real overage mechanism: credits.** Credits are a prepaid balance that pays for usage after the
  included windows (`primary`/`secondary`) are used up.
  - The pricing docs say: "After you reach your included limits, available credits let you continue working"
    (https://learn.chatgpt.com/docs/pricing, the redirect target of `developers.openai.com/codex/pricing`).
  - The TUI acts on this. When credits are usable (`hasCredits || unlimited`), it stops showing percentage warnings and
    the switch-model prompt, and treats ordinary usage as recoverable even when `ordinaryUsageAllowed == false`
    (`tui/src/chatwidget/rate_limits.rs:186-188, 298-325, 362-364`; `tui/src/chatwidget/backend_banners.rs:317-327`).
- **There is no explicit "now spending credits" signal.** No header, event or field says that overage is active.
  - A client has to infer it: the ordinary allowance is exhausted (`ordinaryUsageAllowed == false`, or a window at
    100%), credits are usable, `spendControlReached != true`, and `rateLimitReachedType == null`.
  - The TUI's own predicate for "usage can continue" is exactly this (`tui/src/chatwidget/backend_banners.rs:322-327`).
    It never shows an "on credits" indicator.
- **Credits are account-wide, not per bucket.**
  - The headers are not family-prefixed, so every header family's snapshot carries the same credits
    (`codex-api/src/rate_limits.rs:82, 219-231`).
  - The usage endpoint attaches credits and spend control only to the default `codex` bucket. Per-model buckets get
    `None` (`backend-client/src/client.rs:617-653`).
  - The TUI reads credits only from the `codex` bucket (`tui/src/chatwidget/rate_limits.rs:308-325`).
- **Spend control (`individualLimit`, `spendControlReached`) is a cap on credit spend.**
  - It is a per-user monthly credit limit that a workspace owner or admin sets. The TUI labels it "Monthly credit
    limit" and shows "X of Y credits used" (`tui/src/status/rate_limits.rs:114, 341-352`).
  - On workspaces that have no fixed windows (usage-based or flexible pricing), it is the only limit reported, so there
    it acts as the effective base limit. This is UNVERIFIED and inferred from the public docs, see Gaps.
- **`rateLimitReachedType` mostly describes overage state.** Four of its five values are overage states:
  - `*_credits_depleted`: the credit pool is empty.
  - `*_usage_limit_reached`: the spend cap is reached.
  - Only `rate_limit_reached` is a base-limit state.
  - The `owner` and `member` variants select the remedy: add credits or raise the cap yourself, or ask the owner.
- **Reset credits are not overage.** They are discrete, earned or purchasable vouchers that reset the base windows
  immediately ("the eligible rate-limit windows were reset"). The user redeems one explicitly, with a confirmation step.
- **The user is never asked to consent to spending credits.** The backend applies credits automatically. The only
  confirmation dialogs are for redeeming a reset, for notifying the workspace owner, and for an optional model switch.

## Fields

### Overage state and meter (on `RateLimitSnapshot`, core snake_case / v2 camelCase)

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `credits` | `CreditsSnapshot` | nullable. Present only on the `codex` bucket from the usage endpoint. From headers, on every family | Credit (overage) pool state | `protocol/src/protocol.rs:2357`; `app-server-protocol/src/protocol/v2/account.rs:664-676` |
| `credits.has_credits` / `hasCredits` | `bool` | required inside `credits` | Credits are available or enabled for the account. The TUI's doc comment reads "Whether credits tracking is enabled for the account". It is treated as "usable credits" together with `unlimited` | `protocol/src/protocol.rs:2403-2408`; `tui/src/status/rate_limits.rs:121-122`; `tui/src/chatwidget/rate_limits.rs:186-188` |
| `credits.unlimited` | `bool` | required | Unlimited credits, so no balance applies | same |
| `credits.balance` | `string` (decimal) | nullable. Absent when the backend hides it, or when the header is empty | Remaining balance in **credits**, with no currency. The TUI rounds it to an integer and shows "N credits", or "Available" when it is hidden or not positive | `codex-api/src/rate_limits.rs:222-226`; `tui/src/status/rate_limits.rs:394-436` |
| backend `credits.approx_local_messages`, `approx_cloud_messages` | JSON arrays | optional | Approximate message counts the balance buys. Dropped by Codex | `codex-backend-openapi-models/src/models/credit_status_details.rs:24-39`; `backend-client/src/client.rs:756-764` |
| `individual_limit` / `individualLimit` | `SpendControlLimitSnapshot` | nullable. Only on the `codex` bucket. Only from the usage endpoint, never from headers | Per-user credit spend cap ("effective monthly credit limit from workspace spend controls") | `protocol/src/protocol.rs:2358`; `tui/src/status/rate_limits.rs:114-115`; `codex-api/src/rate_limits.rs:96` |
| `individualLimit.limit` / `.used` | `string` (decimal credits) | required | Absolute cap and absolute amount used, in credits | `protocol/src/protocol.rs:2411-2416`; `.../v2/account.rs:792-799` |
| `individualLimit.remaining_percent` | `i32` | required | Percentage remaining (not used). The TUI clamps it to 0..100 | same; `tui/src/status/rate_limits.rs:198` |
| `individualLimit.resets_at` | `i64` Unix seconds | required | Start of the next cap period. Monthly per the TUI label. Period type (calendar month or billing cycle) is not on the wire | same; `tui/src/status/rate_limits.rs:343` |
| backend `spend_control.individual_limit.{source, remaining, used_percent, reset_after_seconds}` | `string?`, `string`, `i32`, `i32` | `source` optional, the others required | Extra backend fields dropped by Codex. `source` is likely the origin of the cap (seat type, group or user override). UNVERIFIED | `codex-backend-openapi-models/src/models/spend_control_limit_details.rs:15-36`; `backend-client/src/client.rs:766-775` |
| `spend_control_reached` / `spendControlReached` | `bool` | nullable. "`None` is unavailable, not a sparse-update recovery" | Backend says the spend cap is hit. Can be present without `individual_limit` | `protocol/src/protocol.rs:2359-2360`; `codex-backend-openapi-models/src/models/spend_control_status_details.rs:15-24`; `backend-client/src/client.rs:671, 1002-1020` |
| `rate_limit_reached_type` / `rateLimitReachedType` | enum, see the next table | nullable. From the usage endpoint `{type}` wrapper or the `x-codex-rate-limit-reached-type` header on 429. Unknown backend values map to `null` | Why usage is blocked | `protocol/src/protocol.rs:2362-2389`; `codex-backend-openapi-models/src/models/rate_limit_status_payload.rs` (`RateLimitReachedType{type}`, `Unknown` via `serde(other)`); `backend-client/src/client.rs:690-708`; `codex-api/src/api_bridge.rs:186-193` |
| `GetAccountRateLimitsResponse.ordinaryUsageAllowed` | `bool` | nullable. Null means unavailable | Backend permission for **included** (non-credit) usage. It comes from usage endpoint `rate_limit.allowed`. `rate_limit.limit_reached` is dropped | `.../v2/account.rs:332-334`; `backend-client/src/client/rate_limit_resets.rs:38-43`; `codex-backend-openapi-models/src/models/rate_limit_status_details.rs:15-19` |
| `GetAccountRateLimitsResponse.rateLimitUpsell` | opaque JSON | nullable | Backend-owned banner with `banner_type`, `title`, `description`, `ctas[{action,label}]`, `reset_at?`, `model_slug?`, `blocked_model_slug?`, `fallback_model_slugs[]`, `presentation`, `request_url?` | `.../v2/account.rs:342-344`; `tui/src/backend_banners.rs:18-36` |

### `rateLimitReachedType` values

| Value | Describes | Remedy text the client shows | Source |
|---|---|---|---|
| `rate_limit_reached` | Base windows exhausted | Generic plan or promo copy | `protocol/src/error.rs:737-739` |
| `workspace_owner_credits_depleted` | Overage pool empty, and the viewer is an owner | "Your workspace is out of credits. Add credits to continue." | `protocol/src/error.rs:715-719` |
| `workspace_member_credits_depleted` | Overage pool empty, and the viewer is a member | "...Ask your workspace owner to refill in order to continue." The TUI offers "Notify owner?" | `protocol/src/error.rs:721-725`; `tui/src/chatwidget/rate_limits.rs:540-551` |
| `workspace_owner_usage_limit_reached` | Spend cap reached, and the viewer is an owner | "You hit your spend cap set in your workspace. Increase your spend cap to continue." | `protocol/src/error.rs:727-731` |
| `workspace_member_usage_limit_reached` | Spend cap reached, and the viewer is a member | "...Ask an owner to increase your spend cap to continue." The TUI offers "Request increase?" | `protocol/src/error.rs:733-736`; `tui/src/chatwidget/rate_limits.rs:552-555` |

### Reset credits (not overage)

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `rateLimitResetCredits.availableCount` | `i64` | required | Number of unredeemed resets | `.../v2/account.rs:350-358` |
| `rateLimitResetCredits.credits` | `RateLimitResetCredit[]` | nullable. Null means only the count is known. The list can be shorter than the count | Detail rows | same |
| `RateLimitResetCredit.{id, resetType, status, grantedAt, expiresAt, title, description}` | `string`, `codexRateLimits\|unknown`, `available\|redeeming\|redeemed\|unknown`, Unix s, Unix s or null, `string?`, `string?` | as listed | One voucher. It can expire | `.../v2/account.rs:363-398` |
| `account/rateLimitResetCredit/consume` params `{idempotencyKey, creditId?}` | JSON-RPC | `creditId` optional, and the backend picks one if omitted | Redeem a voucher | `.../v2/account.rs:403-411`; `app-server-protocol/src/protocol/common.rs:1333` |
| consume result `outcome` | `reset\|nothingToReset\|noCredit\|alreadyRedeemed` | required | "A reset credit was consumed and the eligible rate-limit windows were reset". The backend also returns `windows_reset: i64`, which is dropped | `.../v2/account.rs:416-431`; `backend-client/src/types.rs:111-125`; `app-server/src/request_processors/account_processor/rate_limit_resets.rs:81-94` |

### Adjacent credit meters and remediation

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `account/usage/read` → `threadUsage.{estimatedUsageCreditsMicros, estimatedUsageUsdMicros, groups[]}` | `i64` micro-credits, `i64?` micro-USD | `threadUsage` optional, USD nullable | Estimated credits spent by one thread, broken down by model, effort and speed. A status-line item that is "Enterprise workspaces only; omitted when unavailable" | `app-server-protocol/src/protocol/v2/thread_usage.rs:9-29`; `.../v2/account.rs:437-455`; `tui/src/bottom_pane/status_line_setup.rs:202-206` |
| analytics `on_demand_credits` | `f64` | optional | Credits consumed beyond included usage, per model or client, in the analytics reports only. This is the only field that names overage spend separately | `codex-backend-openapi-models/src/models/analytics.rs:29, 41, 151, 216`; `tui/src/analytics/normalize.rs:142` |
| `account/sendAddCreditsNudgeEmail` `{creditType: credits\|usage_limit}` → `{status: sent\|cooldown_active}` | JSON-RPC | required | A member asks the owner to add credits or raise the cap | `.../v2/account.rs:514-540`; `common.rs:1351` |
| `usage_limit_reached` 429 body `{type, code?, plan_type?, resets_at?, limit_window_minutes?}` plus headers `x-codex-active-limit`, `x-codex-rate-limit-reached-type`, `x-codex-promo-message` | HTTP | each optional | Turn-ending error that carries the reached type | `codex-api/src/api_bridge.rs:181-216, 340-352` |
| 429 codes `insufficient_quota`, `credit_balance_exhausted`, `organization_spend_limit_exceeded`, `project_spend_limit_exceeded`, `organization_usage_limit_exceeded` | HTTP `error.code` | n/a | Shown as `QuotaExceeded` ("Quota exceeded. Check your plan and billing details."), which reaches clients as `UsageLimitExceeded`. Platform (API key) billing, separate from ChatGPT credits | `codex-api/src/api_bridge.rs:218-229`; `protocol/src/error.rs:162-163, 469-472` |

## State and coverage semantics

1. **When credits are consumed.** Credits are consumed after the included allowance is exhausted, not alongside it.
   - The docs say so ("After you reach your included limits, available credits let you continue working";
     https://learn.chatgpt.com/docs/pricing).
   - The code agrees. The TUI suppresses window warnings and the switch-model prompt whenever credits are usable,
     because hitting 100% will not block the user (`tui/src/chatwidget/rate_limits.rs:308-325, 362-364`;
     `tui/src/chatwidget/usage_notice.rs:133-140`).
   - Recovery is allowed when `ordinaryUsageAllowed == true` **or** credits are usable
     (`tui/src/chatwidget/backend_banners.rs:322-327`).
   - On flexible-pricing Enterprise and Edu there are "no fixed rate limits. Usage scales with credits", so all usage
     is billed in credits (same URL).
   - Per-token credit rates are a pricing-page concept. Codex never computes or receives a rate.
2. **The "active" signal is inferred, not reported.**
   - The best client-side proxy is `ordinaryUsageAllowed == false && (credits.hasCredits || credits.unlimited) &&
     spendControlReached != true && rateLimitReachedType == null`.
   - `ordinaryUsageAllowed` is available only from `account/rateLimits/read`, not from the push notification.
   - The streamed `account/rateLimits/updated` carries `credits` from the response headers, but not
     `ordinaryUsageAllowed`.
3. **Coverage (which base limits credits cover).**
   - The wire never states coverage. Credits hang off the `codex` bucket only (`backend-client/src/client.rs:617-653`),
     so the implied coverage is the ordinary windows of the `codex` bucket.
   - For per-model buckets (`limit_name` set), the 429 message is "You've hit your usage limit for {limit_name}. Switch
     to another model now". This message is returned before any credits or plan copy (`protocol/src/error.rs:696-711`),
     which suggests credits do not cover per-model limits. UNVERIFIED.
4. **Disabled states.**
   - Credits depleted: `*_credits_depleted`. A finite `balance` is expected to read `0` or be hidden. UNVERIFIED.
   - Spend cap reached: `spendControlReached == true`, or `*_usage_limit_reached`.
   - Either case is treated as a "workspace hard stop". It cancels the "has credits" suppression
     (`tui/src/chatwidget/rate_limits.rs:298-312`) and triggers an immediate full refresh
     (`tui/src/app/app_server_events.rs:309-327`).
   - When a `UsageLimitExceeded` error arrives while the last known type was `*_credits_depleted`, the TUI rewrites it
     to the matching `*_usage_limit_reached` (`tui/src/chatwidget/turn_runtime.rs:441-458`). The reason is not
     documented in code. UNVERIFIED.
5. **Not available or not enabled.** `credits == null`, or `hasCredits == false && unlimited == false`. The `/status`
   row is then hidden (`tui/src/status/rate_limits.rs:394-402`). The protocol cannot tell "plan cannot buy credits"
   apart from "plan can buy credits but has none".
6. **Units.** All credit amounts are "credits", as decimal strings. There is no currency field. USD appears only as
   `estimatedUsageUsdMicros` on thread usage and `cost_usd` in analytics.
7. **Who enables and pays.** Plan copy (`protocol/src/error.rs:742-790`):
   - Plus, Pro, ProLite and ProMax: "visit https://chatgpt.com/settings/usage to purchase more credits", i.e. personal
     purchase.
   - Team and Business-like plans: "send a request to your admin".
   - Free and Go: upgrade.
   - Enterprise and Edu: no remedy.
   - Backend CTA actions (`tui/src/backend_banners/actions.rs:17-56`):
     - `add_credits` or `buy_credits` opens admin billing for workspace plans and the personal usage page otherwise;
     - `increase_spend_cap` opens `/admin/usage-limits/workspace`;
     - `buy_reset` opens a reset-purchase page;
     - `notify_owner` or `request_increase` sends the nudge email.
   - Per OpenAI Help Center articles, read only through search-result summaries because direct fetches returned 403
     (UNVERIFIED wording):
     - only workspace owners buy credits or configure automatic reload;
     - owners and admins set monthly credit limits per seat type, with per-user overrides;
     - Enterprise and Edu also have a workspace-level "overage limit" for usage after the shared credit pool is gone,
       and a usage period of "Monthly" or "Aligned to billing cycle".
   - No workspace-level overage limit appears in the Codex wire model. Only the per-user `individual_limit` does.
8. **Consent.** The TUI has no prompt to opt in to, or approve, spending credits. The confirmations that do exist:
   - "Use this reset?" with "Yes, use reset" or "No, go back" (`tui/src/chatwidget/usage.rs:245-270`);
   - the owner-nudge Yes/No banner (`tui/src/chatwidget/rate_limits.rs:534-581`);
   - the switch-model prompt, "Switch to {model} for lower credit usage?"
     (`tui/src/chatwidget/rate_limits.rs:462-528`).
9. **Reset credits.** These are a remedy for base limits, not an overage meter.
   - Discovery: `/usage` → "Redeem reset" (`tui/src/chatwidget/usage.rs:31-68`). After a limit is hit, the TUI shows
     the hint "You have N resets available. Run /usage to use one." (`tui/src/chatwidget/usage.rs:551-560`).
   - Periodic polls skip the reset-credit detail lookup (`excludeResetCreditDetails`,
     `.../v2/account.rs:320-323`).
10. **Reserve fallback (product-specific, adjacent).** When ordinary usage is blocked, the backend can return an upsell
    banner that names `blocked_model_slug` and an ordered list of `fallback_model_slugs`. A client that sets
    `supportsLunaReserve` can switch to a reserve model (`tui/src/backend_banners.rs:11, 18-36`;
    `.../v2/account.rs:315-319`; `protocol/src/error.rs:693-694`, where the comment reads "Reserve is a fallback for
    exhausted ordinary usage"). It is a model fallback, not a metered overage. Its billing is UNVERIFIED. Its names are
    product-specific and should not be copied.

## Mapping to the draft overage entity

| Draft field | Codex source | Notes |
|---|---|---|
| `id` | none native. Synthesize, e.g. `"credits"` | Credits are a single account-wide pool. There is no id on the wire |
| `state = allowed` | `credits.hasCredits \|\| credits.unlimited`, with no hard stop | "Overage is available if needed" |
| `state = active` | inferred: `ordinaryUsageAllowed == false` (or a covered window ≥ 100%) and `allowed` | No native signal. Pull-only, because `ordinaryUsageAllowed` is absent from pushes |
| `state = disabled`, reason `depleted` | `rateLimitReachedType ∈ {workspace_*_credits_depleted}` | `owner` or `member` maps to the actor who can fix it. Worth a separate `actionableBy` or remedy hint |
| `state = disabled`, reason `cap_reached` | `spendControlReached == true` or `rateLimitReachedType ∈ {workspace_*_usage_limit_reached}` | This could instead be modeled as a second overage meter (the cap) reaching its limit |
| `state = disabled`, reason `not_enabled` | `credits == null` or both bools false | Cannot be told apart from "unavailable on this plan" |
| `meter.remaining` | `credits.balance` (decimal string, credits) | Can be hidden. Not meaningful when `unlimited` |
| `meter.limit` / `meter.used` / `meter.share` / `meter.reset` | `individualLimit.limit`, `.used`, `100 - remainingPercent`, `.resetsAt` | These belong to the **spend cap**, not the balance. The balance has no limit or reset |
| `meter.unit` | `"credits"` (implicit) | No currency. Decimal strings, so precision must be kept |
| `covers` | implicit: the `codex` bucket's `primary` + `secondary` | Per-model buckets are probably not covered (UNVERIFIED) |
| `unlimited` | `credits.unlimited` | Needs a flag, or a null limit with explicit semantics |

The draft entity should therefore allow for two meters: a balance (remaining only) and a spend cap
(limit/used/reset). Alternatively, it could model the spend cap as its own limit that the overage is subject to.

Reset credits do not fit the overage entity. They are better modeled as an available action on base limits (a count
plus a redeem method).

## Gaps and unverified items

- UNVERIFIED: credits do not cover per-model (`limit_name`) buckets. This is inferred from the error-message ordering
  and from credits being attached only to `codex`.
- UNVERIFIED: the exact meaning of `has_credits` (enabled, or balance > 0). The TUI comment says "tracking is enabled",
  yet the TUI also treats it as "usable credits". A finite balance of `0` with `has_credits: true` renders as
  "Available".
- UNVERIFIED: whether `spend_control_reached == true` always comes with a `*_usage_limit_reached` reason.
- UNVERIFIED: the Help Center content (owner-only purchase, auto-reload, the seat, group and user limit hierarchy,
  the workspace overage limit, period options). Direct fetches returned 403, so this rests on search-result summaries
  of https://help.openai.com/en/articles/20001155-managing-credits-and-spend-controls-in-chatgpt-business and
  https://help.openai.com/en/articles/20001001-manage-usage-limits-and-overages-in-chatgpt-enterprise-and-edu.
- UNVERIFIED: on usage-based Business and Enterprise accounts, `/wham/usage` returns only `spend_control` and no
  `rate_limit`. The only evidence is a third-party issue title
  (https://github.com/zioder/TaskbarQuota/issues/102).
- The reason the TUI rewrites `*_credits_depleted` to `*_usage_limit_reached` on a `UsageLimitExceeded` error is
  undocumented.
- The meaning of the spend-control `source` field is unknown, because Codex drops it.
- I did not verify whether WebSocket `codex.rate_limits` events ever carry a changing `credits.balance` mid-turn,
  which would give a near-live overage meter.

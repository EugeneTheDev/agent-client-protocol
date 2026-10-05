# Codex (native): rate-limit and quota data model and delivery

Scope: Codex's own data model, independent of ACP. The ACP adapter is out of scope.

Source: `github.com/openai/codex` at commit `823ea830c0fd418b09ff02d36cad9a1fff66465b` (2026-10-05). All paths below
are relative to `codex-rs/` at that commit. URL form:
`https://github.com/openai/codex/blob/823ea830c0fd418b09ff02d36cad9a1fff66465b/codex-rs/<path>#L<line>`.

## Summary

- **One snapshot shape everywhere.** Codex models quota as a `RateLimitSnapshot` per metered bucket (`limit_id`). Each
  snapshot has:
  - up to two percentage windows (`primary`, `secondary`), each with `used_percent`, `window_minutes` and `resets_at`;
  - a credits block (`has_credits`, `unlimited`, `balance` as a string);
  - a spend-control block (`individual_limit`, `spend_control_reached`);
  - `plan_type`;
  - a "reached" reason enum (`rate_limit_reached_type`).
- **No absolute counts for windows.** Windows carry no absolute limit, used or remaining counts, only a percentage.
  Absolute numbers appear only for spend control (`limit` and `used` as decimal strings) and credits (`balance` string).
- **Three sources feed the same type.**
  1. Response headers on every Responses API call (`x-codex-primary-used-percent` and related headers, plus
     per-model families such as `x-codex-bengalfox-*`).
  2. An in-band WebSocket event, `codex.rate_limits`.
  3. A backend usage endpoint (`GET {chatgpt_base_url}/wham/usage` or `/api/codex/usage`), which is the only source
     for `plan_type`, spend control, additional per-model limits with `limit_name`, and `normal_model_slug`.
- **Core delivery (push).** Core pushes the latest snapshot inside `EventMsg::TokenCount { info, rate_limits }`, next to
  token usage and context-window size.
- **App-server v2 delivery.** The app-server splits that event into `thread/tokenUsage/updated` and
  `account/rateLimits/updated`. The updated notification is a sparse push. Clients also poll `account/rateLimits/read`,
  which returns the full multi-bucket view (`rateLimitsByLimitId`).
- **API-key users.** `account/rateLimits/read` is rejected for API-key users. The TUI does not poll for them.
- **TUI.** Warns at 50/75/90/95% used, with a plan-dependent floor. `/status` and the status line render
  "N% left" bars that are labeled by window duration ("5h", "weekly") rather than by primary/secondary position.

## Fields

### Core protocol (`codex_protocol`, snake_case on the wire, used in `EventMsg` and rollout files)

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `TokenCountEvent.info` | `TokenUsageInfo` | yes (`Option`) | Cumulative and last-turn token usage plus context window | `protocol/src/protocol.rs:2343-2346` |
| `TokenCountEvent.rate_limits` | `RateLimitSnapshot` | yes (`Option`) | Latest quota snapshot held by the session | `protocol/src/protocol.rs:2345` |
| `RateLimitSnapshot.limit_id` | `string` | `Option`; core normalizes `None` to `"codex"` | Metered bucket id, e.g. `codex`, `codex_other`, `codex_bengalfox` | `protocol/src/protocol.rs:2350`; `core/src/state/session.rs:419-421` |
| `RateLimitSnapshot.limit_name` | `string` | `Option` | Display name of the bucket, typically a model name (e.g. `gpt-5.2-codex-sonic` in tests) | `protocol/src/protocol.rs:2351`; `codex-api/src/rate_limits.rs:333-348` |
| `RateLimitSnapshot.normal_model_slug` | `string` | `Option`, `skip_serializing_if = None` | "Normal model metadata for a quota alias; never a replacement for the request model" | `protocol/src/protocol.rs:2352-2354` |
| `RateLimitSnapshot.primary` | `RateLimitWindow` | `Option` | First window | `protocol/src/protocol.rs:2355` |
| `RateLimitSnapshot.secondary` | `RateLimitWindow` | `Option` | Second window | `protocol/src/protocol.rs:2356` |
| `RateLimitSnapshot.credits` | `CreditsSnapshot` | `Option` | Credits state | `protocol/src/protocol.rs:2357` |
| `RateLimitSnapshot.individual_limit` | `SpendControlLimitSnapshot` | `Option` | Per-user spend cap within a workspace | `protocol/src/protocol.rs:2358` |
| `RateLimitSnapshot.spend_control_reached` | `bool` | `Option` | "`None` is unavailable, not a sparse-update recovery" | `protocol/src/protocol.rs:2359-2360` |
| `RateLimitSnapshot.plan_type` | `PlanType` enum (lowercase strings) | `Option` | Subscription plan | `protocol/src/protocol.rs:2361`; `protocol/src/account.rs:9-45` |
| `RateLimitSnapshot.rate_limit_reached_type` | enum: `rate_limit_reached`, `workspace_owner_credits_depleted`, `workspace_member_credits_depleted`, `workspace_owner_usage_limit_reached`, `workspace_member_usage_limit_reached` | `Option` | Why usage is blocked | `protocol/src/protocol.rs:2362-2389` |
| `RateLimitWindow.used_percent` | `f64` (0-100) | required | "Percentage (0-100) of the window that has been consumed" | `protocol/src/protocol.rs:2392-2395` |
| `RateLimitWindow.window_minutes` | `i64` | `Option` | "Rolling window duration, in minutes" | `protocol/src/protocol.rs:2396-2398` |
| `RateLimitWindow.resets_at` | `i64` Unix seconds | `Option` | "Unix timestamp ... when the window resets" | `protocol/src/protocol.rs:2399-2401` |
| `CreditsSnapshot.has_credits` | `bool` | required | Account has purchasable or workspace credits | `protocol/src/protocol.rs:2404-2408` |
| `CreditsSnapshot.unlimited` | `bool` | required | Unlimited credits | same |
| `CreditsSnapshot.balance` | `string` | `Option` | Balance as a decimal string, not a number | same |
| `SpendControlLimitSnapshot.limit` / `.used` | `string` | required | Absolute spend cap and amount used (credits, decimal strings) | `protocol/src/protocol.rs:2411-2416` |
| `SpendControlLimitSnapshot.remaining_percent` | `i32` | required | Percent of the cap remaining (note: remaining, not used) | same |
| `SpendControlLimitSnapshot.resets_at` | `i64` Unix seconds | required | Reset time of the cap | same |
| `PlanType` values | `free, go, plus, pro, prolite, promax, team, self_serve_business_prolite, self_serve_business_usage_based, business, ent26, enterprise_cbp_automation, enterprise_cbp_usage_based, enterprise, edu, edu_plus, edu_pro, unknown` (catch-all) | n/a | Plan enum | `protocol/src/account.rs:9-45` |
| `TokenUsageInfo.total_token_usage` / `.last_token_usage` | `TokenUsage` | required | Running session total and the last response | `protocol/src/protocol.rs:2276-2282` |
| `TokenUsageInfo.model_context_window` | `i64` | `Option` (marked TODO "make this not optional") | Model context window size in tokens | `protocol/src/protocol.rs:2279-2281` |
| `TokenUsage.{input_tokens, cached_input_tokens, cache_write_input_tokens, output_tokens, reasoning_output_tokens, total_tokens}` | `i64` | required (`cache_write_input_tokens` has `serde(default)`) | Token counters | `protocol/src/protocol.rs:2241-2261` |

### App-server protocol v2 (camelCase JSON-RPC)

The v2 `Option` fields have no `skip_serializing_if`, so they serialize as explicit `null`, not as omitted keys. See the
test at `app-server/src/outgoing_message.rs:1005-1050` and the generated TypeScript
`app-server-protocol/schema/typescript/v2/RateLimitSnapshot.ts` (`limitId: string | null`, etc.).

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `RateLimitSnapshot.{limitId, limitName, normalModelSlug, primary, secondary, credits, individualLimit, spendControlReached, planType, rateLimitReachedType}` | same as core, camelCase | all nullable | Same as core | `app-server-protocol/src/protocol/v2/account.rs:664-676` |
| `RateLimitWindow.usedPercent` | `i32` | required | Core `f64` rounded to an integer (`value.used_percent.round() as i32`) | `.../v2/account.rs:752-768` |
| `RateLimitWindow.windowDurationMins` | `number` | nullable | Renamed from core `window_minutes` | `.../v2/account.rs:754-755` |
| `RateLimitWindow.resetsAt` | `number` (Unix seconds) | nullable | Reset time | `.../v2/account.rs:756-757` |
| `CreditsSnapshot.{hasCredits, unlimited, balance}` | `bool, bool, string\|null` | `balance` nullable | Credits | `.../v2/account.rs:773-778` |
| `SpendControlLimitSnapshot.{limit, used, remainingPercent, resetsAt}` | `string, string, i32, number` | required | Spend cap | `.../v2/account.rs:792-799` |
| `AccountRateLimitsUpdatedNotification.rateLimits` | `RateLimitSnapshot` | required | One snapshot (one bucket) per notification | `.../v2/account.rs:650-659` |
| `GetAccountRateLimitsParams.supportsLunaReserve` | `bool` | default false, omitted when false | Client capability flag (opt-in fallback offering) | `.../v2/account.rs:315-324` |
| `GetAccountRateLimitsParams.excludeResetCreditDetails` | `bool` | default false | Skip the reset-credit detail lookup "for background usage polls" | same |
| `GetAccountRateLimitsResponse.ordinaryUsageAllowed` | `bool` | nullable | "Backend permission for ordinary included usage... Null means unavailable; clients must not infer recovery from percentages or reset times" | `.../v2/account.rs:331-334` |
| `GetAccountRateLimitsResponse.rateLimits` | `RateLimitSnapshot` | required | "Backward-compatible single-bucket view" (the `codex` bucket, else the first) | `.../v2/account.rs:335-336`; `app-server/src/request_processors/account_processor.rs:1251-1256` |
| `GetAccountRateLimitsResponse.rateLimitsByLimitId` | `map<string, RateLimitSnapshot>` | nullable (always `Some` from this server) | "Multi-bucket view keyed by metered `limit_id`" | `.../v2/account.rs:337-338`; `account_processor.rs:1287-1292` |
| `GetAccountRateLimitsResponse.rateLimitResetCredits` | `{availableCount: i64, credits: RateLimitResetCredit[] \| null}` | nullable | Earned "reset my window" credits. `credits: null` means only the count is known | `.../v2/account.rs:339, 350-358` |
| `RateLimitResetCredit.{id, resetType, status, grantedAt, expiresAt, title, description}` | `string, enum{codexRateLimits, unknown}, enum{available, redeeming, redeemed, unknown}, number, number\|null, string\|null, string\|null` | as listed | One reset credit | `.../v2/account.rs:363-398` |
| `GetAccountRateLimitsResponse.accountId` | `string` | nullable | Account the snapshot belongs to | `.../v2/account.rs:340-341` |
| `GetAccountRateLimitsResponse.rateLimitUpsell` | opaque JSON | nullable | Backend-owned banner (snake_case inside) | `.../v2/account.rs:342-344` |
| `ThreadTokenUsageUpdatedNotification.{threadId, turnId, tokenUsage}` | `string, string, ThreadTokenUsage` | required | Token-usage half of `TokenCount` | `app-server-protocol/src/protocol/v2/thread.rs:1894-1898` |
| `ThreadTokenUsage.{total, last, modelContextWindow}` | `TokenUsageBreakdown, TokenUsageBreakdown, number\|null` | `modelContextWindow` nullable | Session totals, last response, context window | `.../v2/thread.rs:1934-1940` |

### Wire inputs (where values come from)

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `x-<family>-primary-used-percent` | float header | absent means no window | Primary used percent. Default family is `codex` (`x-codex-primary-used-percent`) | `codex-api/src/rate_limits.rs:61-73` |
| `x-<family>-primary-window-minutes` / `-primary-reset-at` | int headers | optional | Window length in minutes and reset time in Unix seconds | same |
| `x-<family>-secondary-used-percent` / `-secondary-window-minutes` / `-secondary-reset-at` | headers | optional | Secondary window | `codex-api/src/rate_limits.rs:75-80` |
| `x-<family>-limit-name` | string header | optional | `limit_name` | `codex-api/src/rate_limits.rs:84-88` |
| `x-codex-credits-has-credits`, `x-codex-credits-unlimited`, `x-codex-credits-balance` | bool (`true`/`1`/`false`/`0`) and string headers | snapshot exists only if both bools parse | Credits. Not family-prefixed, so every family's snapshot gets the same credits | `codex-api/src/rate_limits.rs:219-231` |
| `x-codex-active-limit` | header (on 429) | optional | Which family caused `usage_limit_reached` | `codex-api/src/api_bridge.rs:186-195, 275` |
| `x-codex-rate-limit-reached-type` | header (on 429) | optional | `rate_limit_reached_type` | `codex-api/src/rate_limits.rs:188-193` |
| `x-codex-promo-message` | header (on 429) | optional | Upsell text | `codex-api/src/rate_limits.rs:181-186` |
| 429 body `error.{type, code, plan_type, resets_at, limit_window_minutes}` | JSON | each optional | `type == "usage_limit_reached"` produces `UsageLimitReachedError`. `usage_not_included` and `insufficient_quota` (or the `credit_balance_exhausted` / `*_spend_limit_exceeded` codes) produce other errors | `codex-api/src/api_bridge.rs:181-233, 340-352` |
| WS event `{type:"codex.rate_limits", plan_type?, rate_limits?:{primary?,secondary?:{used_percent, window_minutes?, reset_at?}}, credits?:{has_credits, unlimited, balance?}, metered_limit_name?, limit_name?}` | JSON | as marked | WebSocket transport equivalent of the headers. `metered_limit_name` (or `limit_name`) becomes `limit_id` | `codex-api/src/rate_limits.rs:104-169`; `codex-api/src/endpoint/responses_websocket.rs:775-779` |
| Usage endpoint `rate_limit.{allowed, limit_reached, primary_window, secondary_window}` | JSON | windows optional | Per-window fields: `{used_percent:i32, limit_window_seconds:i32, reset_after_seconds:i32, reset_at:i32}` | `codex-backend-openapi-models/src/models/rate_limit_status_details.rs:16-34`; `.../rate_limit_window_snapshot.rs:14-22` |
| Usage endpoint top-level `plan_type, credits, spend_control, additional_rate_limits[], rate_limit_reached_type, rate_limit_reset_credits, account_id, user_id, rate_limit_upsell` | JSON | all except `plan_type` optional | `additional_rate_limits[]` = `{limit_name, metered_feature, rate_limit, normal_model_slug?}` | `.../models/rate_limit_status_payload.rs:17-53`; `.../additional_rate_limit_details.rs:17-27`; `backend-client/src/types.rs:54-75` |
| Usage endpoint `credits.{approx_local_messages, approx_cloud_messages}` | JSON arrays | optional | Present in the backend model but dropped by Codex's mapping | `.../models/credit_status_details.rs:32-39`; `backend-client/src/client.rs:756-764` |
| Usage endpoint `spend_control.individual_limit.{source?, limit, used, remaining, used_percent, remaining_percent, reset_after_seconds, reset_at}` | JSON | `source` optional | Codex keeps only `limit, used, remaining_percent, reset_at` | `.../models/spend_control_limit_details.rs:22-36`; `backend-client/src/client.rs:766-775` |

## Delivery mechanism

1. **Response headers (push, once per model request).** For each streamed Responses call, Codex parses every header
   family into snapshots and emits one `ResponseEvent::RateLimits` per family before the stream body
   (`codex-api/src/sse/responses.rs:43, 78-80`).
   - `parse_all_rate_limits` always yields a default `codex` snapshot, even with no headers. Other families are
     discovered by scanning for `x-*-primary-used-percent` and kept only if they carry data
     (`codex-api/src/rate_limits.rs:28-51, 259-268`).
   - A window is dropped when `used_percent == 0`, `window_minutes` is 0 or absent, and `resets_at` is absent
     (`codex-api/src/rate_limits.rs:207-215`).
   - Over WebSocket, the server instead sends an in-band `codex.rate_limits` event
     (`codex-api/src/endpoint/responses_websocket.rs:775-779`).
2. **Core session state.** `ResponseEvent::RateLimits` updates the session's single `latest_rate_limits`. Emission is
   deferred "until token usage is available to avoid duplicate TokenCount events" (`core/src/session/turn.rs:2957-2962`).
   - The merge keeps the previous `credits`, `individual_limit`, `spend_control_reached` and `plan_type` when the new
     snapshot lacks them, and defaults `limit_id` to `"codex"` (`core/src/state/session.rs:327-332, 412-436`).
   - The state is one `Option<RateLimitSnapshot>` (`core/src/state/session.rs:75`), not a per-`limit_id` map. With
     several header families, the last-parsed family overwrites the earlier ones, and credits or plan can carry over
     from a different bucket. This is an inference from reading the code. I found no test for it.
3. **`EventMsg::TokenCount`.** `send_token_count_event` emits `TokenCountEvent { info, rate_limits }` from state
   (`core/src/session/mod.rs:4859-4866`).
   - It fires after a sampling step that saw RateLimits or Completed, once pending tools resolve
     (`core/src/session/turn.rs:3001-3007, 3191-3197`).
   - It also fires immediately on `UsageLimitReached`, using the snapshot parsed from 429 headers
     (`core/src/session/turn.rs:1720-1726`; `core/src/session/mod.rs:4825-4832`), on context-window overflow
     (`set_total_tokens_full`, `core/src/session/mod.rs:4868-4874`), and during compaction (`core/src/compact.rs:812-813`).
4. **Local file.** `TokenCount` events, including `rate_limits`, are persisted in the rollout JSONL
   (`rollout/src/policy.rs:127-140`; example line in `rollout/src/tests.rs:56`). This makes a local on-disk record of
   past snapshots.
5. **App-server v2 (JSON-RPC).**
   - Push: `account/rateLimits/updated`. Each `TokenCount` with `rate_limits: Some` becomes one
     `account/rateLimits/updated`, and `info: Some` becomes `thread/tokenUsage/updated`
     (`app-server/src/bespoke_event_handling.rs:1026-1029, 1579-1605`; method names in
     `app-server-protocol/src/protocol/common.rs:1963, 2004`).
   - The doc comment calls the notification a "Sparse rolling rate-limit update. Clients should merge available values
     into the most recent `account/rateLimits/read` response or refetch that snapshot. Nullable account metadata may
     be unavailable in a rolling update and does not clear a previously observed value"
     (`.../v2/account.rs:650-656`).
   - Pull: `account/rateLimits/read` (`common.rs:1327-1331`). The handler requires Codex-backend auth, calls the usage
     endpoint, and returns both the legacy single bucket and the `rateLimitsByLimitId` map
     (`app-server/src/request_processors/account_processor.rs:1190-1301`).
   - The usage URL is `{base}/wham/usage` (ChatGPT path style) or `{base}/api/codex/usage`
     (`backend-client/src/client/rate_limit_resets.rs:124-129`).
   - Window seconds are converted to minutes by rounding up, and a value of 0 or less becomes `None`
     (`backend-client/src/client.rs:741-754, 812-819`).
   - Additional per-model buckets get `limit_id = metered_feature` and `limit_name = limit_name`, and inherit
     `plan_type` from the main bucket. They have no credits or spend control (`backend-client/src/client.rs:613-653`;
     `backend-client/src/client/rate_limit_resets.rs:34-66`).
   - Related methods: `account/rateLimitResetCredit/consume` (`common.rs:1333-1336`). There is also a separate
     `account/usage/read` method that returns an account token-usage summary (lifetime and daily tokens), which is not
     a quota (`common.rs:1339`; `.../v2/account.rs:432-505`).
6. **TUI (an app-server client).**
   - Poll: it polls `account/rateLimits/read` adaptively, every 60 s by default, 30 s at 75% or more used, 15 s at 90%
     or more, and 5 s at 99% or more. Only the `codex` bucket and the bucket whose `limit_name` equals the current
     model count toward that percentage (`tui/src/chatwidget/rate_limits.rs:191-218`; loop in
     `tui/src/app/startup.rs:1234-1237`; deadline in `tui/src/app/rate_limit_refresh.rs:36-45`).
   - Polling runs only when `requires_openai_auth && has_chatgpt_account` (`tui/src/chatwidget/rate_limits.rs:419-421`).
     Periodic polls set `excludeResetCreditDetails`, and `/status` forces a refresh
     (`tui/src/app/background_requests.rs:814-842`; `tui/src/chatwidget/slash_dispatch.rs:544-557`).
   - Push: it also consumes `account/rateLimits/updated` (`tui/src/app/app_server_events.rs:309-330`). Streamed updates
     drive warnings and recovery but "must not overwrite status data". Only full reads populate the per-bucket display
     map (`tui/src/chatwidget/rate_limits.rs:374-388`).

## Windows and semantics

- **Primary and secondary are positional, not semantic.** The protocol defines no fixed duration. The duration comes
  from `window_minutes` (`windowDurationMins` in v2).
  - The TUI derives labels by matching that duration within ±5%: "5h", "daily", "weekly", "monthly", "annual". The
    fallbacks are "usage" and "secondary usage" (`tui/src/chatwidget/rate_limits.rs:103-146`).
  - The status-line items `five-hour-limit` and `weekly-limit` are described as "primary" and "secondary"
    (`tui/src/bottom_pane/status_line_setup.rs:110-114, 189-194`). In practice they select a window by matching the
    "5h" or "weekly" label, then fall back to primary or secondary (`tui/src/chatwidget/status_surfaces.rs:760-775,
    1119-1150`).
  - The usual pairing of primary = 5h and secondary = weekly is a convention of the backend data, not a schema
    guarantee. A test uses primary=60 and secondary=10080 (`rollout/src/tests.rs:56`). Another uses a 1440-minute
    primary (`codex-api/src/rate_limits.rs:306-330`).
- **Per-model limits.** Separate buckets keyed by `limit_id`, such as `codex_bengalfox` with
  `limit_name: "gpt-5.2-codex-sonic"`. They come from the header families or from the usage endpoint's
  `additional_rate_limits`.
  - The usage-limit error message names the model: "You've hit your usage limit for {limit_name}. Switch to another
    model now" (`protocol/src/error.rs:697-711`).
  - `normal_model_slug` maps a quota alias to the normal model's display metadata.
- **Reported values.**
  - Windows report `used_percent` only. Remaining is derived by clients as `100 - used`; the TUI shows "N% left"
    (`tui/src/status/rate_limits.rs:369-389`).
  - Reset time is an absolute Unix-seconds timestamp. The backend also sends `reset_after_seconds`, which Codex drops.
  - No absolute limit or count exists for windows.
  - Spend control is the exception: it has absolute `limit` and `used` as strings plus `remaining_percent`. The TUI
    shows it as "Monthly credit limit" with "X of Y credits used" (`tui/src/status/rate_limits.rs:341-352`).
- **Overage.** Represented by credits (`has_credits`, `unlimited`, `balance`). When usable credits exist, the TUI
  suppresses percentage warnings and the switch-model prompt, since usage continues on credits
  (`tui/src/chatwidget/rate_limits.rs:186-188, 300-323`).
- **Status and severity.**
  - Status comes from three fields: `rate_limit_reached_type` (reason), `spend_control_reached` (bool), and
    `ordinaryUsageAllowed` (read-only, nullable, explicitly "do not infer recovery from percentages or reset times").
  - There is no severity enum. Severity is computed client-side:
    - warnings at 50/75/90/95% used, with only thresholds at or above a floor emitted. The floor is 50 for
      Plus or Team on a ~5h window and 75 otherwise (`tui/src/chatwidget/rate_limits.rs:19, 30-101`;
      `tui/src/chatwidget/usage_notice.rs:154-164`);
    - a switch-model prompt at 90% (`tui/src/chatwidget/rate_limits.rs:16, 350-371`);
    - no warnings once a window is at exactly 100% (`tui/src/chatwidget/rate_limits.rs:38-43`).
  - Warning text: "Heads up, you have less than {N}% of your {5h|weekly|…} limit left. Run /status for a breakdown."
- **Staleness.** `/status` marks data stale when it is more than 15 minutes old (`tui/src/status/rate_limits.rs:65-66,
  233-256`). It distinguishes `Available`, `Stale`, `Unavailable` (fetched but empty), and `Missing` (`:52-63`).
- **Relation to tokens and context.**
  - `TokenCountEvent` bundles `info` (`total_token_usage`, `last_token_usage`, `model_context_window`) with
    `rate_limits` in one event. App-server v2 splits them into `thread/tokenUsage/updated` (per thread and turn) and
    `account/rateLimits/updated` (account-wide, no thread id).
  - Quota percentages are not derived from token counts. They are server-reported.
  - Context occupancy is computed client-side from `last`/`total` and `modelContextWindow`. On context overflow, core
    fills usage to the full window (`protocol/src/protocol.rs:2318-2340`).
  - A separate, non-replayed `RawResponseCompletedEvent` carries exact per-response usage plus
    `usage_metadata.amount` (`protocol/src/protocol.rs:1929-1936`; `.../v2/thread.rs:1900-1919`).
- **API-key users.**
  - `account/rateLimits/read` returns an invalid-request error, "chatgpt authentication required to read rate limits",
    when `!auth.uses_codex_backend()` (`app-server/src/request_processors/account_processor.rs:1202-1206`).
    `ApiKey`, `BedrockApiKey` and `BedrockAccessKeys` are non-Codex-backend modes (`protocol/src/auth.rs:54-63`).
  - The TUI does not poll and shows no rate-limit rows for them (`tui/src/chatwidget/rate_limits.rs:419-421`;
    `tui/src/chatwidget/slash_dispatch.rs:545-556`).
  - `plan_type`, spend control, reset credits and additional limits are absent because they come only from the usage
    endpoint.

## Gaps and unverified items

- UNVERIFIED: whether `api.openai.com` (API-key auth) ever sends `x-codex-*` headers. In code, `parse_all_rate_limits`
  always returns a default `codex` snapshot with all-null windows (`codex-api/src/rate_limits.rs:370-381`), and core
  records it unconditionally. API-key sessions therefore likely emit `TokenCount.rate_limits` and
  `account/rateLimits/updated` with `limitId: "codex"` and everything else null. I have not confirmed this at runtime.
- Inference, not tested: core keeps a single `latest_rate_limits`. When a response carries several header families,
  the `TokenCount` after that step reflects only the last family parsed, which is the non-`codex` family sorted last.
  App-server pushes are therefore not a reliable multi-bucket view. Clients rely on `rateLimitsByLimitId` from
  `account/rateLimits/read` for that.
- UNVERIFIED: the actual production window lengths (5h = 300 min and weekly = 10080 min). They are inferred from the TUI
  label logic and the Plus/Team 5h special case, not from a backend contract.
- I did not check upstream docs (`developers.openai.com/codex`) for user-facing descriptions of `/status` or limits.
  `codex-rs/app-server/README.md` does not mention `rateLimits`.
- The meaning of `RateLimitStatusDetails.allowed` and `limit_reached` beyond their names is backend-defined. Codex maps
  `allowed` to `ordinaryUsageAllowed` and discards `limit_reached` (`backend-client/src/client/rate_limit_resets.rs:38-43`).
- Field and wire names like `supports_luna_reserve` and `x-openai-codex-luna-reserve` are product-specific. The RFD
  should not copy them.

# Claude Code (native): rate-limit and quota data model and delivery

Scope: Claude Code CLI and Claude Agent SDK on their own, without ACP. The ACP adapter is out of scope.

Versions examined: `@anthropic-ai/claude-agent-sdk@0.3.289` (npm tarball, `sdk.d.ts`), which bundles CLI `2.1.289`
(`manifest.json`). I also read the locally installed CLI binary `~/.local/share/claude/versions/2.1.289` (Bun
bundle). Below, `bin@N` is a byte offset into that binary where the minified JS was read. Anything drawn only from the
bundle is undocumented, internal behavior and may change in any release.

Doc sources (fetched 2026-10-05):

- SL = https://code.claude.com/docs/en/statusline
- COSTS = https://code.claude.com/docs/en/costs
- CMDS = https://code.claude.com/docs/en/commands
- ERR = https://code.claude.com/docs/en/errors
- IM = https://code.claude.com/docs/en/interactive-mode
- TS = https://code.claude.com/docs/en/agent-sdk/typescript
- GW = https://code.claude.com/docs/en/claude-apps-gateway-spend-limits
- APIRL = https://platform.claude.com/docs/en/api/rate-limits

## Summary

Claude Code has two separate quota regimes:

1. **claude.ai subscription "unified" limits** (Pro, Max, Team, Enterprise; also a Claude apps gateway spend cap). They
   are expressed as **utilization of rolling windows**: 5-hour "session", 7-day "weekly", per-model weekly (Opus,
   Sonnet, plus server-named model buckets), and an "overage" bucket (usage credits / extra usage, monthly spend). The
   data contains **no absolute limits and no remaining amounts**, only fractions or percentages plus reset times. The
   only exception is overage spend in currency minor units.
2. **API-key rate limits** (Console API key, Bedrock, Vertex). These are the documented `anthropic-ratelimit-*`
   RPM/ITPM/OTPM headers. Claude Code does **not** surface them as quota data. They show up only as 429 errors and
   retries (`error: 'rate_limit'`, `system/api_retry`).

There are three delivery paths:

- **Push, per API response.** The server sends `anthropic-ratelimit-unified-*` response headers. The CLI turns them into
  `SDKRateLimitInfo`, emits `rate_limit_event` on change, and builds the status line `rate_limits` from them.
- **Poll, on demand.** `/usage`, and the SDK `get_usage` control request, fetch a claude.ai usage endpoint that returns
  per-window utilization, server-graded `limits[]` rows with `severity`, and extra-usage spend.
- **Text.** Warnings and limit messages in the TUI, such as `You've used 85% of your session limit · resets 3:45pm`.

Severity is three-valued (`allowed` / `allowed_warning` / `rejected`) on the push path. On the poll path, each row
carries a server string (`normal` / `warning` / `critical`, open-ended).

## Fields

### A. `SDKRateLimitEvent` / `SDKRateLimitInfo` (Agent SDK stream message, push)

Source: `sdk.d.ts:5630-5667` (0.3.289). The public docs (TS, section `SDKRateLimitEvent`) show only a subset: `status`,
`resetsAt`, `utilization`, `errorCode`, `canUserPurchaseCredits`, `hasChargeableSavedPaymentMethod`.

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `type` | `'rate_limit_event'` | req | discriminator | sdk.d.ts:5634 |
| `rate_limit_info` | `SDKRateLimitInfo` | req | "Rate limit information for claude.ai subscription users" | sdk.d.ts:5635-5638 |
| `uuid`, `session_id` | `UUID`, `string` | req | envelope | sdk.d.ts:5639-5640 |
| `status` | `'allowed' \| 'allowed_warning' \| 'rejected'` | req | overall state of the most relevant limit | sdk.d.ts:5647 |
| `resetsAt` | `number` | opt | reset time, **Unix epoch seconds** (parsed with `Math.round(Number(header))`) | sdk.d.ts:5648; bin@187244077 (`Lke`) |
| `rateLimitType` | `'five_hour' \| 'seven_day' \| 'seven_day_opus' \| 'seven_day_sonnet' \| 'seven_day_overage_included' \| 'overage'` | opt | which window the status/reset refer to. Comes from header `representative-claim`, or from the window whose reading triggered a warning | sdk.d.ts:5649; bin@187244077 |
| `utilization` | `number` | opt | window utilization. From the bundle it is a **fraction 0..1**, not a percentage: warning thresholds are `0.9`, `0.75`, etc., and the status line converts it with `Math.round(e*1000)/10` → percent (`ovt`, bin@180300529). The docs do not state the unit (UNVERIFIED in docs) | sdk.d.ts:5650; bin@187244077, bin@180300529 |
| `overageStatus` | `'allowed' \| 'allowed_warning' \| 'rejected'` | opt | state of the usage-credits/extra-usage bucket | sdk.d.ts:5652 |
| `overageResetsAt` | `number` | opt | epoch seconds when the overage period resets | sdk.d.ts:5653 |
| `overageDisabledReason` | enum of 13: `overage_not_provisioned`, `org_level_disabled`, `org_level_disabled_until`, `out_of_credits`, `seat_tier_level_disabled`, `member_level_disabled`, `seat_tier_zero_credit_limit`, `group_zero_credit_limit`, `member_zero_credit_limit`, `org_service_level_disabled`, `no_limits_configured`, `fetch_error`, `unknown` | opt | why overage can't be used. The bundle maps the raw value `org_spend_cap_reached` → `org_level_disabled_until` | sdk.d.ts:5654; bin@194694922 |
| `isUsingOverage` | `boolean` | opt | true when `status==='rejected'` and overage is allowed (requests are drawing on credits) | sdk.d.ts:5655; bin@187244077 |
| `overageInUse` | `boolean` | opt | from header `overage-in-use: true`. Omitted in some auth modes | sdk.d.ts:5656; bin@205837574 |
| `surpassedThreshold` | `number` | opt | server-declared threshold that the window crossed (from `*-surpassed-threshold`) | sdk.d.ts:5657 |
| `limitScope` | `'service' \| 'channel' \| 'group_pool'` | opt | which spend limit blocked the request. Absent on a plain member denial | sdk.d.ts:5661-5664 |
| `errorCode` | `'credits_required'` | opt | included usage exhausted, so the user must buy credits. Requires CLI ≥ 2.1.181 | sdk.d.ts:5665; TS |
| `canUserPurchaseCredits`, `hasChargeableSavedPaymentMethod` | `boolean` | opt | present only with `errorCode` | sdk.d.ts:5666-5667; TS |
| (runtime only) `rateLimitGraceActive`, `overagePeriodMonthly {utilization}`, `overagePeriodChannel {utilization}`, `unifiedWindows {five_hour?, seven_day?, seven_day_overage_included?: {utilization, resetsAt}}` | various | opt | emitted by the CLI's serializer but **not in the public `.d.ts`**. The blank lines at sdk.d.ts:5658-5660 suggest stripped internal fields. Treat as unstable/UNVERIFIED | bin@194694922 (`ka`), bin@205837574 (`is`, `Srt`) |

### B. Unified response headers (`anthropic-ratelimit-unified-*`), push per API response

These headers are **not publicly documented field by field**. APIRL lists none of them. GW names only the
`anthropic-ratelimit-unified-*` prefix and says the gateway adds them to each successful `/v1/messages` response for a
capped developer. The names below come from strings and parsing code in the CLI bundle (bin@187244077 `Lke`/`ivn`,
bin@186164275, bin@181743160). All values are strings.

| header | parsed as | meaning (from parser) |
|---|---|---|
| `status` | `allowed` \| `allowed_warning` \| `rejected` (default `allowed`) | overall state. The CLI turns header `allowed_warning` into `allowed`, then **re-derives** a warning from the per-window readings (see Windows) |
| `reset` | epoch seconds | reset of the representative claim |
| `representative-claim` | window id (`five_hour`, `seven_day`, `seven_day_opus`, `seven_day_sonnet`, ...) | which limit is binding → `rateLimitType` |
| `{5h,7d,7d_oi,overage}-utilization` | number (fraction) | per-window utilization |
| `{5h,7d,7d_oi,overage}-reset` | epoch seconds | per-window reset |
| `{5h,7d,7d_oi,overage}-surpassed-threshold` | number | server-side warning threshold crossed |
| `fallback` | `available` | model fallback is available |
| `overage-status`, `overage-reset`, `overage-disabled-reason`, `overage-in-use`, `overage-scope` | as in table A | overage bucket |
| `overage-period-monthly-utilization`, `overage-period-channel-utilization` | number | overage spend utilization |
| `upgrade-paths` | comma-separated list | upsell options |
| `grace-5h-utilization`, `grace-7d-utilization` | number | grace window (UNVERIFIED semantics) |
| `slow-*` (`status`, `offer`, `retry-after`, `max-wait`, `budget-utilization`, `budget-reset`) | various | "lower priority" continuation mode (UNVERIFIED, likely experiment-gated) |

In the bundle, window key `7d_oi` maps to `seven_day_overage_included` (bin@187244077, `Dw` table).

### C. Standard API rate-limit headers (API-key users), from APIRL "Response headers"

| header | type | meaning |
|---|---|---|
| `retry-after` | seconds | wait before retry. Absent on the spend-cap 429 |
| `anthropic-ratelimit-requests-{limit,remaining,reset}` | int / int / RFC 3339 | RPM bucket |
| `anthropic-ratelimit-tokens-{limit,remaining,reset}` | int / int (rounded to 1000) / RFC 3339 | most restrictive token limit currently in effect |
| `anthropic-ratelimit-input-tokens-{limit,remaining,reset}` | same | ITPM |
| `anthropic-ratelimit-output-tokens-{limit,remaining,reset}` | same | OTPM |
| `anthropic-priority-{input,output}-tokens-{limit,remaining,reset}` | same | Priority Tier only |

Other documented behavior: a spend-cap 429 has `error.type = rate_limit_error` with
`error.details.error_code = "enforced_spend_limit_reached"` and no `retry-after` (APIRL). Fast mode adds
`anthropic-fast-*` headers (APIRL).

I found no Claude Code surface (SDK type, status line, `/usage`) that exposes these limit/remaining/reset values. The
bundle contains only the bare prefix string `anthropic-ratelimit-` (5 occurrences), whose use I did not trace
(UNVERIFIED). For API-key sessions, the SDK sees rate limits only through these:

- `SDKAssistantMessageError` value `'rate_limit'`, meaning "a 429 against your quota", as distinct from
  `'overloaded'`/529 (sdk.d.ts:3693; TS).
- `SDKAPIRetryMessage {subtype:'api_retry', attempt, max_retries, retry_delay_ms, error_status, error}`
  (sdk.d.ts:3589-3605).

### D. `/usage` structured data, poll (SDK `get_usage` control request)

The request is `{subtype:'get_usage', skip_behaviors?}` (sdk.d.ts:4240-4246). The SDK method is
`usage_EXPERIMENTAL_MAY_CHANGE_DO_NOT_RELY_ON_THIS_API_YET({skipBehaviors?})` (sdk.d.ts:3039-3058). Its response is
`SDKControlGetUsageResponse` (sdk.d.ts:4251-4345).

| name | type | optional? | meaning |
|---|---|---|---|
| `session.{total_cost_usd,total_api_duration_ms,total_duration_ms,total_lines_added,total_lines_removed,model_usage}` | numbers, `Record<string, ModelUsage>` | req | local session totals |
| `subscription_type` | `string \| null` | req | `'pro' \| 'max' \| 'team' \| 'enterprise'`, or null for API key/3P |
| `rate_limits_available` | `boolean` | req | false for API key, Bedrock, Vertex, or a missing profile scope |
| `rate_limits` | object \| `null` | req (nullable) | null when unavailable |
| `rate_limits.{five_hour,seven_day,seven_day_oauth_apps,seven_day_opus,seven_day_sonnet}` | `{utilization: number\|null /*0-100*/, resets_at: string\|null /*ISO 8601*/} \| null` | opt + nullable | fixed per-window buckets. **The unit here is percent 0-100 and time is ISO 8601**, unlike the push path (fraction, epoch seconds) |
| `rate_limits.model_scoped[]` | `{display_name, utilization: number\|null, resets_at: string\|null}[]` | opt | per-model weekly windows with a server-supplied label. Absent = unknown; `[]` = server listed none |
| `rate_limits.extra_usage` | `{is_enabled, monthly_limit: number\|null, used_credits: number\|null, utilization: number\|null, currency?: string\|null} \| null` | opt | overage spend. Amounts are in currency minor units (stated on `SDKUsageReport`, sdk.d.ts:6209) |
| `behaviors` | object \| null | req (nullable) | local-transcript attribution over 24h / 7d. Null for non-subscribers |

The `SDKUsageReport` variant (sdk.d.ts:6152-6222) is attached as `usage_report` on the synthetic assistant message that
answers a `/usage` prompt (sdk.d.ts:3671). It carries the server's generic rows, which are the most flexible shape here:

| name | type | meaning |
|---|---|---|
| `rate_limits.limits[]` | array \| null | server rows "as sent", so a new meter needs no client release. Empty = no meters; null = server predates rows or the fetch failed |
| `.kind` | string | e.g. `session`, `weekly_all`, `weekly_scoped`. "Classify a row on this, never on a label" |
| `.group` | string | e.g. `session`, `weekly`. Rows render grouped, in server order |
| `.percent` | number | 0-100 |
| `.resets_at` | string \| null | ISO 8601 |
| `.scope` | `{model?: {display_name}\|null, surface?: {display_name}\|null} \| null` | opt; a scoped row targets a model or a surface |
| `.severity` | string | e.g. `normal`, `warning`, `critical`. Server-graded; "a client never grades a row itself" |
| `.is_active` | boolean | the row a single-value indicator should show |
| `rate_limits.extra_usage` | as in D | overage spend |

The backing endpoint is not in public docs: `/api/oauth/usage` on the claude.ai host, seen in bundle strings
(bin@186535524).

### E. Status line JSON input (`rate_limits`), push on render

Source: SL "Available data" and "Rate limit usage". Built from header readings (bin@188453333).

| name | type | optional? | meaning |
|---|---|---|---|
| `rate_limits.five_hour.used_percentage` | number 0-100 | opt | 5-hour window |
| `rate_limits.five_hour.resets_at` | epoch seconds | opt | |
| `rate_limits.seven_day.{used_percentage,resets_at}` | same | opt | 7-day window |
| `rate_limits.spend_limit.{used_percentage,resets_at}` | number (can exceed 100), epoch s | opt | gateway spend cap only. Requires ≥ 2.1.251 |
| `rate_limits.spend_limit.{used_usd,limit_usd,period}` | number, number, `daily\|weekly\|monthly` | opt | gateway only, ≥ 2.1.284. Fetched about every 5 min, so it can lag `used_percentage` |

Per SL, `rate_limits` appears "only for claude.ai Pro and Max subscribers, or behind a Claude apps gateway that sets a
spend limit", and only after the session's first API response. Each window may be independently absent, and a window
is dropped once its `resets_at` passes. Per-model and overage windows are **not** included.

## Delivery mechanism

| surface | push/poll | cadence | who gets it |
|---|---|---|---|
| unified headers | push, piggybacked on every `/v1/messages` response | per API request | subscription OAuth, gateway with spend cap |
| `rate_limit_event` (SDK stream, `--output-format stream-json`) | push | Emitted from the CLI's `statusChanged` subscription, i.e. when the derived `SDKRateLimitInfo` changes. The CLI deep-compares before emitting. A separate `quotaRejected` path re-emits a rejection, throttled to once per 30 s per `resetsAt` (bin@205837574 `ndo`, bin@206166708 `Xh`). Doc wording differs: TS says "emitted when the session encounters a rate limit"; sdk.d.ts:5631 says "when rate limit info changes" | subscription only (the type doc says "for claude.ai subscription users") |
| status line stdin JSON | push to the script | re-run on new assistant message, `/compact`, permission/vim change, optional `refreshInterval`, and when a `rate_limits.*.resets_at` passes; 300 ms debounce (SL "When it updates") | Pro/Max, gateway |
| `/usage` TUI (aliases `/cost`, `/stats`) | poll, user-initiated | Fetches on open; `r` retries. On failure it shows a last-known snapshot ≤ 60 min old (COSTS "When the usage request fails"). The bundle caches snapshots, writing at most once per 60 s, with a max age of 1 h (bin@187244077 `UPo`/`eGn`, `MXe=60000`, `bvn=3600000`) | session block for all; plan bars for Pro/Max/Team/Enterprise |
| SDK `get_usage` / `usage_report` | poll (request/response) / on `/usage` prompt | on demand | `rate_limits` null for API key/3P |
| TUI warnings and errors | push (text) | on status change | text such as `You've used N% of your session limit · resets …` (ERR), `You've hit your {session\|weekly\|Opus\|Sonnet} limit · resets …` (ERR), `Usage limit reached · limit resets 3:45pm` with auto-continue (IM), and gateway warnings at 75% and 95% (GW). Prefix lists are exported as `USAGE_LIMIT_ERROR_PREFIXES` / `USAGE_TRANSITION_PREFIXES` / `USAGE_WARNING_PREFIXES` (`@alpha`, sdk.d.ts:9805-9829) |
| `/status` | — | — | Shows version, model, account, connectivity (CMDS). No quota data is documented there |
| `/rate-limit-options` | — | shown on a limit hit | options menu: wait, credits, upgrade (CMDS) |

## Windows and semantics

- **Window ids on the push path:** `five_hour`, `seven_day`, `seven_day_opus`, `seven_day_sonnet`,
  `seven_day_overage_included`, `overage` (sdk.d.ts:5649).
- **Window ids on the poll path:** `five_hour`, `seven_day`, `seven_day_oauth_apps`, `seven_day_opus`,
  `seven_day_sonnet`, `model_scoped[]` (open-ended, server-labelled), `extra_usage`, plus the generic `limits[]` rows
  keyed by `kind`/`group`/`scope` (sdk.d.ts:4274-4345, 6152-6222).
- **Shared vs model-scoped windows:** the 5h "session" and 7d "weekly" limits are shared across models. The Opus and
  Sonnet limits apply only to that model family, so switching models helps only for those (ERR). Usage counts against
  the session and weekly windows at the same time (ERR).
- **Window lengths:** the bundle hard-codes 5h = 18000 s and 7d = 604800 s (bin@187244077 `tvn`; bin@181743160).
- **Warning derivation.** When the header status is `allowed`, the CLI raises `allowed_warning` under either of two
  conditions (bin@187244077 `mvn`/`hvn`/`tvn`):
  - a window carries `surpassed-threshold`;
  - the CLI's own **pace-based** thresholds are crossed:
    - 5h: utilization ≥ 0.9 while ≤ 72% of the window has elapsed;
    - 7d: ≥ 0.75 at ≤ 60%, ≥ 0.5 at ≤ 35%, or ≥ 0.25 at ≤ 15% elapsed.

  So a warning means "ahead of pace", not just "near 100%". This logic is internal and may change.
- **Overage:** when `status==='rejected'` but `overageStatus` is `allowed`/`allowed_warning`, requests continue on usage
  credits (`isUsingOverage: true`). Overage has its own status, reset, and disabled reason, plus monthly/channel
  utilization.
- **Units are inconsistent across surfaces:**

  | surface | utilization | reset time |
  |---|---|---|
  | push (`SDKRateLimitInfo`, headers) | fraction 0..1 | epoch s |
  | status line | percent | epoch s |
  | `get_usage` / `usage_report` | percent 0-100 | ISO 8601 string |

- **API-key users:** none of the subscription windows apply (`rate_limits_available:false`, `rate_limits:null`, no
  status line `rate_limits`, no `rate_limit_event` per the type doc). Their limits are per-minute RPM/ITPM/OTPM token
  buckets, visible only as 429 plus `retry-after`.
- **Relation to token usage and context window.** Plan utilization is computed by the server and has no stated
  relationship to the token counts Claude Code reports. The token and context figures are separate data:
  - per-model `ModelUsage {inputTokens, outputTokens, cacheRead/CreationInputTokens, costUSD, contextWindow,
    maxOutputTokens}` (sdk.d.ts:1432-1458);
  - `SDKResultSuccess.usage` / `modelUsage` / `total_cost_usd` (sdk.d.ts:5781-5791);
  - `getContextUsage()` (sdk.d.ts:3036);
  - status line `context_window.*`, where `used_percentage` counts input tokens only (SL).

  The `/usage` "behaviors" section attributes plan usage from local transcripts, approximately and only for this
  machine (COSTS "Plan usage breakdown"). A context or auto-compact warning is explicitly "not a usage limit" (COSTS).

## Gaps and unverified items

- The unified headers have no public, field-level documentation. Names and parsing come from the CLI bundle only.
  `grace-*` and `slow-*` semantics are UNVERIFIED.
- The `SDKRateLimitInfo.utilization` unit (fraction) comes from the bundle, not from docs. UNVERIFIED as a stable
  contract.
- The runtime-only fields (`unifiedWindows`, `rateLimitGraceActive`, `overagePeriodMonthly/Channel`) are not in the
  public types. UNVERIFIED stability.
- Whether `rate_limit_event` is ever emitted for API-key, Bedrock, or Vertex sessions is UNVERIFIED. The type doc says it
  is subscription-only, and gateway sessions are excluded from the 30 s rejection re-emit (`ndo` returns false for the
  gateway).
- The public docs (TS) lag the `.d.ts`. The docs omit `rateLimitType`, the overage fields, `surpassedThreshold`, and
  `limitScope`.
- `/usage` and `get_usage` are marked EXPERIMENTAL and may change (sdk.d.ts:3050-3053).
- No per-window *absolute* limit or *remaining count* exists on any subscription surface. The only absolute amounts are
  the overage `monthly_limit` / `used_credits` and the gateway `limit_usd` / `used_usd`.
- `/status` contents beyond the CMDS description were not inspected.

# GitHub Copilot CLI: quota and rate-limit reporting

Researched against `@github/copilot` 1.0.91 (npm) and `@github/copilot-sdk` 1.0.16 (npm, generated types in
`dist/generated/rpc.d.ts` and `dist/generated/session-events.d.ts`). Paths below prefixed `sdk:` refer to the unpacked
`@github/copilot-sdk` tarball.

## Summary

- Copilot CLI DOES have quota data, but it is exposed mainly through its own SDK JSON-RPC (not ACP).
  - Pull: `account.getQuota` returns per-bucket quota snapshots (entitlement, used, remaining percent, overage, reset
    date). Marked `@experimental`.
  - Push: `session.error` events carry `errorType: "rate_limit" | "quota"` and an `errorCode`;
    `auto_mode_switch.requested` carries `retryAfterSeconds`; `session_limits_exhausted.requested` reports a
    user-configured AI-credit cap.
  - Per-session usage: `session.usage.getMetrics` (premium request cost, nano-AIU, tokens).
- In ACP mode (`copilot --acp`) the public docs describe NO usage, quota or rate-limit surface. The only related thing is
  the `/usage` slash command, sent as a normal prompt; it returns plain text (format not documented). No on-demand
  query exists for it (https://docs.github.com/en/copilot/reference/copilot-cli-reference/acp-server).
- Interactive user surface: `/usage` shows session stats (AI Credits used this session, duration, lines edited, per-model
  token breakdown). It is NOT a monthly-quota or reset dashboard
  (https://docs.github.com/en/copilot/how-tos/use-copilot-agents/use-copilot-cli). Reports of older versions showing
  "Premium requests" and a remaining percentage are third-party/UNVERIFIED.
- Billing model is in transition: legacy premium-request quotas (Pro 300/mo, Pro+ 1500/mo, reset 1st of month 00:00 UTC)
  versus usage-based "AI Credits" (`token_based_billing`). Both appear in the types.

## Fields

### A. `account.getQuota` result (SDK JSON-RPC, pull) - `AccountGetQuotaResult`

Method string `account.getQuota` (sdk:`dist/generated/rpc.js:124`). Types at sdk:`dist/generated/rpc.d.ts:6317-6381`.
Request: `{ selectionId?: string; gitHubToken?: string }`. Result: `{ quotaSnapshots: { [quotaType: string]:
AccountQuotaSnapshot | undefined } }`. Keys are quota types, e.g. `chat`, `completions`, `premium_interactions`
(rpc.d.ts:6336).

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `isUnlimitedEntitlement` | boolean | required | user has unlimited entitlement | rpc.d.ts:6353 |
| `entitlementRequests` | number | required | requests included in entitlement; `-1` = unlimited | rpc.d.ts:6357 |
| `usedRequests` | number | required | requests used so far this period | rpc.d.ts:6361 |
| `usageAllowedWithExhaustedQuota` | boolean | required | usage still permitted after quota exhaustion | rpc.d.ts:6365 |
| `remainingPercentage` | number | required | percent of entitlement remaining | rpc.d.ts:6369 |
| `overage` | number | required | additional requests made this period beyond entitlement | rpc.d.ts:6373 |
| `overageAllowedWithExhaustedQuota` | boolean | required | additional (paid) usage allowed once exhausted | rpc.d.ts:6377 |
| `resetDate` | string (ISO 8601) | optional | when the quota resets | rpc.d.ts:6379-6381 |

### B. Raw user-response passthrough (`CopilotUserResponse`, rpc.d.ts:5780-5940, auth info `copilotUser`)

Raw snake_case Copilot API payload, all fields optional. `quota_snapshots` has keys `chat`, `completions`,
`premium_interactions` (rpc.d.ts:5946-5949), each with:

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `entitlement` | number | yes | units included; `-1` = unlimited | rpc.d.ts:5976 |
| `overage_count` | number | yes | pay-per-request usage beyond entitlement | rpc.d.ts:5980 |
| `overage_permitted` | boolean | yes | may continue at pay-per-request rates | rpc.d.ts:5984 |
| `percent_remaining` | number | yes | percent of entitlement remaining | rpc.d.ts:5988 |
| `quota_id` | string | yes | bucket identifier | rpc.d.ts:5992 |
| `quota_remaining`, `remaining` | number | yes | amount remaining (two near-duplicate fields) | rpc.d.ts:5996, 6000 |
| `unlimited` | boolean | yes | unlimited for this category | rpc.d.ts:6004 |
| `timestamp_utc` | string | yes | when snapshot was captured | rpc.d.ts:6008 |
| `has_quota` | boolean | yes | if false and not unlimited, requests blocked until reset | rpc.d.ts:6012 |
| `quota_reset_at` | number | yes | Unix epoch seconds of next reset | rpc.d.ts:6016 |
| `token_based_billing` | boolean | yes | category uses usage-based (AI credit) billing, not premium-request count | rpc.d.ts:6020 |

Top-level extras on `CopilotUserResponse`: `quota_reset_date` (raw string), `quota_reset_date_utc` (normalized),
`token_based_billing` (boolean), `limited_user_quotas` / `monthly_quotas` (`{[category]: number}` for free tier),
`limited_user_reset_date` (string), `can_upgrade_plan` (boolean), `access_type_sku` (SKU string)
(rpc.d.ts:5780-5900). `session.model.list` result also has optional `quotaSnapshots?: {[k]: JsonValue}` (rpc.d.ts:21789).

### C. Error events (push) - `session.error`, `ErrorData` (sdk:`dist/generated/session-events.d.ts:1610-1650`)

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `errorType` | string | required | category: `authentication`, `authorization`, `quota`, `rate_limit`, `context_limit`, `query` | session-events.d.ts:1620 |
| `errorCode` | string | optional | for `rate_limit`: `RateLimitErrorCode`, e.g. `user_weekly_rate_limited`, `user_global_rate_limited`, `rate_limited`, `user_model_rate_limited`, `integration_rate_limited`; for `quota`: `quota_exceeded`, `session_quota_exceeded`, `billing_not_configured` | session-events.d.ts:1616 |
| `message` | string | required | human-readable | session-events.d.ts:1624 |
| `statusCode` | number | optional | upstream HTTP status | session-events.d.ts:1643 |
| `eligibleForAutoSwitch` | boolean | optional | only on `rate_limit`; runtime will follow with `auto_mode_switch.requested` | session-events.d.ts:1612 |
| `providerCallId`, `serviceRequestId` | string | optional | `x-github-request-id`, `x-copilot-service-request-id` for correlation | session-events.d.ts:1628, 1634 |
| `remediation` | enum | optional | `sign_in`, `switch_account`, `show_account`, `review_sandbox_policy`, `allow_sandbox_outbound` (no quota-related action) | session-events.d.ts:64-74 |

`auto_mode_switch.requested` (session-events.d.ts:11065-11076): `{ requestId: string; errorCode?: string;
retryAfterSeconds?: number }`; `retryAfterSeconds` = "Seconds until the rate limit resets, when known". This is the
only structured reset/retry hint for rate limits.

### D. Session-level AI-credit cap (user-configured, not account quota)

`session_limits_exhausted.requested` (session-events.d.ts:11146-11165): `{ requestId: string; maxAiCredits: number;
usedAiCredits: number }`. Cap set via `--max-ai-credits N` on `/autopilot` and `/goal`
(https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference). `SessionLimitsChangedEvent`
carries `maxAiCredits?` (session-events.d.ts:1451).

### E. Per-session usage (not quota)

- `assistant.usage` event (session-events.d.ts:5795-5935, ephemeral): per-call `model`, `inputTokens?`, `outputTokens?`,
  `cacheReadTokens?`, `cacheWriteTokens?`, `reasoningTokens?`, `cost?` (model multiplier, experimental),
  `copilotUsage?: { model?; totalNanoAiu: number }`. Description text says "tokens, costs, quotas" but NO quota fields
  exist on the type in this version.
- `session.usage.getMetrics` (rpc.js:3046; result rpc.d.ts:26095-26140): `totalPremiumRequestCost: number` (fractional,
  multiplier-based), `totalUserRequests`, `totalNanoAiu?`, `tokenDetails?`, `modelMetrics`, `lastCallInputTokens`,
  `lastCallOutputTokens`, `codeChanges`, etc.
- `session.usage_info` event (session-events.d.ts:3032-3060): context-window occupancy `currentTokens` (req),
  `tokenLimit` (req), `messagesLength` (req), `conversationTokens?`, `systemTokens?`, `toolDefinitionsTokens?`,
  `isInitial?`. Separate from quota.

## Delivery mechanism

- Surfaces and who sees what:
  - Interactive TUI: `/usage` (local, no model call, no premium request charged), `/context`, autopilot credit panel.
  - Copilot SDK JSON-RPC (a separate protocol from ACP): pull via `account.getQuota`, `session.usage.getMetrics`;
    push via session events (`session.error`, `auto_mode_switch.requested`, `session_limits_exhausted.requested`,
    `assistant.usage`, `session.usage_info`).
  - ACP (`copilot --acp --stdio` or `--port`): public docs list only `initialize`, `session/new`, prompt,
    `session/update` (`agent_message_chunk`, `available_commands_update`), `requestPermission`. No custom/`_meta` usage
    fields are documented; `/usage` is invoked as a text prompt. I ran `initialize` against the 1.0.91 binary: response
    has `agentCapabilities` (`loadSession`, `mcpCapabilities`, `promptCapabilities`, `sessionCapabilities.{close,list}`),
    `agentInfo`, `authMethods`; nothing quota-related.
  - Server-side: Copilot API response header `X-GitHub-Copilot-Request-TE` is read (`rte` boolean) but is a treatment
    signal, not a quota; no `x-ratelimit-*` header handling was found in the typed surface.
- Cadence: quota is pull-only (no push event carries quota snapshots in the typed surface). Rate-limit and quota-exceeded
  errors are pushed at the time of failure. Usage events are per model call.
- Monthly reset for legacy premium requests: 1st of month 00:00:00 UTC, no rollover
  (https://docs.github.com/en/copilot/concepts/billing/copilot-requests).

## Windows and semantics

- Quota buckets by type (`chat`, `completions`, `premium_interactions`), each a single period (monthly) with a reset
  timestamp. There is no 5-hour window; a weekly rate-limit exists only as an error code (`user_weekly_rate_limited`)
  with no structured window/limit/used data. Per-model rate limit exists as `user_model_rate_limited` (code only).
- Distinction between "rate limit" (temporary, `errorType: "rate_limit"`, may have `retryAfterSeconds`) and "quota"
  (`errorType: "quota"`) is explicit.
- Overage is modeled as `overage`/`overage_count` plus a permission boolean; a plan with unlimited use is `-1`/`unlimited`.
- Unit is "requests" (premium requests, fractional via model multipliers) in legacy billing, and nano-AIU / AI Credits in
  usage-based billing. Quota snapshot fields are unit-ambiguous (`token_based_billing` flags which applies).
- Relation to tokens/context: separate. `session.usage_info` is context occupancy; `assistant.usage` and
  `getMetrics` give tokens and cost; quota snapshots are account-level.
- Public docs: rate-limit page gives no numbers, reset timing, or error wording
  (https://docs.github.com/en/copilot/concepts/rate-limits).

## Gaps and unverified items

- UNVERIFIED: whether ACP mode forwards any quota/usage data via `_meta`, `usage_update`, or notifications. Docs say no;
  I could not exercise an authenticated session. The 1.0.91 binary is a bundled/compressed executable, so grepping it
  yielded nothing.
- UNVERIFIED: the SDK `account.getQuota` is `@experimental`; whether the shipped CLI (`copilot`) exposes it only via the
  SDK server mode (not ACP) is inferred from the SDK client, not confirmed from CLI source (closed source; the
  github/copilot-cli repo holds only README/issues).
- UNVERIFIED: exact `/usage` text format; official docs list contents only. Third-party posts showing "234 / 300" and
  reset date conflict with GitHub's description.
- Known bug: `/usage` premium-request counter undercounts vs real quota and drops after compaction (CLI 0.0.413)
  (https://github.com/github/copilot-cli/issues/1582, open, no maintainer reply when fetched).
- The `RateLimitErrorCode` enum is referenced in a doc comment only; full list not seen. `retryAfterSeconds` is optional
  ("when known"), and only present on the auto-mode-switch prompt, not on every rate-limit error.
- Quota snapshot field presence is all-optional in the raw passthrough; the typed `AccountQuotaSnapshot` makes all but
  `resetDate` required.
- Not checked: `statusLine` config fields in the TUI, Copilot Language Server's own ACP mode (used by some IDE clients), and
  GitHub REST billing/usage APIs (the third-party `copilot-usage` tool uses them via `gh`).

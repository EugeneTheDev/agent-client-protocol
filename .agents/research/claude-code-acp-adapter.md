# Claude Code ACP adapter: rate limits, quota, and token usage over ACP

Repo: `github.com/zed-industries/claude-code-acp` now redirects to
`github.com/agentclientprotocol/claude-agent-acp`, published as `@agentclientprotocol/claude-agent-acp` v0.85.1
(`package.json`). It depends on `@anthropic-ai/claude-agent-sdk` 0.3.287.

All `file:line` references are at commit `3a61172e194c32656abcbe6d1c86fdca6a04f573` (2026-10-05). Permalink base:
`https://github.com/agentclientprotocol/claude-agent-acp/blob/3a61172e194c32656abcbe6d1c86fdca6a04f573/`.

ACP-side definitions are in this repo at `b459540a`: `schema/v1/schema.json` and `schema/v1/schema.unstable.json`.

## Summary

- **Rate-limit data takes one path.** On the SDK `rate_limit_event`, the adapter copies `message.rate_limit_info`
  verbatim, without changing it, into `_meta["_claude/rateLimit"]` of a `usage_update` session notification
  (`src/acp-agent.ts:6961-6973`). ACP has no rate-limit field, so the data rides as an extension inside the
  context-window notification.
  - That notification must carry the required `used`/`size`, and the adapter fills them with the current
    context-window occupancy.
  - If no occupancy reading exists yet in the current turn (`lastAssistantTotalUsage === null`), the adapter drops
    the event and the client never sees it.
- **No structured quota state.** The adapter keeps no rate-limit state and builds no quota object of its own. It does
  not map limits onto `PromptResponse` or `session_info_update`. The forwarded content (`status`, `resetsAt`,
  `utilization`, and anything else the SDK adds) is whatever the SDK emits.
- **`/usage` slash command.** The adapter calls the experimental SDK control request
  `query.usage_EXPERIMENTAL_MAY_CHANGE_DO_NOT_RELY_ON_THIS_API_YET()` and renders it as Markdown in an agent message
  (`src/acp-agent.ts:418-452`, `src/usage-markdown.ts:150-215`). The Markdown shows 5-hour, weekly, and per-model
  windows with utilization % and reset time. This is display only: the client gets no structured data.
- **Rate-limit or quota rejection becomes a turn failure, not quota data.**
  - The SDK's categorical `SDKAssistantMessage.error` (`rate_limit`, `billing_error`, ...) and the synthetic
    "usage limit" assistant text are mapped to failure kinds `rate_limited` / `quota_exhausted`
    (`src/session-failure-extension.ts:449-487`).
  - Legacy clients get a JSON-RPC `internalError` with `data: { errorKind }`.
  - Clients that negotiated a vendor-namespaced session-failure `_meta` extension get `stopReason: "end_turn"` plus a
    typed failure record in `PromptResponse._meta`.
  - SDK `api_retry` events (including 429 retries) become warning records for those clients only.
- **Token usage.** `PromptResponse.usage` (unstable ACP) is **per turn**: reset at turn activation and summed from
  `result.usage` of the main agent loop. `_meta.quota` on the prompt response repeats it in a codex-acp-compatible
  shape and adds per-model rows. The adapter never reports cumulative session tokens over ACP, except as text inside
  `/usage`. `usage_update.used/size` is context-window occupancy. `usage_update.cost` is `result.total_cost_usd`.

## Fields

### A. Rate-limit data forwarded on `usage_update`

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `sessionUpdate` | `"usage_update"` | required | ACP context/cost update, reused here as the rate-limit carrier | `src/acp-agent.ts:6966` |
| `used` | uint64 | required by ACP | Context occupancy: the last top-level assistant usage, summed as input+output+cache_read+cache_creation. It is **not** a rate-limit quantity. | `src/acp-agent.ts:6967`, `totalTokens` at `:10174-10181` |
| `size` | uint64 | required by ACP | Context-window size of the current model (`session.contextWindowSize`). It is **not** a rate-limit quantity. | `src/acp-agent.ts:6968`, field doc `:1007-1016` |
| `_meta["_claude/rateLimit"]` | object (SDK `rate_limit_info`, copied unchanged) | present only on this update | Rate-limit snapshot from the SDK | `src/acp-agent.ts:6969` |
| `_meta["_claude/rateLimit"].status` | string, e.g. `"allowed_warning"` | depends on the SDK | Severity or status | test fixture `src/tests/acp-scenarios/scenarios.ts:1144-1149` |
| `_meta["_claude/rateLimit"].resetsAt` | number (fixture `1700000000`, so epoch seconds) | depends on the SDK | Reset time | same fixture |
| `_meta["_claude/rateLimit"].utilization` | number (fixture `0.9`, so a 0..1 fraction) | depends on the SDK | Fraction of the window used | same fixture |
| other `rate_limit_info` keys (window type, overage, ...) | — | — | Forwarded unchanged if the SDK sends them. Not enumerated here. | UNVERIFIED (see the SDK-types note) |
| `_meta["_claude/model"]` | string | omitted until a model is seen in the turn | Model of the last top-level assistant message. Added to every adapter `usage_update`. | `src/acp-agent.ts:4105-4125` |

Wire example (snapshot `src/tests/acp-scenarios/origin-main/plain/rate-limit-and-origin.jsonl:6`):

```json
{"sessionUpdate":"usage_update","used":2,"size":200000,
 "_meta":{"_claude/model":"claude-sonnet-4-6",
          "_claude/rateLimit":{"resetsAt":1700000000,"status":"allowed_warning","utilization":0.9}}}
```

### B. `/usage` command (rendered as Markdown, not structured over ACP)

These are the input fields the adapter validates before rendering (`src/usage-markdown.ts:6-61`). The client receives
only the Markdown text.

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `rate_limits_available` | boolean | required | Whether to render the limits section | `src/usage-markdown.ts:31`, `:156` |
| `rate_limits.five_hour` / `.seven_day` / `.seven_day_oauth_apps` / `.seven_day_opus` / `.seven_day_sonnet` | `{utilization: number(0..100) \| null, resets_at: string \| null} \| null` | optional, nullable | Per-window utilization **in percent** and an ISO reset time | `src/usage-markdown.ts:6-9`, `:33-38` |
| `rate_limits.model_scoped[]` | `{display_name, utilization \| null, resets_at \| null}` | optional | Per-model weekly windows. When present they replace the Opus/Sonnet rows. | `src/usage-markdown.ts:39-47`, `:160-167` |
| `rate_limits.extra_usage` | `{is_enabled, monthly_limit \| null, used_credits \| null, utilization \| null, currency?}` | optional, nullable | Overage / extra-usage credits. Validated but **not rendered**. | `src/usage-markdown.ts:48-57` |
| `subscription_type` | string \| null | nullable | Plan label | `src/usage-markdown.ts:30`, `:152-154` |
| `session.total_cost_usd`, `total_api_duration_ms`, `total_duration_ms`, `model_usage{}` | numbers / record | required | Cumulative session cost, time, and per-model tokens (only place cumulative tokens are shown) | `src/usage-markdown.ts:24-29`, `:174-200` |
| `behaviors.day/week` | `{request_count, session_count, mcp_servers[]}` | nullable | "What's using your limits" breakdown | `src/usage-markdown.ts:60`, `:202-213` |

A window with `utilization === null` is skipped (`src/usage-markdown.ts:120`). If the request times out, fails, or
does not validate, the adapter falls back to Claude Code's own `/usage` text (`src/acp-agent.ts:432-451`).

### C. Token usage and cost

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `PromptResponse.usage.{inputTokens,outputTokens,cachedReadTokens,cachedWriteTokens,totalTokens}` | numbers | always set by the adapter. ACP marks it unstable and optional. | **Per-turn** sum of `result.usage` for the main loop. Autonomous results are excluded. `totalTokens` is the sum of the four counters. `thoughtTokens` is never set. | `sessionUsage` `src/acp-agent.ts:9988-9996`; accumulation `:5884-5895`; reset `resetTurnScratch` `:4163-4168` |
| `PromptResponse._meta.quota.token_count` | `{totalTokens,inputTokens,cachedInputTokens,cachedWriteTokens,outputTokens,reasoningOutputTokens:0}` | always | Same per-turn tally in codex-acp shape | `src/acp-agent.ts:10027-10069` (`turnQuotaMeta` at :10040, `quotaTokenCount` at :10058) |
| `PromptResponse._meta.quota.model_usage[]` | `{model, token_count}` | always (may be `[]`) | Per-model per-turn increment derived from `result.modelUsage`. That source is a running total for the `query()` call and counts subagents and compaction, so the rows may sum to more than `token_count`. | `src/acp-agent.ts:5897-5918`, `:10101-10123` (`modelUsageIncrement`) |
| `usage_update.used` / `size` | uint64 | required | Context occupancy and window size | see section D |
| `usage_update.cost.{amount,currency:"USD"}` | number/string | only on the `usage_update` emitted at a `result` | `result.total_cost_usd` | `src/acp-agent.ts:5965-5981` |
| `_meta["_claude/origin"]` on the result `usage_update` | SDK origin object | when the result has `origin` | Origin of the result, for example `{kind:"human"}` or autonomous | `src/acp-agent.ts:5977-5979` |

### D. Errors and failures on rate-limit or quota rejection

| Signal | Shape over ACP | Source |
|---|---|---|
| SDK `SDKAssistantMessage.error` captured before `result` (`rate_limit`, `billing_error`, `account_on_hold`, `overloaded`, ...) | Legacy clients: the `session/prompt` call is rejected with `RequestError.internalError(data: { errorKind: <SDK error kind> }, message)`. JSON-RPC code is internal error. | `src/acp-agent.ts:3841-3847`, `:6224-6231`, `:6286-6295`, `errorKindData` `:10198-10202`, `failActive` `:4614-4634` |
| Synthetic usage-limit assistant message (`model === "<synthetic>"` and text starts with an SDK `USAGE_LIMIT_ERROR_PREFIXES` entry) | Classified as `quota_exhausted`. Legacy clients also get the limit text as a normal `agent_message_chunk`. Negotiated clients do not get the chunk. | `src/session-failure-extension.ts:205-212`, `src/acp-agent.ts:6695-6698`, `:6810-6821` |
| Mapping: `rate_limit` becomes `rate_limited` (category `limit`, action `retry`). `billing_error`, `account_on_hold`, and usage limit become `quota_exhausted` (category `limit`, no action). `error_max_budget_usd` becomes `budget_exhausted`. | Only for clients that negotiated the vendor session-failure capability: `PromptResponse{stopReason:"end_turn", _meta:{<vendor ns>.sessionFailure:{id,revision,category,severity,title,details?,reason?,actions}}}`. The record carries no reset time and no utilization; only `title` holds any detail. | `src/session-failure-extension.ts:89-166`, `:168-179`, `:449-487`; `src/acp-agent.ts:4645-4681`; vendor extension docs in the adapter repo |
| SDK `system/api_retry` (retry after a 429 or 5xx) | Same vendor extension, `severity:"warning"`, title `Retrying Claude, attempt n of max.`, sent in a `session_info_update`. Nothing reaches legacy clients. | `src/acp-agent.ts:5568-5597` |
| Fast-mode `cooldown` (suspended after a rate limit) | Not surfaced. The Fast toggle stays as it was. | `src/acp-agent.ts:9030-9032`, `:10374-10380` |

## Delivery mechanism

- **Rate limit: push.**
  - The adapter emits one `session/update` with `usage_update` per SDK `rate_limit_event`, while a turn's consumer
    loop runs.
  - It adds no throttling and no deduplication. The SDK decides the cadence (UNVERIFIED how often it emits).
  - Guard: the update is sent only if `lastAssistantTotalUsage !== null` (`src/acp-agent.ts:6962`).
  - `lastAssistantTotalUsage` is cleared at every turn activation (`:4147`). It is set again only by:
    - a top-level `stream_event` `message_start`/`message_delta` (`:6461-6519`),
    - a non-synthetic top-level `assistant` message (`:6689-6694`),
    - a `compact_boundary` (`:5093-5094`).

  So an event that arrives before the turn's first model response is **silently dropped**. That covers a request
  rejected outright, where only a synthetic frame exists. The client also never receives a "limit cleared" update.
- **Raw passthrough (opt-in).** `_meta.claudeCode.emitRawSDKMessages` on session creation (true, or filters by
  `type`/`subtype`/`origin`) makes the adapter also send every matching SDK message, `rate_limit_event` included, as
  `extNotification("_claude/sdkMessage", {sessionId, message})` (`src/acp-agent.ts:1458`, `:1492-1499`,
  `:4879-4886`). This path has no null-usage guard.
- **`/usage`: pull.** It runs only when the prompt is exactly `/usage` (`src/usage-markdown.ts:72-74`,
  `src/acp-agent.ts:3287-3299`). The result arrives as the turn's agent message, as Markdown.
- **Token usage.**
  - Pushed as `usage_update` on each change of the top-level streaming usage (`message_start`/`message_delta`), on
    each consolidated assistant message, on `compact_boundary`, and on each `result`, which also carries `cost`.
  - Returned per turn on `PromptResponse.usage` and `_meta.quota`.
  - `usage_update` passes unchanged through the ACP v2 surface (`src/v2/session-update.ts:32-37`).
- **Failures.** Returned on the prompt response, or as a JSON-RPC error. Warnings and session-scoped failures go
  out as `session_info_update` (vendor extension only).

## Windows and semantics

- **Windows.**
  - On the push path, the adapter does not interpret windows. Any window type, overage, or per-model indicator the SDK
    puts in `rate_limit_info` reaches the client unchanged, and only one snapshot travels per event.
  - On the `/usage` path, the adapter renders 5-hour, weekly (all models), and weekly per-model windows (from
    `model_scoped`, or else Opus/Sonnet). It does not render `seven_day_oauth_apps` or `extra_usage`.
- **Units differ between the two paths** (fixture evidence only):
  - `rate_limit_info.utilization` is `0.9`, a fraction, and `resetsAt` is a number, apparently epoch seconds.
  - `/usage` `utilization` is 0..100 percent (`percentSchema = max(100)`) and `resets_at` is an ISO string.
  - Both are UNVERIFIED against SDK types. The other researcher covers them.
- **Relation to per-turn token usage.** None. Rate-limit data is not derived from or combined with
  `PromptResponse.usage` or `_meta.quota`. The prompt response carries no rate-limit or reset information. The name
  `_meta.quota` misleads: it holds only per-turn **token counts**, copied from codex-acp's shape
  (`src/acp-agent.ts:10027-10050`), not quota or limit state.
- **Relation to cumulative session tokens.**
  - The adapter never sends a cumulative session token total as a field.
  - `sessionUsage()` has a misleading name: it returns the per-turn accumulator.
  - The ACP unstable `Usage` schema conflicts with itself. Its field descriptions say "across session" / "across all
    turns", while `PromptResponse.usage` says "Token usage for this turn" (`schema/v1/schema.unstable.json`). The
    adapter follows the per-turn reading.
  - `usage_update.cost.amount` is `result.total_cost_usd`. ACP defines it as cumulative session cost. Whether the SDK
    value is cumulative for the `query()` process or for the whole session, including after resume, is UNVERIFIED.
- **Relation to context-window occupancy.** This is where they are **conflated**. The rate-limit snapshot is
  delivered inside the context-occupancy notification:
  1. A client that reads every `usage_update` as a context update gets an extra, repeated `used/size` reading on each
     rate-limit event. The value can lag the latest streamed reading: the fixture shows `used: 3` mid-stream, then
     `used: 2` on the rate-limit update taken from the consolidated message.
  2. The rate-limit signal depends on occupancy being known, which causes the drop described above.
  3. ACP gives no way to send a rate-limit update without also restating context state, or to clear a previous one.

  `used` is a proxy for occupancy: the last top-level request's input+output+cache tokens (`:10170-10181`).
  Subagent usage is excluded (`parent_tool_use_id === null` guards at `:6461`, `:6689`). The rate-limit event is not
  filtered by subagent.

## Gaps and unverified items

- UNVERIFIED: the full key set and types of SDK `rate_limit_info`, for example a window-type discriminator, overage
  fields, or a "rejected" status value. The adapter forwards whatever it receives. Only `status`, `resetsAt`, and
  `utilization` appear in its tests.
- UNVERIFIED: how often the SDK emits `rate_limit_event`, and whether it emits one on hard rejection before any
  assistant frame. If it does, the adapter drops it, per the guard at `src/acp-agent.ts:6962`.
- UNVERIFIED: the exact strings in `USAGE_LIMIT_ERROR_PREFIXES` (an SDK export).
- UNVERIFIED: whether `result.total_cost_usd` is cumulative for the session or for the process.
- Missing over ACP today:
  - structured reset time and utilization on failures (the failure record has only text),
  - structured `/usage` data (Markdown only),
  - `extra_usage` / overage (validated, never shown),
  - a cleared or recovered rate-limit signal,
  - any rate-limit information for legacy clients beyond `errorKind` and message text.
- The session-failure extension uses a vendor-specific `_meta` namespace defined in a vendor extension module and
  documented in the vendor extension docs in the adapter repo. Its exact key is left out of this note on purpose.

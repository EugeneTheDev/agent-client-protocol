# Codex ACP adapter: rate limits, quota, and token usage forwarding

Scope: what the Codex ACP adapter emits over ACP today. Codex-native type semantics are only summarized where needed;
another note covers them in depth.

## Repos and pinned commits

- **Current adapter:** `github.com/agentclientprotocol/codex-acp` (TypeScript, built on the Codex App Server JSON-RPC
  protocol), npm `@agentclientprotocol/codex-acp` 2.1.1, commit `ca1d97173ad37b471d5a4e5847725a4657d34e29`
  (2026-10-02). It depends on `@agentclientprotocol/sdk ^1.5.0` and `@openai/codex ^0.159.1` (`package.json:69-70`).
  All `src/...` and `docs/...` references below are at this commit unless marked otherwise.
- **Legacy adapter:** `github.com/zed-industries/codex-acp` (Rust, embedded Codex core), archived. Its README says
  development has moved to the repo above (README.md:3-7 at `296069e841634cd4bb9bc4515602d836e49231ec`). It is covered
  briefly at the end, for history only.

## Summary

- **Codex inputs:** the adapter consumes three app-server sources.
  1. The `thread/tokenUsage/updated` notification, which carries `{total, last, modelContextWindow}`. The adapter maps it
     to session state and to an ACP `usage_update`.
  2. The `account/rateLimits/updated` notification, a sparse rolling snapshot. It is merged into in-memory state only
     and is never forwarded on the wire.
  3. The `account/rateLimits/read` request. It is polled only when the user runs `/status`.
- **ACP outputs:**
  - `session/update` → `usage_update {used, size}` (context occupancy). `used` comes from `last.totalTokens`.
  - `PromptResponse.usage` (ACP `Usage`). It also comes from `last`, the most recent model request, and not from a sum
    over the turn.
  - `PromptResponse._meta.quota = {token_count, model_usage[]}`. It holds the same `last` numbers in an older snake_case
    wrapper. **It contains no rate-limit data**, even though the adapter's docs say it does.
  - `/status` slash-command output. This is a markdown `agent_message_chunk` with cumulative session tokens, context
    percentage, and every rate-limit window (percent left plus reset time), credits, and the individual spend limit.
    **This is the only place rate limits reach the client.**
  - Rate-limit and usage-limit *errors*:
    - For clients that opt into a vendor-namespaced, capability-gated `_meta` failure record, the adapter emits a typed
      failure with category `limit`.
    - For other clients, it emits plain text, and for `usageLimitExceeded` only, a JSON-RPC internal error with
      `codexErrorInfo` in `data`.
- **Conflation:**
  - Per-turn usage, per-request usage, and context occupancy all come from the same `last` breakdown.
  - `_meta.quota.token_count` is commented as "aggregated" but holds `last` too.
  - `model_usage` attributes everything to the current model, which a FIXME in the code acknowledges.
  - Cumulative session tokens (`total`) are kept, but they are only shown as `/status` text.

## Fields

### A. Codex inputs consumed (shape only, as vendored in the adapter's generated types)

| Name | Type | Optional? | Meaning | Source |
| --- | --- | --- | --- | --- |
| `ThreadTokenUsageUpdatedNotification` | `{threadId: string, turnId: string, tokenUsage: ThreadTokenUsage}` | — | App-server push, method `thread/tokenUsage/updated` | `src/app-server/v2/ThreadTokenUsageUpdatedNotification.ts`; consumed at `src/CodexEventHandler.ts:529-530` |
| `ThreadTokenUsage.total` | `TokenUsageBreakdown` | required | Cumulative thread usage | `src/app-server/v2/ThreadTokenUsage.ts` |
| `ThreadTokenUsage.last` | `TokenUsageBreakdown` | required | Usage of the latest model request. The adapter's tests treat it as a "context proxy" (see Windows and semantics) | same; `src/__tests__/CodexACPAgent/token-usage-events.test.ts:197` |
| `ThreadTokenUsage.modelContextWindow` | `number \| null` | nullable | Context window size | same |
| `TokenUsageBreakdown` | `{totalTokens, inputTokens, cachedInputTokens, cacheWriteInputTokens, outputTokens, reasoningOutputTokens}` (all `number`) | required | Codex counts `inputTokens` *including* cached tokens | `src/app-server/v2/TokenUsageBreakdown.ts`; `src/TokenCount.ts:25` |
| `AccountRateLimitsUpdatedNotification` | `{rateLimits: RateLimitSnapshot}` | — | Sparse rolling update, method `account/rateLimits/updated`. "Nullable account metadata ... does not clear a previously observed value." | `src/app-server/v2/AccountRateLimitsUpdatedNotification.ts`; consumed at `src/CodexEventHandler.ts:568-570` |
| `GetAccountRateLimitsResponse` | `{ordinaryUsageAllowed: boolean\|null, rateLimits: RateLimitSnapshot, rateLimitsByLimitId: {[id]: RateLimitSnapshot}\|null, rateLimitResetCredits, accountId, rateLimitUpsell}` | mixed | Response to `account/rateLimits/read` | `src/app-server/v2/GetAccountRateLimitsResponse.ts`; called at `src/CodexAppServerClient.ts:784-785`, `src/CodexAcpClient.ts:540-541` |
| `RateLimitSnapshot` | `{limitId, limitName, normalModelSlug, primary, secondary, credits, individualLimit, spendControlReached, planType, rateLimitReachedType}` (all nullable) | nullable | One metered bucket | `src/app-server/v2/RateLimitSnapshot.ts` |
| `RateLimitWindow` | `{usedPercent: number, windowDurationMins: number\|null, resetsAt: number\|null}` | — | `resetsAt` is in epoch seconds (the adapter multiplies it by 1000) | `src/app-server/v2/RateLimitWindow.ts`; `src/CodexCommands.ts:608` |
| `CreditsSnapshot` | `{hasCredits: boolean, unlimited: boolean, balance: string\|null}` | — | Credit balance | `src/app-server/v2/CreditsSnapshot.ts` |
| `SpendControlLimitSnapshot` | `{limit: string, used: string, remainingPercent: number, resetsAt: number}` | — | Individual spend limit (decimal strings) | `src/app-server/v2/SpendControlLimitSnapshot.ts` |
| `RateLimitReachedType` | `"rate_limit_reached" \| "workspace_owner_credits_depleted" \| "workspace_member_credits_depleted" \| "workspace_owner_usage_limit_reached" \| "workspace_member_usage_limit_reached"` | nullable | Which limit was hit. **Not used by the adapter** (no reads outside generated types) | `src/app-server/v2/RateLimitReachedType.ts` |
| `ErrorNotification.error.codexErrorInfo` | `CodexErrorInfo` (string enum or `{kind: {httpStatusCode?...}}`) | nullable | Includes `usageLimitExceeded`, `rateLimitExceeded`, `contextWindowExceeded`, `sessionBudgetExceeded` | `src/CodexEventHandler.ts:157-173`, `1106-1131` |

### B. Adapter session state (in memory, per session)

| Name | Type | Optional? | Meaning | Source |
| --- | --- | --- | --- | --- |
| `lastTokenUsage` | `TokenCount \| null` | null until the first usage event of the turn | From `tokenUsage.last`. Reset to `null` at every prompt or command turn start | `src/CodexAcpServer.ts:187`, `3063`, `3169`; `src/CodexEventHandler.ts:1155` |
| `totalTokenUsage` | `TokenCount \| null` | null until the first event | From `tokenUsage.total`. Never reset per turn | `src/CodexAcpServer.ts:188`; `src/CodexEventHandler.ts:1156` |
| `modelContextWindow` | `number \| null` | nullable | From `tokenUsage.modelContextWindow` | `src/CodexAcpServer.ts:189`; `src/CodexEventHandler.ts:1157` |
| `rateLimits` | `Map<limitId, {limitId, limitName, snapshot: RateLimitSnapshot}> \| null` | null until a push or `/status` | Keyed by `limitId`, defaulting to `"codex"`. A push merges into the existing entry, keeping `credits`, `individualLimit`, `spendControlReached`, and `planType` when the update omits them. `/status` replaces the whole map | `src/CodexAcpServer.ts:190`; `src/RateLimitsMap.ts:3-47`; `src/CodexEventHandler.ts:1176-1190`; `src/CodexCommands.ts:468-477` |
| `TokenCount` (adapter type) | `{totalTokens, inputTokens, cachedInputTokens, outputTokens, reasoningOutputTokens}` | — | `inputTokens = codex.inputTokens - cachedInputTokens` (non-cached input only). **`cacheWriteInputTokens` is dropped**. `totalTokens` is passed through unchanged | `src/TokenCount.ts:14-36` |

Session creation and load both initialize all four fields to `null` (`src/CodexAcpServer.ts:796-799` in
`tryCreateSession`, `2222-2225` in `getOrCreateSessionWithHistory`). No `usage_update` is replayed on load or resume
(no emit site outside the event handler; `grep usage_update src/` finds only `CodexEventHandler.ts:1170`).

### C. ACP outputs

| Name | Type | Optional? | Meaning | Source |
| --- | --- | --- | --- | --- |
| `session/update` `usage_update.used` | `number` (uint64 in ACP) | sent only if `used != null && size != null && size > 0` | `lastTokenUsage.totalTokens`, which is `last.totalTokens` from Codex | `src/CodexEventHandler.ts:1160-1174` |
| `usage_update.size` | `number` | same condition | `modelContextWindow` | same |
| `usage_update.cost` | — | **never sent** | — | absent in `src/CodexEventHandler.ts:1169-1173` |
| `PromptResponse.usage` | ACP `Usage \| null` | `null` if no usage event arrived during the turn | `{totalTokens, inputTokens (non-cached), cachedReadTokens, outputTokens, thoughtTokens}` mapped from `lastTokenUsage`. **`cachedWriteTokens` is never set** | `src/TokenCount.ts:43-51`; `src/CodexAcpServer.ts:3527-3532` |
| `PromptResponse._meta.quota.token_count` | adapter `TokenCount \| null` (camelCase keys inside a snake_case key) | nullable | Same values as `usage`, in the adapter's field names (`cachedInputTokens`, `reasoningOutputTokens`). The source comment says "Aggregated token count across all models", but the value is `lastTokenUsage` | `src/QuotaMeta.ts:8-14`; `src/CodexAcpServer.ts:3508-3525` |
| `PromptResponse._meta.quota.model_usage` | `Array<{model: string, token_count: TokenCount}>` | `[]` when there is no usage | At most one entry: `currentModelId` with any `[effort]` suffix stripped. The code says "FIXME: currently all tokens are reported for the current model" | `src/QuotaMeta.ts:3-6`; `src/CodexAcpServer.ts:3511-3517` |
| `usage` and `_meta.quota` carriers | — | — | Present on every PromptResponse path: normal `end_turn` (`3376-3380`), command-handled turn (`3134-3138`), `cancelled` (`3480-3486`), and terminal-failure `end_turn` (`3488-3506`, where `quota` is merged with the failure record) | `src/CodexAcpServer.ts` |
| `/status` output | markdown text in `agent_message_chunk` | — | Lines include `**Token usage:** <total> total (<input> input + <cached> cached input, <output> output)` from `totalTokenUsage`; `**Context window:** <pct>% used (<used> used / <size>)` from `lastTokenUsage.totalTokens / modelContextWindow`; and per limit bucket `**<limitName> <5h\|7d\|Weekly\|Nm> limit:** <100-usedPercent>% left (resets HH:MM [on D Mon])`, `**<name> Credits:** unlimited\|<balance>`, `**<name> individual spend limit:** <pct>% left (<used> of <limit> credits used; resets <date>)`, or `**Limits:** data not available yet` | `src/CodexCommands.ts:166-170`, `294-300`, `443-466`, `495-620` |
| Typed failure record (vendor-namespaced `_meta`, client capability gated) | `{id, revision, category, severity: "error"\|"warning", title, details?, actions: ("retry"\|"login"\|"new_session")[]}` | sent only if the client advertises the capability | `rateLimitExceeded` or HTTP 429 → `limit` + `["retry"]`; `usageLimitExceeded` → `limit` + `[]`; `contextWindowExceeded` and `sessionBudgetExceeded` → `limit` + `["new_session"]`. A terminal failure rides on `PromptResponse._meta` next to `quota`. A retry warning or a failure outside the turn goes in `session_info_update`. **No reset time, window, or percentage is attached** | `src/CodexAcpServer.ts:214-233`; `src/CodexEventHandler.ts:77-133`, `157-173`, `970-974`, `1084-1120`; vendor extension docs in the adapter repo |
| Untyped error path, retrying | `session_info_update._meta.codex.error = {message, codexErrorInfo, additionalDetails, turnId, willRetry: true}` | — | Raw Codex error passthrough under a `codex` key | `src/CodexEventHandler.ts:954-968`, `675-682` |
| Untyped error path, terminal | `agent_message_chunk` text, plus, **only for `usageLimitExceeded`**, a JSON-RPC internal error with `data = {message, codexErrorInfo?, additionalDetails?}` | — | A terminal `rateLimitExceeded` becomes only a text chunk followed by `end_turn` | `src/CodexEventHandler.ts:976-981`, `1133-1152` |
| Service error envelope cleanup | text | — | Unwraps `{type:"error", status, error:{type, message}}` for types including `rate_limit_error` and `insufficient_quota` and shows only `message` | `src/ServiceErrorMessage.ts:1-35` |

The adapter's own documentation describes `_meta.quota` as "Token usage and rate limits of the turn"
(vendor extension docs in the adapter repo). The code puts no rate limits in it (`src/CodexAcpServer.ts:3508-3525`; the test snapshot
`src/__tests__/CodexACPAgent/data/token-usage-multiple-updates.json` shows only `token_count` and `model_usage`).

## Delivery mechanism

| Data | Codex side | ACP side | Push or poll | Cadence |
| --- | --- | --- | --- | --- |
| Token usage and context | push `thread/tokenUsage/updated` | push `usage_update`, one per Codex event | push | Per Codex usage event, so several per turn. The test shows three events yielding three `usage_update`s (`token-usage-session-update-multiple.json`) |
| Per-turn usage | same events, last one wins | `PromptResponse.usage` and `_meta.quota` | response | Once per prompt |
| Cumulative tokens | same events (`total`) | `/status` text only | poll (user command) | On demand |
| Rate limits | push `account/rateLimits/updated` (no `threadId`, so it is broadcast to every subscribed session handler, `src/CodexAppServerClient.ts:846-859`); poll `account/rateLimits/read` | `/status` text only | push is cached, never forwarded; poll is user-triggered | On demand |
| Limit hit | push `error` with `codexErrorInfo` | typed failure record, or text / JSON-RPC error | push or response | On occurrence |

Session notification subscription: the adapter registers the per-session app-server handler when a prompt is sent
(`src/CodexAcpServer.ts:3033-3039`). After the prompt ends, the handler keeps routing notifications through
`handleSessionScopedNotification`, which calls the same `handleNotification` for non-error events
(`src/CodexEventHandler.ts:276-280`). So `usage_update` can also be emitted between prompts, for example after
`/compact`. Rate-limit pushes that arrive before a session's first prompt have no handler for that session and are not
cached (inferred from `onServerNotification` registration, `src/CodexAppServerClient.ts:837-859`; `UNVERIFIED` at
runtime).

## Windows and semantics

- **Windows:** the adapter never interprets windows itself. It shows `primary` and `secondary` per `limitId` bucket and
  labels each by `windowDurationMins`: `<60` gives `Nm`, `<1440` gives `Nh`, `<10080` gives `Nd`, anything else gives
  `Weekly` (`src/CodexCommands.ts:586-602`). Multiple buckets come from `rateLimitsByLimitId`, for example a per-model
  alias with `normalModelSlug`. The adapter falls back to the single `rateLimits` value when that map is empty
  (`src/RateLimitsMap.ts:15-33`). The displayed value is "percent left" = `100 - usedPercent`. Absolute limits and
  remaining counts are not available for windows (Codex gives only a percentage). Absolute numbers appear only for the
  spend limit (`used` and `limit` decimal strings).
- **Not surfaced anywhere:** `ordinaryUsageAllowed`, `rateLimitReachedType`, `spendControlReached`, `planType` from the
  snapshot (the account plan comes from a separate account read, `src/CodexCommands.ts:486-487`),
  `rateLimitResetCredits`, `rateLimitUpsell`, `accountId`, `credits.hasCredits`.

### Relation to the three token concepts

1. **Per-turn token usage (`PromptResponse.usage`):**
   - The adapter reports **only the last model request** of the turn (`last` from the final usage event), not the sum
     over the turn. The test "should use last token usage from multiple updates"
     (`src/__tests__/CodexACPAgent/token-usage-events.test.ts:136-161`) feeds `last` = 1000, 1000, 1500 with `total`
     rising 1000 → 2000 → 3500, and expects `usage.totalTokens = 1500`, not 3500.
   - A turn with tool round-trips therefore under-reports usage.
   - It also differs from the ACP unstable schema descriptions of `Usage`, which say "Sum of all token types across
     session" and "Total input tokens across all turns" (`schema/v1/schema.unstable.json`, `$defs.Usage`). The RFD
     leaves per-turn vs cumulative open (`docs/rfds/end-turn-token-usage.mdx`, Open Questions #1).
   - The adapter fits neither reading.
2. **Cumulative session tokens:** the adapter keeps `totalTokenUsage` but sends it only as `/status` text. It is not on
   `PromptResponse`, `usage_update`, or `_meta.quota`, even though the `QuotaMeta.token_count` comment says
   "Aggregated".
3. **Context-window occupancy (`usage_update.used/size`):**
   - `used = last.totalTokens`, which includes the last request's output and reasoning tokens. `size =
     modelContextWindow`.
   - This is the same number that feeds `PromptResponse.usage.totalTokens` and `/status` "Context window".
   - The test name calls it "latest turn usage as a context proxy" (`token-usage-events.test.ts:197`).
   - The legacy Rust adapter used Codex's `last_token_usage.tokens_in_context_window()` instead (see below). Whether
     that helper differs from `last.totalTokens` belongs to the Codex-native note (`UNVERIFIED` here).
   - `usage_update` is skipped when `modelContextWindow` is null (`token-usage-events.test.ts:245-256`).

**Conflations:**

- (a) Per-turn usage and context occupancy share one source (`last`).
- (b) `_meta.quota.token_count` duplicates `PromptResponse.usage` under different key names and claims aggregation it
  does not do.
- (c) `model_usage` is not really per-model.
- (d) The `_meta.quota` name suggests quota or rate limits, but it carries only token counts. Real quota data (the
  rate-limit windows) is cached in the adapter and leaves it only as human-readable `/status` text.

## Legacy adapter (archived zed-industries/codex-acp, `296069e`)

- Consumes `EventMsg::TokenCount(TokenCountEvent { info, .. })`. The `..` discards the event's rate-limit field. The
  adapter emits `usage_update(used = info.last_token_usage.tokens_in_context_window().max(0), size =
  info.model_context_window)` only when the window is known (`src/thread.rs:1128-1137`).
- `PromptResponse` carries only `stopReason`, with no `usage` or `_meta` (`src/codex_agent.rs:799-808`).
- Terminal `EventMsg::Error` becomes a JSON-RPC internal error with `data = {message, codex_error_info}`
  (`src/thread.rs:1370-1383`). `StreamError` (retry) is only logged (`src/thread.rs:1361-1369`).
- It reported no rate limits and had no `/status` command.

## Gaps and unverified items

- `UNVERIFIED`: whether Codex `last` means the last model request or the last turn. The adapter's test fixture and the
  code comment in `src/TokenCount.ts:5` ("Token usage information for a turn") disagree in spirit. Defer to the
  Codex-native note.
- `UNVERIFIED`: the update cadence of `account/rateLimits/updated` from the app server (per response? per turn?). The
  adapter does not depend on it.
- `UNVERIFIED`: runtime behavior for rate-limit pushes before a session's first prompt (inferred above from code only).
- `UNVERIFIED`: routing of `thread/tokenUsage/updated` for native subagent (child) threads. The handler is keyed by
  `threadId`, and child routing goes through `CodexSubagentEventRouter`, which was not inspected.
- The typed failure record's `_meta` key path is vendor-namespaced. It is omitted here on purpose. See
  the vendor extension module, `src/CodexEventHandler.ts:1084-1095`, and vendor extension docs in the adapter repo for the exact
  shape.
- Not exercised against a live Codex backend. Every claim is from source and unit tests at the pinned commits.

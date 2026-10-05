# Gemini CLI: what built-in ACP mode emits for usage and quota

Source snapshot: `google-gemini/gemini-cli` at commit `fb972b2f87fe7d5b06d37eac711490162d98de2c` (2026-10-02, package
version `0.64.0-nightly.20260929`), ACP TS SDK `@agentclientprotocol/sdk` `0.16.1` (`packages/cli/package.json:33`).
The ACP code now lives in `packages/cli/src/acp/` (the old `zed-integration` directory is gone). All `file:line` refs
below are at that commit. Permalink base:
`https://github.com/google-gemini/gemini-cli/blob/fb972b2f87fe7d5b06d37eac711490162d98de2c/`.

## Summary

- ACP mode starts with `--acp` (`--experimental-acp` is a deprecated alias) (`packages/cli/src/config/config.ts:363-370`,
  `:793`). It dispatches to `runAcpClient` (`packages/cli/src/gemini.tsx:874-875`).
- Gemini CLI sends all three token channels: the `usage_update` session notification, `PromptResponse.usage`, and a
  vendor `PromptResponse._meta.quota` object. Despite its name, `_meta.quota` holds only token counts. It has no
  limits, remaining amounts, reset times, windows, tiers, or severity.
- No ACP message carries quota or rate-limit state (limit, remaining, reset, retry-after). Quota exhaustion only shows
  up as a JSON-RPC error on `session/prompt`, with `code: 429` and a text message. Retry delay, quota reason, and
  model fallback details are dropped.
- ACP mode registers no interactive model-fallback handler, so the "switch to fallback model / upgrade / use credits"
  flow never runs. Policy-driven silent fallbacks still happen, but no ACP notification reports them.
- `/stats` (and `/model`, `/quota`, etc.) are not exposed. The ACP command registry holds only `memory`,
  `extensions`, `init`, `restore`, `about`, and `help`. `/about` prints the plan tier as free text.
- How the token channels map: `usage_update.used` approximates context occupancy for the latest model call.
  `PromptResponse.usage` sums every model call in one `session/prompt` (per prompt turn, not per session), which
  re-counts history on each call. Nothing reports cumulative session tokens. The ACP unstable schema's field
  descriptions say "across session"/"across all turns", but Gemini's values are per prompt turn. That is a semantic
  mismatch.

## Fields

### `session/update` with `sessionUpdate: "usage_update"`

| Name | Type | Optional? | Meaning (as implemented) | Source |
|---|---|---|---|---|
| `used` | integer | required, always sent | `promptTokenCount + candidatesTokenCount` from the `usageMetadata` of the latest model response's finish chunk. Thought tokens are excluded. Gemini's `promptTokenCount` covers the whole request (history, system prompt, tools), so this approximates context occupancy after the call. | `packages/cli/src/acp/acpSession.ts:472-491` |
| `size` | integer | required, always sent | `tokenLimit(model)`, a static lookup table: 1,048,576 for known Gemini models and as the default, 256,000 for Gemma 4. It is not dynamic or server-reported. | `acpSession.ts:486-488`; `packages/core/src/core/tokenLimits.ts:20-39` |
| `cost` | `Cost` | never sent | n/a | `acpSession.ts:483-489` (absent) |
| `_meta` | object | never sent | n/a | same |

### `PromptResponse.usage` (ACP unstable `Usage`)

| Name | Type | Optional? | Meaning (as implemented) | Source |
|---|---|---|---|---|
| `inputTokens` | integer | always present on normal and `max_*` completions | Sum of `promptTokenCount` over every model call in this `session/prompt` (tool loop iterations). Each call re-counts the full history. | `acpSession.ts:375, 386, 475, 571` |
| `outputTokens` | integer | same | Sum of `candidatesTokenCount` over calls in this prompt | `acpSession.ts:387, 476-477, 572` |
| `cachedReadTokens` | integer | omitted when 0 (`|| undefined`) | Sum of `cachedContentTokenCount`. Gemini counts it as a subset of the prompt tokens (UNVERIFIED in source; Gemini API semantics). | `acpSession.ts:388, 478-479, 573` |
| `thoughtTokens` | integer | omitted when 0 | Sum of `thoughtsTokenCount` | `acpSession.ts:389, 480-481, 574` |
| `totalTokens` | integer | always present | `inputTokens + outputTokens`. Thought tokens are excluded. | `acpSession.ts:390` |
| `cachedWriteTokens` | integer | never sent | n/a | `acpSession.ts:385-391` |

### `PromptResponse._meta.quota` (vendor extension, snake_case)

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `quota.token_count.input_tokens` | integer | present whenever `usage` is | Same value as `usage.inputTokens` | `acpSession.ts:392-397` |
| `quota.token_count.output_tokens` | integer | same | Same value as `usage.outputTokens` | same |
| `quota.model_usage[]` | array | present (may be `[]`) | Per-model breakdown for this prompt turn. A model is added only if its in+out tokens > 0. The key comes from the resolved model (`GeminiEventType.ModelInfo`), so a silent fallback shows up here only as a different model name. | `acpSession.ts:398-407, 494-496, 576-584`; `packages/core/src/core/client.ts:810` |
| `quota.model_usage[].model` | string | required | Resolved model id | `acpSession.ts:400` |
| `quota.model_usage[].token_count.{input_tokens,output_tokens}` | integer | required | Per-model sums | `acpSession.ts:401-404` |

### Quota-related error on `session/prompt`

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| JSON-RPC `error.code` | integer | required | The HTTP status reused as the JSON-RPC code: `429` for quota or rate limit, and also 499, 503, 403, etc. Defaults to `500`. | `acpSession.ts:510-518, 527-531, 557-560` |
| JSON-RPC `error.message` | string | required | Either the upstream message (JSON blobs unwrapped by `getAcpErrorMessage`), or the fixed string `"Rate limit exceeded. Try again later."` | `acpSession.ts:516, 530, 559`; `packages/cli/src/acp/acpErrors.ts:14-44` |
| JSON-RPC `error.data` | any | never sent | `RequestError` is built without `data`. `retryDelayMs`, quota `reason`, and the Google `ErrorInfo` details are all lost. | `acpSession.ts:514, 528`; SDK `RequestError(code, message, data)` at `https://unpkg.com/@agentclientprotocol/sdk@0.16.1/dist/acp.js` line 913 |

## Delivery mechanism

- **`usage_update`**: pushed. It is sent once per `GeminiEventType.Finished` that carries `usageMetadata`, which means
  once per model call. A prompt with N tool-loop iterations produces about N updates (`acpSession.ts:472-491`).
  `Finished` is emitted only on the chunk that has a `finishReason` (`packages/core/src/core/turn.ts:396-411`). No
  `usage_update` is sent on `session/new`, on `session/load` or resume, or after chat compression
  (`GeminiEventType.ChatCompressed` falls to `default: break`). A grep of `packages/cli/src/acp/*.ts` finds exactly
  one `usage_update` site, `acpSession.ts:484`. The update is sent unconditionally, with no client-capability check.
- **`PromptResponse.usage` and `_meta.quota`**: delivered in the response, once per `session/prompt`, from
  `buildPromptResponse` (`acpSession.ts:381-409`). That builder covers `end_turn`, `max_turn_requests`,
  `max_tokens`, and graceful invalid-stream endings (`acpSession.ts:417, 586-588, 562, 612`).
  - Not sent on `cancelled`. Those returns are a bare `{ stopReason: 'cancelled' }` (`acpSession.ts:422, 442, 538`).
  - Not sent when the prompt fails with a JSON-RPC error, so tokens consumed before the error are never reported.
  - A prompt handled as a slash command returns all-zero `usage` and `_meta.quota` (`acpSession.ts:357-371`).
- **Quota and rate-limit errors**: delivered as the JSON-RPC error response to `session/prompt`. No notification
  precedes them.
  1. The core retry loop (`packages/core/src/utils/retry.ts:340-480`) classifies the Google error into
     `TerminalQuotaError` or `RetryableQuotaError` (each has `status = cause.code` and `retryDelayMs`:
     `packages/core/src/utils/googleQuotaErrors.ts:28-75`). It retries silently. Capacity-exhausted terminal errors
     also become silent retries (`retry.ts:352-395`).
  2. Once retries are exhausted, it calls `onPersistent429`, which maps to `handleFallback`
     (`packages/core/src/core/geminiChat.ts:1093-1114`, `client.ts:1143-1171`).
  3. If there is no fallback, it throws the error, with an extra "Tip: set your own Google Cloud project" paragraph
     appended for Google-login auth when no project is configured (`retry.ts:215-245, 416-418, 467-469`).
  4. `Turn.run` catches the error and yields `GeminiEventType.Error { error: { message, status } }`
     (`turn.ts:416-463`). ACP converts that into `RequestError(status ?? 500, message)` (`acpSession.ts:510-518`).
     For quota errors this gives `code: 429` with the upstream text.
  5. The generic `"Rate limit exceeded. Try again later."` path (`acpSession.ts:527-531`) only fires when an error
     with a numeric `status === 429` is thrown directly out of `sendMessageStream` instead of being yielded. A
     `RequestError` has `code`, not `status`, so step 4's error passes through unchanged at `acpSession.ts:541-543`.
  6. Retry progress (`GeminiEventType.Retry`, `coreEvents.emitRetryAttempt`) is not forwarded. ACP subscribes only to
     `CoreEvent.ApprovalModeChanged` (`acpSession.ts:83-84`), and `Retry` hits `default`. Silent waits are invisible
     to the client.
  7. A 403 `VALIDATION_REQUIRED` error is not handled interactively, because no `ValidationHandler` is registered in
     ACP. It surfaces as an error with code 403 through the same path (inferred from `retry.ts:420-437` and
     `turn.ts:449-462`; UNVERIFIED end to end).
- **Model fallback**: `handleFallback` (`packages/core/src/fallback/handler.ts:26-137`) can switch models silently
  when the policy action is `silent` (`handler.ts:84-96`). Otherwise it calls `config.getFallbackModelHandler()`,
  which ACP never sets. The only registration is the TUI hook
  `packages/cli/src/ui/hooks/useQuotaAndFallback.ts:250`, plus the separate a2a-server. In ACP the handler is
  therefore absent, it returns `null` (`handler.ts:112-115`), and the quota error propagates. Credits, upgrade, and
  "retry later" intents (`handler.ts:155-195`) are unreachable. A silent switch calls `config.setModel(...)`
  (`packages/core/src/config/config.ts:1945-1951`), but no ACP session update reports it. The full list of
  `sessionUpdate:` kinds sent by ACP contains no model or config update (`acpSession.ts` grep: lines 182 through
  1459).
- **Slash commands**: the ACP registry is `packages/cli/src/acp/acpCommandHandler.ts:22-31`. It advertises
  `available_commands_update` (`acpSession.ts:239`) and replies with `agent_message_chunk`
  (`acpSession.ts:622-627`). `/stats`, `/model`, and `/quota` are not available. `/about` emits
  `- Model: ...` and `- Tier: ${getUserTierName()}` as plain text (`packages/cli/src/acp/commands/about.ts:39, 47, 58`).
  That tier string is the only plan or tier information reachable over ACP.
- **Initialize**: `authMethods[]._meta` carries `api-key`/`gateway` hints only. It has no quota capability
  (`packages/cli/src/acp/acpRpcDispatcher.ts:47-79`).

## Windows and semantics

- No quota windows (5h, daily, weekly, per-model), limits, remaining amounts, reset timestamps, overage, or severity
  are reported over ACP. `retryDelayMs`, the only time-like quota datum in core, is dropped at the ACP boundary.
- **(1) Per-turn token usage**: `PromptResponse.usage` and `_meta.quota` are per ACP prompt turn. They sum all model
  calls inside one `session/prompt`. Within a single `sendMessageStream`, a second `Finished` event overwrites the
  first rather than adding to it (`acpSession.ts:475-481` use `= usage.x ?? prev`). Accumulation happens only across
  tool-loop iterations (`acpSession.ts:571-574`). Because every call re-sends the history, `inputTokens` measures
  billed input and can exceed `size` by a wide margin. This diverges from the ACP unstable schema text, where
  `Usage.inputTokens` is "Total input tokens across all turns" and `totalTokens` is the "Sum of all token types
  across session" (`schema/v1/schema.unstable.json:4305`). Gemini's `totalTokens` also leaves out `thoughtTokens`,
  even though the schema says "all token types". `docs/rfds/end-turn-token-usage.mdx:26, 97, 130` lists
  per-turn vs cumulative as an open question.
- **(2) Cumulative session tokens**: not reported by any channel. A client would have to sum
  `PromptResponse.usage`, and that sum undercounts because cancelled and errored prompts carry no usage
  (see Delivery).
- **(3) Context-window occupancy**: `usage_update` `used` and `size`. This matches the stable ACP definition, "Tokens
  currently in context" / "Total context window size" (`schema/v1/schema.json:4216`;
  `docs/rfds/session-usage.mdx:60-67`). Caveats:
  - `used` comes from the last API call's prompt+candidates, and drops thought tokens.
  - It is stale after compression or on session load until the next model call.
  - `size` is a static table. It falls back to 1,048,576 for unknown or alias model ids, and is computed from the
    resolved model reported by `ModelInfo` for that turn.
- **Conflation**: the channels are mostly kept apart. Occupancy goes in `usage_update` and per-prompt billing totals
  go in `PromptResponse`. The two places where they blur are:
  - Labeling: `_meta.quota` uses the word "quota" for plain token counts. A client could mistake these for a quota
    or allowance indicator, which they are not.
  - Scope: `PromptResponse.usage` is per prompt, while the ACP unstable schema describes it as session-cumulative.
  `usage_update` is not cumulative-billing and carries no `cost`.

## Gaps and unverified items

- UNVERIFIED: that Gemini's `promptTokenCount` includes `cachedContentTokenCount`, and that `candidatesTokenCount`
  excludes `thoughtsTokenCount`. Both are Gemini API semantics and are not checked in this repo. They decide whether
  `cachedReadTokens` is a subset of `inputTokens` and whether `used` undercounts.
- UNVERIFIED: the end-to-end JSON-RPC payload for a real 429 was not captured live. The flow is inferred from code and
  the unit tests at `packages/cli/src/acp/acpSession.test.ts:252-305` (usage and usage_update shapes) and `:728-752`
  (status 429 maps to `{ code: 429, message: 'Rate limit exceeded. Try again later.' }`).
- UNVERIFIED: whether any ACP client reads `_meta.quota`. No consumer was checked.
- The local clone is shallow, so the commit that introduced `usage_update` and `_meta.quota` could not be identified.
- Not covered (out of scope, handled by another researcher): the native Code Assist quota API (`retrieveUserQuota`
  and similar) and the TUI `/stats` and `/model` quota displays. Nothing in `packages/cli/src/acp/` calls a quota
  API. A grep for `quota|Quota` in non-test ACP files matched only `acpSession.ts:366, 393`, the `_meta.quota` key.

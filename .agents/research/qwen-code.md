# Qwen Code: quota and rate-limit reporting

Source: github.com/QwenLM/qwen-code, shallow clone of `main` at commit `8d864a9` (2026-10-05). Paths below are relative to the repo root. Gemini CLI was not inspected here. Comparisons to it rely only on the "Google LLC" license headers and on code that was clearly inherited.

## Summary

Qwen Code reports no structured quota or rate-limit data. It has no remaining, used, limit or reset-time fields. It has no quota slash command and no quota push channel. Its ACP mode emits nothing quota-related beyond a bare 429 error.

What exists is error handling and retry:

- Provider 429 errors are detected and classified as transient throttling or permanent quota exhaustion.
- For permanent exhaustion, the provider's error text is passed to the user as free-form text.
- A reset time appears only if the provider's message string contains one. It is never parsed into a field.

The Qwen OAuth free tier (requests per day and per minute) was discontinued on 2026-04-15. There is no free-tier counter to report:

- `docs/users/configuration/auth.md:11-22`
- `docs/users/quickstart.md:77`
- `packages/core/src/utils/retry.ts:382-396`

## Fields

No quota data model exists. The only structured rate-limit data is internal retry metadata, plus a status line payload that has no quota fields.

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `RetryInfo.message` | string | yes | Formatted error text for display. | `packages/core/src/utils/rateLimit.ts:23-33` |
| `RetryInfo.attempt` | number | no | 1-based retry attempt. | same |
| `RetryInfo.maxRetries` | number | no | Maximum retries allowed. | same |
| `RetryInfo.delayMs` | number | no | Delay before the retry. | same |
| `RetryInfo.skipDelay` | function | no | Skips the delay. | same |
| `RateLimitErrorDetails.statusCode` | number | yes | HTTP status of the throttled call. | `rateLimit.ts:35-52` |
| `RateLimitErrorDetails.providerCode` | string | yes | Provider error `code` (for example `insufficient_quota`). | `rateLimit.ts:35-52` |
| `RateLimitErrorDetails.providerType` | string | yes | Provider error `type`. | `rateLimit.ts:35-52` |
| `RateLimitErrorDetails.providerMessage` | string | yes | Provider error text. | `rateLimit.ts:35-52` |
| `RateLimitErrorDetails.requestId` | string | yes | Provider request id. | `rateLimit.ts:35-52` |
| `RateLimitErrorDetails.transport` | `'http'\|'sse'\|'unknown'` | no | Where the throttle surfaced. | `rateLimit.ts:35-52` |
| `StatusLine` input JSON (`context_window.*`, `metrics.models.*.api.{total_requests,total_errors,total_latency_ms}`, `tokens.*`) | numbers | partly | Context-window and token counters handed to a user status-line script. These are not quota fields. | `packages/cli/src/ui/hooks/useStatusLine.ts:40-85` |

`RetryInfo` and `RateLimitErrorDetails` are used by the CLI UI and by logging. They are not sent over ACP (see Delivery mechanism).

## Delivery mechanism

- **Errors, not data.** Quota handling is request-failure handling in `packages/core/src/utils/retry.ts`.
  - **Qwen OAuth quota error.** `isQwenQuotaExceededError` matches HTTP 429, `code === 'insufficient_quota'` and a message containing "free allocated quota exceeded" (`quotaErrorDetection.ts:~108-125`). `retry.ts:382-396` fast-fails with a static message saying the free tier is discontinued.
  - **Permanent quota exhaustion.** `isQuotaExhaustedError` (`quotaErrorDetection.ts:~160-170`) fires when the message string contains "quota", "exhausted" or "exceeded", plus "will reset" or "reset at". The example in the code is a Bailian token-plan "1-week quota ... will reset at 07-27 09:25:00 UTC". `retry.ts:404-418` fast-fails with `Quota exhausted: <provider text>` plus a hint. The reset time exists only inside that string.
  - **Throttling.** 429, 503, 1302 and 1305 are treated as rate limit (`rateLimit.ts:14-18`). Retry-After headers (`retry-after-ms`, `retry-after`) are read for backoff only (`retryPolicy.ts:82-98`). `Throttling.AllocationQuota` is classed fail-fast (`retryErrorClassification.ts:371`; `retry.ts:429-436`).
  - **Message rewriting.** Rate-limit messages are rewritten per auth type in `packages/core/src/utils/errorParsing.ts:15-35`. The Gemini and Vertex wording ("request a quota increase through AI Studio / Vertex") is inherited from Gemini CLI.
- **Slash commands.** `/stats` (alias `/usage`) shows session duration, prompt count, API request count, token counts, tool calls and file diffs (`packages/cli/src/ui/commands/statsCommand.ts:709-760`). It is supported in the `acp` execution mode. It has no quota, limit or remaining output.
- **Status line.** `useStatusLine.ts:97` and `:235` mention "quota" only in a comment: a user script may fetch quota itself and the command can be re-run every N seconds via `refreshInterval`. Nothing is supplied by the CLI.
- **ACP mode.** The ACP session handles `StreamEventType.RETRY` and `MODEL_FALLBACK` only to rewind output buffers (`packages/cli/src/acp-integration/session/Session.ts:6995-7015`). It does not forward `RetryInfo` to the client.
  - On a 429 it throws `new RequestError(429, 'Rate limit exceeded. Try again later.')` (`Session.ts:7099-7103`, `:8231`). There is no `data` payload, no reset time and no quota identity.
  - There is no `usage_update` or other ACP usage notification in `packages/cli/src/acp-integration` (grep for `usage_update` returned nothing).
- **Push or poll.** Neither. Nothing is polled or pushed. Errors are discovered only when a request fails.

## Windows and semantics

- None are modeled. The code recognizes provider text naming a window ("1-week quota", token-plan or Coding Plan weekly) only as a substring to match. It does not extract the window.
- Alibaba Coding Plan is described as "Weekly quota included" in the provider preset description (`packages/core/src/providers/presets/alibaba-coding-plan.ts:99`). No usage fetch exists for it.
- TPM and RPM throttling is treated as transient, lifting "within seconds-minutes". Allocated quota is treated as permanent until a reset time (`quotaErrorDetection.ts` doc comment on `isQuotaExhaustedError`).
- Context-window occupancy and token usage are tracked separately in the status line payload and `/stats`. They are not related to quota.

## Divergence from Gemini CLI

- **Inherited helpers with no remaining caller.** `isProQuotaExceededError` and `isGenericQuotaExceededError` are carried over from the Gemini CLI fork. They match Gemini "Quota exceeded for quota metric" text. A grep found no non-test callers outside their own file (`quotaErrorDetection.ts:38,87`). `modelSwitchedFromQuotaError` state remains in the UI (`AppContainer.tsx:865`, `use-llm-stream.ts:571,5752`). The only call to its setter passes `false` (`use-llm-stream.ts:3727`), so it is never set to true.
- **Qwen-specific additions.**
  - A Qwen OAuth free-tier detector, now only producing a "discontinued" message.
  - A token-plan "reset at" string detector.
  - DashScope and Z.AI error codes 1302 and 1305.
- **No quota feature in /stats.** I could not verify what Gemini CLI's stats output contains. Qwen's has no quota section.

## Gaps and unverified items

- No server-side quota API or response header carrying remaining quota was found. Searches for `x-ratelimit`, `ratelimit-` and `retry-after` found only the Retry-After handling.
- Gemini CLI behavior is UNVERIFIED. This note did not inspect that repository.
- Whether the provider (DashScope, Bailian) emits structured quota headers is UNVERIFIED. Qwen Code ignores them in any case.
- The `packages/acp-bridge`, `packages/sdk-*` and `packages/web-shell` packages were only grepped for `quota|rate.?limit` hits. The quota-related hits were in IPC, peer admission and compaction code, which are unrelated to model quota. A hit in `packages/acp-bridge/src/session-control-plane.ts` was not inspected.
- The `docs/design/daemon-acp-capacity-admission-pr1.md` hit concerns daemon-side session capacity admission, not model quota. It was not read.

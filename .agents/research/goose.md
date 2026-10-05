# Goose (block/goose): rate limits and quotas

Source: shallow clone of https://github.com/block/goose at commit d9dda30 (2026-10-05). Paths are relative to the repo root.

## Summary

Goose reports no quota, usage-limit, remaining or reset information anywhere (CLI, desktop UI, ACP). It only handles per-request provider errors:

- A 429 or rate-limit error becomes `ProviderError::RateLimitExceeded { details, retry_delay }`.
- The provider layer retries it silently with backoff, honoring `retry_delay` when the provider supplied one.
- If retries run out, the user sees a plain-text message ("Ran into this error: Rate limit exceeded: ...").
- Retries are logged with `tracing::warn!` only. No ACP or UI event is emitted for a retry.
- Goose does not parse `x-ratelimit-*`, `anthropic-ratelimit-*` or any 5h/weekly window headers. Grepping `crates/` and `ui/` for `x-ratelimit`, `ratelimit`, `used_percent`, `resets_at`, `five_hour`, `weekly` and `usage_limit` finds nothing relevant.
- The only quota-like signal is `CreditsExhausted` (HTTP 402 or Anthropic `billing_error`). It reaches ACP as a JSON-RPC error with `data.reason = "credits_exhausted"` and an optional `data.url`.
- ACP usage output is context-window occupancy and cost, not limits.

Verdict: nothing beyond errors and retries, plus the credits-exhausted error. Goose is not a data source for quota fields. It is a counter-example.

## Fields

### Internal error types (not on the ACP wire)

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `ProviderError::RateLimitExceeded.details` | string | no | Provider message text, free-form | `crates/goose-provider-types/src/errors.rs:18-22` |
| `ProviderError::RateLimitExceeded.retry_delay` | `Option<Duration>` | yes, usually `None` | Server-suggested wait before retry | `errors.rs:21` |
| `ProviderError::CreditsExhausted.details` | string | no | Provider message | `errors.rs:51-54` |
| `ProviderError::CreditsExhausted.top_up_url` | `Option<String>` | yes | Link to add credits | `errors.rs:53` |

Where `retry_delay` is populated:

- Generic HTTP path (`crates/goose-providers/src/http_status.rs:47-63`, applied at `:414-419`).
  - It prefers the body field `error.metadata.retry_after_seconds` (an f64, OpenRouter shape).
  - It falls back to the `Retry-After` header, as delta-seconds or an HTTP-date.
  - It clamps the value to 3600 s (`:41`). Negative, NaN or infinite values give `None`.
- Gemini OAuth provider (`crates/goose/src/providers/gemini_oauth.rs:822-829`, `:906-910`). It scrapes digits after "after " in the 429 body text, for example "reset after 15s".
- Every other provider sets `retry_delay: None`. Examples:
  - Anthropic `rate_limit_error`: `crates/goose-providers/src/anthropic.rs:289-292`.
  - Codex CLI (substring match on "rate limit"): `crates/goose/src/providers/codex.rs:399-403`.
  - Kimi: `kimicode.rs:359`.
  - Muse Code: `muse_code_def.rs:128`.
  - Plain 429: `http_status.rs:286-289`.
- `gcpvertexai.rs:294-380` has its own 429 retry loop using `retry_config.delay_for_attempt`. It does not read a server delay.

### Surfaced to clients

| surface | name | type | optional? | meaning | source |
|---|---|---|---|---|---|
| Final assistant text (the ACP agent-message chunk path) | n/a | free text | n/a | "Ran into this error: Rate limit exceeded: {details}.\n\nPlease retry if you think this is a transient or recoverable error." No structured fields. | `crates/goose/src/agents/agent.rs:3292-3300`; `crates/goose-provider-types/src/conversation/message.rs:1269-1284` |
| `MessageContent::Error` | `kind` | enum `authentication`, `contextLengthExceeded`, `creditsExhausted`, `other` | no | Rate limit maps to `other`, so it is indistinguishable from other errors. | `message.rs:261-280` |
| ACP error for credits | JSON-RPC code `-32603`; `data.reason` | string `"credits_exhausted"` | no | Quota-like prompt failure | `crates/goose/src/acp/server.rs:1668-1681`, `:1685-1707`; `crates/goose/src/acp/mod.rs:24` |
| same | `data.url` | string | yes | Top-up URL, from `notification.data.top_up_url` | `server.rs:1696-1703` |
| Goose SDK (uniffi) `GooseStreamError` | `kind=RateLimited`, `message`, `retry_after_ms` | enum, string, `Option<u64>` | `retry_after_ms` optional | Programmatic rate-limit signal for embedders. This is the Goose SDK, not ACP. | `crates/goose-sdk/src/bindings.rs:45-48`, `:87-95`, `:609-636` |

When Goose is an ACP client wrapping another ACP agent, `provider_error_from_acp` maps only `AuthRequired` to Authentication. Every other ACP error becomes `RequestFailed(error.to_string())`. Any rate-limit structure from the downstream agent is lost (`crates/goose/src/acp/provider.rs:182-188`).

### ACP usage output (context and cost only)

`build_usage_updates` (`crates/goose/src/acp/server.rs:771-801`) produces two notifications.

- Standard ACP `UsageUpdate` with `used` (u64), `size` (u64, the context limit) and optional `cost { amount, currency: "USD" }`. `cost` is set only when `totals.accumulated_cost` is `Some`.
- Goose custom `SessionUsageUpdate` (`crates/goose-sdk-types/src/custom_notifications.rs:73-83`), with these fields:
  - `used`
  - `contextLimit`
  - `accumulatedInputTokens`
  - `accumulatedOutputTokens`
  - `accumulatedCost`, optional
- Per-message `MessageUsageUpdate` (`server.rs:1762-1783`) with these fields:
  - `inputTokens`, `outputTokens`, `totalTokens`
  - `cacheReadTokens`, `cacheWriteTokens`
  - `cost`, `costSource` (`providerReported` or `estimated`)
  - `elapsedMs`, `timeToFirstTokenMs`, `isCompaction`
- Prompt response `Usage::new(total, input, output)` (`server.rs:729-734`).

The custom notifications are sent only when the client supports Goose custom notifications (`server.rs:2205`). None of these carry limit, remaining or reset data.

## Delivery mechanism

- Provider-level retry (`crates/goose-provider-types/src/retry.rs`).
  - Retries `RateLimitExceeded`, `ServerError` and `NetworkError` (`:106-115`).
  - Defaults are 3 retries, 1000 ms initial interval, 2.0 multiplier, 30 s max (`:14-17`).
  - The delay is the provider-supplied `retry_delay` if present, else the exponential backoff (`:141-150`).
  - Each retry is logged with `tracing::warn!` ("Request failed, retrying (n/N)", `:135`). It is not pushed to clients.
- Stream-start retry in the agent loop (`crates/goose/src/agents/reply_parts.rs:416-446`). It retries when the stream fails before its first item, with the same delay logic and a `warn!` log. The env var `GOOSE_PROVIDER_SKIP_BACKOFF` skips the sleep.
- Terminal failure.
  - Rate limit: plain-text assistant message (`agent.rs:3292-3300`). `provider_errored` is set, with no structured event.
  - Credits exhausted: a `SystemNotification` of type `creditsExhausted` with `data.top_up_url` (`agent.rs:3223-3249`). The ACP server turns it into a JSON-RPC error (`server.rs:1660-1666`). Chat clients get no status message for it (`server.rs:1757` returns `None`).
- Push versus poll: push only, and only for context usage and cost. There is no poll endpoint for limits. I found no slash command, status line or local file for limits.
- Docs (`documentation/docs/guides/handling-llm-rate-limits-with-goose.md`) only advise switching to Tetrate or OpenRouter and say nothing about reporting.

## Windows and semantics

None. Goose has no model of windows (5h, weekly, per-model), remaining quota, reset time, overage or severity. `retry_delay` is a relative one-shot hint for a single 429, clamped to 1 h. Only the SDK exposes it (`retry_after_ms`), and ACP clients never see it. The only budget-like numbers are the context window (`used` / `size`) and accumulated cost, which are tracked separately from provider limits.

## Gaps and unverified items

- UNVERIFIED: whether the desktop UI (`ui/desktop`) renders anything limit-related from the SDK `GooseStreamError.retry_after_ms`. Grepping `ui/**/*.ts(x)` for `ratelimit` or `retry_after` found only analytics comments (`ui/desktop/src/utils/analytics.ts:28`).
- UNVERIFIED: the `goose-cli` terminal rendering of rate-limit errors. Only the shared message text path was traced.
- Not checked: Goose's wrapped-CLI providers other than Codex (for example claude-code, gemini-cli adapters). The repo-wide grep showed no rate-limit-event ingestion from them.
- No evidence of `_meta` carrying limit data in any Goose ACP message.

# OpenCode (anomalyco/opencode, formerly sst/opencode): rate limits and quotas

Source: shallow clone of `https://github.com/anomalyco/opencode`, branch `dev`, commit `907b3bc518fa48e90e8ec24dd327d13eee71c36c` (2026-10-02).
Paths below are relative to `packages/` in that repo.

## Summary

Triage verdict: OpenCode reports **no structured quota or rate-limit state to ACP clients**. In ACP mode it sends only:

- `usage_update` with context occupancy and session cost in USD (`used`, `size`, `cost`).
- Per-turn token `usage` on the prompt response.
- Provider failures, as a flattened error string.

Provider limits do not reach ACP clients as structured data. This is the detail:

- It has an internal retry loop that handles 429 and similar errors, using `retry-after` headers.
- It emits an internal `session.status` event of type `retry`. The event has `attempt`, `message`, `next` and an optional `action`.
- The OpenCode TUI and web app consume that event. The ACP bridge ignores everything except `idle` (`opencode/src/acp/event.ts:95-96`).
- Only OpenCode's own hosted gateways (Zen and "Go" subscription) produce quota-like data: window name (`5 hour`, `weekly`, `monthly`), reset seconds and a workspace id. That data is flattened into an English message string plus an optional UI action. It does not go through ACP.
- For Claude Pro/Max and Copilot there is no quota code. A grep of `opencode/src/provider` and `opencode/src/plugin` found no quota or rate-limit handling (see Gaps).

## Fields

### ACP output (what an ACP client actually receives)

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `sessionUpdate: "usage_update"` | ACP `SessionUpdate` variant | emitted only if a latest assistant message with `providerID` and `modelID` exists and the model's context limit is known | Standard ACP usage update | `opencode/src/acp/usage.ts:213`; duplicate in `opencode/src/acp/service.ts:673` |
| `used` | number | no (when emitted) | `tokens.input + tokens.cache.read + tokens.cache.write` of the latest assistant message. This is context occupancy. | `opencode/src/acp/usage.ts:86-88`, `:214` |
| `size` | number | no (when emitted) | Model context window, `Provider.Info.models[id].limit.context` | `opencode/src/acp/usage.ts:117-123`, `:215` |
| `cost` | `{amount, currency: "USD"}` | no (when emitted) | Sum of `cost` over all assistant messages in the session. This is a spend estimate, not a quota. | `opencode/src/acp/usage.ts:111-116`, `:216` |
| `usage` on prompt response | ACP `Usage` | present only if the assistant message info exists | `inputTokens`, `outputTokens`, `totalTokens`; `thoughtTokens`, `cachedReadTokens`, `cachedWriteTokens` only when > 0 | `opencode/src/acp/usage.ts:90-105`, `opencode/src/acp/service.ts:835-856` |
| prompt-response `_meta` | `{}` | always empty | No quota data is placed in `_meta` | `opencode/src/acp/service.ts:848`, `:855` |

### Internal retry status (not forwarded to ACP)

`session.status` event, `status.type == "retry"`. Schema at `schema/src/session-status-event.ts` (`Info`, `Status`).

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `type` | `"retry"` (union with `"idle"` and `"busy"`) | no | Status discriminator | `schema/src/session-status-event.ts:14` |
| `attempt` | non-negative int | no | Retry attempt number | same file, `:14-16` |
| `message` | string | no | Human-readable reason | same file, `:14` |
| `next` | non-negative int | no | Epoch ms of the next attempt (`now + wait`) | same file, `:27`; set at `opencode/src/session/retry.ts:201` |
| `action` | struct | yes | UI call-to-action | same file, `:17-25` |
| `action.reason` | string. Known values: `"free_tier_limit"`, `"account_rate_limit"`; type is open (`string & {}`) | no (inside `action`) | Why the limit was hit | `opencode/src/session/retry.ts:12` |
| `action.provider` | string | no | Provider id | `retry.ts:14-21` |
| `action.title`, `action.message`, `action.label` | string | no | Display text | `retry.ts:14-21` |
| `action.link` | string | yes | URL, for example `https://opencode.ai/workspace/<workspace>/go` | `retry.ts:20`, `:129-130` |

The ACP bridge handles `session.status` only to detect `idle` (`opencode/src/acp/event.ts:95-96`). The `retry` status is dropped, so ACP clients never see it.

### Server-side hosted-gateway 429 (OpenCode Zen and Go)

Response body of the hosted gateway:

- Status 429 (`console/app/src/routes/zen/util/handler.ts:507-531`).
- Header `retry-after` is set only if `retryAfter` is truthy (`handler.ts:512-514`).
- Body: `{type:"error", error:{type: <class name>, message}, metadata}`.
- `metadata` is `{workspace, limitName}` for `GoUsageLimitError`, otherwise `{}`.

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `error.type` | `"RateLimitError"`, `"FreeUsageLimitError"`, `"GoUsageLimitError"` or `"BlackUsageLimitError"` | no | Which limit class fired | `console/app/src/routes/zen/util/error.ts:15-29`, `handler.ts:507-531` |
| `metadata.workspace` | string | only for Go | Workspace id | `error.ts:21-28` |
| `metadata.limitName` | `"5 hour"` \| `"weekly"` \| `"monthly"` | only for Go | Which window tripped | `error.ts:20-21`, `handler.ts:911-953` |
| `retry-after` header | integer seconds as string (`resetInSec`) | yes | Time until the window resets | `handler.ts:512-514`, `console/core/src/subscription.ts:73-90` |

Client side, `opencode/src/session/retry.ts:112-140` parses the body for `GoUsageLimitError` and turns it into the retry `message` (including a "reset in N days/hours/minutes" string) and the `action` above. `FreeUsageLimitError` becomes `reason: "free_tier_limit"` with an upsell (`retry.ts:99-110`).

### Internal LLM-layer rate-limit details (also not surfaced)

`llm/src/schema/errors.ts:18-23` defines `HttpRateLimitDetails`:

- `retryAfterMs?: number`.
- `limit?`, `remaining?`, `reset?`: each a `Record<string,string>` keyed by suffix.

`llm/src/route/executor.ts:112-146` fills it from:

- `x-ratelimit-{limit,remaining,reset}-<suffix>` (OpenAI style).
- `anthropic-ratelimit-<suffix>-{limit,remaining,reset}`.

It is attached to `RateLimitReason.rateLimit` and `HttpContext.rateLimit`, and is only populated on error responses with status >= 400 (`executor.ts:274-310`). A grep for `rateLimit` and `HttpRateLimitDetails` in `opencode/src` finds no consumer. The session retry logic reads raw `responseHeaders["retry-after-ms"|"retry-after"]` instead (`retry.ts:49-69`). This code is the nearest thing to a structured limit/remaining/reset triple. It is not wired to any user-visible or ACP surface at this commit.

## Delivery mechanism

- Push or poll for ACP: neither. There is no quota channel. `usage_update` is pushed once at the end of each prompt turn, not on a timer or stream:
  - after a normal prompt (`opencode/src/acp/service.ts:542`);
  - after a known slash command (`:566`);
  - after `/compact` or an unknown command (`:588`).
  - It is not sent while the turn is in flight.
- Failure path in ACP: after retries are exhausted, an assistant error becomes `ACPError.ServiceFailureError({service:"session", safeMessage: <error.data.message>, errorName})` (`opencode/src/acp/service.ts:883-886`, `promptErrorMessage` at `:890`).
  - Exceptions: `ProviderAuthError` becomes `AuthRequiredError`; abort, output-length and content-filter map to `cancelled`, `max_tokens` and `refusal` stop reasons (`:850-881`).
  - A 429 or quota error therefore reaches the ACP client only as a JSON-RPC error whose message is free text. There is no code or structured data.
- Retry loop: up to `RETRY_MAX_RETRIES = 5` (`retry.ts:31`).
  - Initial delay 2 s, factor 2, jitter 0.25 (`retry.ts:26-28`).
  - Max delay 30 s when there are no response headers (`retry.ts:29`).
  - `retry-after-ms` and `retry-after` (seconds or HTTP date) are honored when headers exist (`retry.ts:49-72`).
  - Retryable detection is regex-based on message and body (`retry.ts:34-42`) plus status >= 500.
- OpenCode's own UIs (TUI `tui/src/routes/session/index.tsx:89-120`, web `app/src/pages/session/usage-exceeded-dialogs.tsx`) read the `retry.action` and show an upsell or settings dialog for providers `opencode` and `opencode-go` only (`GO_UPSELL_PROVIDERS`, `tui/src/routes/session/index.tsx:92`).
- Local usage display: the TUI prompt footer shows context tokens, percent of context window, and session cost (`tui/src/component/prompt/index.tsx:264-282`). It shows no quota.

## Windows and semantics

- None are exposed to ACP clients.
- On the hosted gateway (server-side, `console/core/src/subscription.ts`):
  - Rolling window: configurable `rollingWindow` in hours, labelled "5 hour" in errors (`subscription.ts:56-90`; `handler.ts:951`). Reset time is `timeRollingUpdated + window`, so the window starts at first use, not at a fixed clock boundary.
  - Weekly and monthly windows, with `getWeekBounds` and `getMonthlyBounds` (`subscription.ts:5`, `:93+`).
  - Each analysis returns `{status: "ok"|"rate-limited", resetInSec, usagePercent}` (`subscription.ts:65-90`). `usagePercent` is floor(0..100) and reaches the OpenCode client only on the console web page, in referral preview (`console/core/src/referral.ts:243+`). It is not in 429 responses.
  - Limits are in dollar-equivalent micro-cents of model usage, not in tokens or requests (`subscription.ts:63`).
  - Free tier: `promoTokens`, `dailyRequests`, `dailyRequestsFallback` (`subscription.ts:13-18`); IP rate limit via `ipRateLimiter.ts:40`.
- Overage: the Go error text says "enable usage from your available balance" (`retry.ts:130`). Server side, `billing.lite.useBalance` makes the gateway swallow the limit error and bill the balance (`handler.ts:964`). This is not exposed as a field.
- Relation to token usage: none for quota. Context occupancy is separate from quota. `usage_update.used` excludes output and reasoning tokens (`usage.ts:86-88`), while the per-turn `usage.totalTokens` includes them (`usage.ts:94-96`). The TUI footer sums all five token kinds, which is a different definition from `contextTokens` (`tui/.../prompt/index.tsx:270-271`).

## Gaps and unverified items

- UNVERIFIED: the hosted `GET /zen/go/v1/usage` endpoint. `console/app/src/lib/inference-proxy.ts:18` maps it to `/go/v1/usage` on a separate inference service whose source is not in this repo. A grep for `go/v1/usage` finds no OpenCode client calling it, and its response shape is unknown.
- UNVERIFIED: Claude Pro/Max and GitHub Copilot subscription limits.
  - `opencode/src/plugin/github-copilot/*` exists, but I did not read it in full.
  - Greps of `opencode/src/provider` and `opencode/src/plugin` found no quota, usage-limit or rate-limit logic beyond `provider/error.ts`.
  - `provider/error.ts:117-128` maps OpenAI `insufficient_quota` to the message "Quota exceeded. Check your plan and billing details." and `usage_not_included`, both non-retryable.
  - Treat provider-subscription quotas as unreported.
- UNVERIFIED: the full installed-package ACP behavior may differ from `dev`. Only `dev` at the commit above was inspected, not a tagged release.
- `Retryable.action` fields, `GoUsageLimitError.metadata` and the `retry-after` header exist only for OpenCode's own gateway. Other providers' 429 bodies and `anthropic-ratelimit-*` or `x-ratelimit-*` headers are parsed in the `llm` package, but nothing downstream uses them.
- `usage_update.used` and `size` are omitted entirely when the model's context limit is unknown (`usage.ts:206`). `cost` is therefore not sent in that case either.
- Takeaway for the RFD: OpenCode is a negative example. The data exists internally (window name, reset seconds, `limit`/`remaining`/`reset` maps, percent used) but is dropped at the ACP boundary and collapsed to a string.

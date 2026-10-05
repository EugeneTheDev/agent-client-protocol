# Gemini CLI (native): quota and rate-limit data model and delivery

Scope: Gemini CLI by itself, without ACP. The ACP integration is covered in `gemini-acp.md`.

Source: https://github.com/google-gemini/gemini-cli at commit `fb972b2f87fe7d5b06d37eac711490162d98de2c` (2026-10-02),
shallow clone. All `path:line` references are relative to the repo root at that commit. Permalink form:
`https://github.com/google-gemini/gemini-cli/blob/fb972b2f87fe7d5b06d37eac711490162d98de2c/<path>#L<line>`.

Abbreviations: `CA` = `packages/core/src/code_assist`, `CFG` = `packages/core/src/config/config.ts`,
`GQE` = `packages/core/src/utils/googleQuotaErrors.ts`, `UI` = `packages/cli/src/ui`.

## Summary

Gemini CLI has three separate sources of quota information. Only the first is structured:

1. **Code Assist `retrieveUserQuota` (poll).** Only available with Google sign-in (OAuth `LOGIN_WITH_GOOGLE` or
   `COMPUTE_ADC`), where requests go through the Code Assist backend (`cloudcode-pa.googleapis.com/v1internal`). The
   response is a flat list of **buckets**. Each bucket has optional `modelId`, `tokenType`, `remainingAmount` (an int64
   string), `remainingFraction` (0..1), and `resetTime` (ISO timestamp). The bucket has **no explicit limit and no
   window label**. The client computes the limit as `remainingAmount / remainingFraction`, or uses a normalized scale
   of 100 when only the fraction is present. The UI assumes the window is daily, and the text "reset daily" is
   hard-coded. The client polls at startup, from `/stats` and `/model manage`, and after each model response if the
   cached value is older than 30 s.
2. **Errors (push, in-band).** These are HTTP 429/499/503 responses carrying a `google.rpc` error. The error details
   are `ErrorInfo.reason` (`RATE_LIMIT_EXCEEDED`, `QUOTA_EXHAUSTED`, `MODEL_CAPACITY_EXHAUSTED`/`_EXCEEDED`,
   `INSUFFICIENT_G1_CREDITS_BALANCE`), `QuotaFailure.violations[].quotaId` (which contains `PerDay`, `Daily`, or
   `PerMinute`), and `RetryInfo.retryDelay`. The client sorts each error into `TerminalQuotaError` (daily, exhausted,
   capacity, or a delay over 5 min) or `RetryableQuotaError` (per-minute, or a short delay). These classes drive
   retry/backoff, the Pro-to-Flash fallback dialog, and the AI-credits (overage) flow. This path applies to every auth
   type, including API key and Vertex.
3. **Tier and credits from `loadCodeAssist` and `generateContent` responses.** `currentTier`/`paidTier` carry `id`
   and `name`. `paidTier.availableCredits[]` holds the Google One AI credits balance. Each Code Assist generate
   response carries `consumedCredits[]`/`remainingCredits[]`, so the credits balance arrives as a push on every
   response.

API-key (Gemini API) and Vertex users get **no proactive quota data**. They see only 429 errors with a generic "request
a quota increase" hint. The client does not read any rate-limit response headers.

Quota is reported separately from token usage and context-window occupancy. Buckets are counted in server-defined
units. The docs describe the limits as "requests per user per day", but `tokenType` hints that a bucket could be
token-based (UNVERIFIED). The client never uses `tokenType`.

## Fields

### A. `retrieveUserQuota` wire types

| Name                                   | Type          | Optional?                         | Meaning                                                                                       | Source              |
| -------------------------------------- | ------------- | --------------------------------- | --------------------------------------------------------------------------------------------- | ------------------- |
| `RetrieveUserQuotaRequest.project`     | string        | required                          | Code Assist project ID (`cloudaicompanionProject`)                                            | `CA/types.ts:250-253` |
| `RetrieveUserQuotaRequest.userAgent`   | string        | optional                          | Not set by the client                                                                         | `CA/types.ts:252`, `CFG:2319-2321` |
| `RetrieveUserQuotaResponse.buckets`    | `BucketInfo[]` | optional                         | List of quota buckets. If it is absent, the client keeps its previous cache                   | `CA/types.ts:263-265`, `CFG:2323` |
| `BucketInfo.remainingAmount`           | string (int64) | optional                         | Absolute remaining units. The client parses it with `parseInt`                                | `CA/types.ts:256`, `CFG:2352-2353` |
| `BucketInfo.remainingFraction`         | number        | optional                          | Remaining share, 0..1. Buckets without it are skipped                                         | `CA/types.ts:257`, `CFG:2328-2330` |
| `BucketInfo.resetTime`                 | string (timestamp) | optional                     | Time the bucket resets, parsed with `new Date()`                                              | `CA/types.ts:258`, `UI/utils/formatters.ts:101-110` |
| `BucketInfo.tokenType`                 | string        | optional                          | Unit or type of the bucket. **Never read by the client**, and its values are UNVERIFIED      | `CA/types.ts:259` (no other refs) |
| `BucketInfo.modelId`                   | string        | optional                          | The model the bucket applies to. Buckets are per model, and buckets without it are skipped    | `CA/types.ts:260`, `CFG:2329` |

The endpoint is `POST {CODE_ASSIST_ENDPOINT}/v1internal:retrieveUserQuota` (`CA/server.ts:73-74`,
`CA/server.ts:367-374`, `CA/server.ts:532-533`). The bucket has no field for a limit, a window name or duration,
usage, a severity or status, or overage.

### B. Client-derived quota model (in memory)

| Name                                      | Type                                         | Optional? | Meaning                                                                                               | Source        |
| ----------------------------------------- | -------------------------------------------- | --------- | ----------------------------------------------------------------------------------------------------- | ------------- |
| `modelQuotas`                             | `Map<modelId, {remaining, limit, resetTime?}>` | -       | Per-model cache. `limit = round(remainingAmount/remainingFraction)`. If only the fraction is sent, `limit=100` and `remaining=round(fraction*100)` | `CFG:859-862`, `CFG:2349-2371` |
| pooled `remaining` / `limit`              | number                                        | optional  | Used in "auto" model mode: the Pro and Flash buckets are **summed**                                  | `CFG:2056-2101` |
| pooled `resetTime`                        | string                                        | optional  | The latest of the Pro and Flash reset times ("most conservative")                                    | `CFG:2086-2091` |
| `QuotaChangedPayload.remaining`/`limit`   | `number \| undefined`                         | required key, value may be undefined | Payload of the internal `quota-changed` event                                      | `packages/core/src/utils/events.ts:190-194`, `:438-445` |
| `QuotaChangedPayload.resetTime`           | string                                        | optional  | Same as above                                                                                         | same          |
| `hasAccessToPreviewModel`                 | boolean                                       | -         | Set when any bucket's `modelId` is a preview model. This means the bucket list also serves as an entitlement list | `CFG:2375-2379` |

Model IDs are normalized: the legacy, base, and latest Flash and Flash-Lite IDs are mapped to whichever variant the
user can access (`CFG:2333-2347`).

### C. Tier and credits

| Name                                   | Type                      | Optional? | Meaning                                                                                 | Source |
| -------------------------------------- | ------------------------- | --------- | --------------------------------------------------------------------------------------- | ------ |
| `LoadCodeAssistResponse.currentTier`   | `GeminiUserTier \| null`  | optional  | The user's tier                                                                          | `CA/types.ts:82-88` |
| `LoadCodeAssistResponse.paidTier`      | `GeminiUserTier \| null`  | optional  | Paid subscription tier. When present, it takes precedence over `currentTier` for id and name | `CA/types.ts:87`, `CA/setup.ts:217-236` |
| `GeminiUserTier.id`                    | `UserTierId` (string)     | optional  | Known values are `free-tier`, `legacy-tier`, `standard-tier`. Other values are passed through as open strings | `CA/types.ts:93-94`, `:150-156` |
| `GeminiUserTier.name`                  | string                    | optional  | Display name, shown as "Tier:" in `/stats`                                               | `CA/types.ts:95`, `UI/commands/statsCommand.ts:32`, `UI/components/StatsDisplay.tsx:318-321` |
| `GeminiUserTier.availableCredits`      | `Credits[]`               | optional  | AI credits balance                                                                       | `CA/types.ts:103-104` |
| `Credits.creditType`                   | `'CREDIT_TYPE_UNSPECIFIED' \| 'GOOGLE_ONE_AI'` | required | Type of credit                                                       | `CA/types.ts:46`, `:53-56` |
| `Credits.creditAmount`                 | string (int64)            | required  | Credit amount                                                                            | `CA/types.ts:55` |
| `CaGenerateContentResponse.consumedCredits` | `Credits[]`          | optional  | Credits consumed by this response, summed for telemetry                                  | `CA/converter.ts:77-82`, `CA/server.ts:163-169` |
| `CaGenerateContentResponse.remainingCredits` | `Credits[]`         | optional  | Balance after this response. Replaces the cached G1 balance                              | `CA/server.ts:170-178`, `:234-255` |
| `overageStrategy` setting              | `'ask' \| 'always' \| 'never'` (default `ask`) | - | Whether to spend credits when a quota is exhausted                                 | `packages/cli/src/config/settingsSchema.ts:1002-1019`, `packages/core/src/billing/billing.ts:24` |
| `MIN_CREDIT_BALANCE`                   | const 50                  | -         | Credits are auto-used, or the overage menu is shown, only at or above this balance. Below it, the "empty wallet" menu is shown | `billing.ts:137-188` |

The overage (credits) path applies only to the models in `OVERAGE_ELIGIBLE_MODELS`, which are the preview Pro, 3.1
Pro, and preview Flash models (`billing.ts:33-37`). When credits are used, the request sends `enabledCreditTypes:
['GOOGLE_ONE_AI']` (`CA/server.ts:113-125`).

### D. Quota errors

| Name                                      | Type     | Optional? | Meaning                                                              | Source |
| ----------------------------------------- | -------- | --------- | -------------------------------------------------------------------- | ------ |
| `GoogleApiError.code`/`message`/`details` | number/string/array | required | Parsed `google.rpc.Status`                                  | `packages/core/src/utils/googleErrors.ts:127-131` |
| `ErrorInfo.reason`                        | string   | required  | The fields listed above, e.g. `RATE_LIMIT_EXCEEDED` or `QUOTA_EXHAUSTED` | `googleErrors.ts:36-41` |
| `ErrorInfo.domain`                        | string   | optional  | `cloudcode-pa.googleapis.com` and its staging/autopush variants enable Code Assist-specific handling | `googleErrors.ts:39`, `GQE:125-139` |
| `ErrorInfo.metadata.quota_limit`          | string   | optional  | If it contains `PerMinute`, the error is treated as retryable with 60 s | `GQE:440-449` |
| `QuotaFailure.violations[]`               | `{subject?, description?, apiService?, quotaMetric?, quotaId?, quotaDimensions?, quotaValue?: string\|number, futureQuotaValue?: number}` | each optional | Which quota was violated. The client reads only `quotaId` | `googleErrors.ts:54-66`, `GQE:327-337`, `:427-438` |
| `RetryInfo.retryDelay`                    | string (duration, e.g. `"51820.6s"`, `"900ms"`) | required in the type | Server-suggested wait | `googleErrors.ts:43-46`, `GQE:98-113` |
| `TerminalQuotaError.retryDelayMs`         | number   | optional  | From `RetryInfo`. Used to render "Access resets at h:mm TZ"          | `GQE:28-51`, `UI/hooks/useQuotaAndFallback.ts:125-127`, `:153-163`, `:373-383` |
| `TerminalQuotaError.reason`               | string   | optional  | Copy of `ErrorInfo.reason`. `isInsufficientCredits` is true when it is `INSUFFICIENT_G1_CREDITS_BALANCE` | `GQE:30`, `:48-50` |
| `RetryableQuotaError.retryDelayMs`        | number   | optional  | Lower bound for the backoff delay                                    | `GQE:56-72`, `packages/core/src/utils/retry.ts:472-485` |

## Delivery mechanism

- **Poll: `retrieveUserQuota`.** The client calls it:
  - at startup, after the content generator is created, if a Code Assist project exists (`CFG:1621-1623`);
  - after every successful `generateContent` and `generateContentStream`, through `refreshUserQuotaIfStale()` with
    `staleMs = 30_000`. It runs fire-and-forget, and errors are only debug-logged
    (`packages/core/src/core/loggingContentGenerator.ts:421-423`, `:572-574`, `CFG:2392-2400`);
  - from `/stats` (session view), forced, together with `refreshAvailableCredits()`
    (`UI/commands/statsCommand.ts:65-79`);
  - from `/model manage`, forced (`UI/commands/modelCommand.ts:55-58`).

  On failure the client returns `undefined` silently (`CFG:2386-2389`). The cache resets when the session resets
  (`CFG:1854-1859`).
- **Internal push.** `coreEvents` emits `quota-changed` (`events.ts:219`, `:438-445`). The event fires when the
  pooled or primary `remaining`/`limit` changes, or when the model changes (`CFG:868-880`, `:1929-1937`). A
  `resetTime`-only change does not emit the event. The TUI subscribes in `UI/AppContainer.tsx:553-570`.
- **Surfaces (TUI):**
  - **Footer item `quota`**, on by default. It shows "N% used" or "limit reached", and the item's description is
    "Percentage of daily limit used (not shown when unavailable)" (`packages/cli/src/config/footerItems.ts:35-39`,
    `:91-98`, `UI/components/Footer.tsx:350-366`). When the item is configured but the footer is not forced to show
    it, `QuotaDisplay` hides until 80% is used. Its colors switch at warning 80% and critical 95%
    (`UI/components/QuotaDisplay.tsx:33-46`, `UI/utils/displayUtils.ts:22-23`).
  - **`/stats model`**: shows the tier and, **only in auto model mode**, the pooled values: "% used (Limit resets in
    Xh Ym)", "Usage limit: N", and the hard-coded "Usage limits span all sessions and reset daily." At 0 remaining it
    adds "Please /auth to upgrade or switch to an API key to continue."
    (`UI/components/ModelStatsDisplay.tsx:357-366`, `UI/components/QuotaStatsInfo.tsx:41-69`). `/stats model` reads
    the cache and does not refresh it (`statsCommand.ts:106-129`). `/stats` is also available as `/usage`
    (`statsCommand.ts:85-86`).
  - **`/stats` (session)**: fetches the quota and stores it in the history item (`quotas`, `pooled*`,
    `creditBalance`; `UI/types.ts:218-238`). However, `HistoryItemDisplay` passes only duration, auth, email, and
    tier to `StatsDisplay` (`UI/components/HistoryItemDisplay.tsx:155-161`). In this view the quota is not shown, and
    the user is told "Use /model to view model quota information" (`StatsDisplay.tsx:144-146`).
  - **`/model` dialog**: `ModelQuotaDisplay` renders the raw buckets, grouped by model *tier* (Pro, Flash, Flash
    Lite). For each group it keeps the bucket with the lowest `remainingFraction` and shows a progress bar, "% used",
    and "Resets: ...". It ignores `remainingAmount` (`UI/components/ModelDialog.tsx:383-386`,
    `UI/components/ModelQuotaDisplay.tsx:16-20`, `:117-186`).
- **Push: errors.** Quota errors arrive as the HTTP error of the generate call. In interactive mode, a
  `TerminalQuotaError` (or a `RetryableQuotaError` that has run out of attempts) calls `onPersistent429`. That runs
  `handleFallback`, which either switches silently or shows `ProQuotaDialog`
  (`packages/core/src/utils/retry.ts:397-418`, `:445-469`; `packages/core/src/fallback/handler.ts:26-138`). The
  message templates are "Usage limit reached for {all Pro models|model}. Access resets at {time}. /stats model for
  usage details / /model to switch models / /auth to switch to API key." and, for capacity errors, "We are currently
  experiencing high demand for ..." (`UI/hooks/useQuotaAndFallback.ts:149-172`). The possible user intents are
  `retry_always | retry_once | retry_with_credits | stop | retry_later | upgrade` (`packages/core/src/fallback/types.ts:17-23`).
- **Headless (`-p`, JSON/stream-json).** These modes have no quota fields. The result stats contain only token
  counts (`packages/core/src/output/types.ts:81-108`). The internal agent-event protocol maps HTTP 429 to the error
  status `RESOURCE_EXHAUSTED` (`packages/core/src/agent/event-translator.ts:384-385`), and stream-json reports such
  errors with `severity: 'error'` (`packages/cli/src/nonInteractiveCliAgentSession.ts:625-637`).
- **No header parsing.** A grep for `ratelimit`, `retry-after`, and `x-goog-quota` finds nothing relevant in
  `packages/core/src`. The only `Retry-After` hit is a log string at `retry.ts:555`.

## Windows and semantics

- **Window.** The bucket carries only `resetTime` and no window type or length. The client treats every bucket as a
  daily limit (`QuotaStatsInfo.tsx:61-62`, `footerItems.ts:38`). The docs give **requests per user per day**, by
  tier: Code Assist Individual 1,000; Google AI Pro 1,500; Ultra 2,000; Standard 1,500; Enterprise 2,000; API-key
  free tier 250, Flash only. The docs add that "Requests are limited per user per minute"
  (`docs/resources/quota-and-pricing.md:17-39`).
- **Per minute.** This limit is visible only through errors: `quotaId` contains `PerMinute`, or
  `ErrorInfo.metadata.quota_limit` contains it. The client then retries after 60 s (`GQE:427-449`).
- **Per model.** There is one bucket per `modelId`. In auto mode the UI sums the Pro and Flash buckets into one
  "pooled" meter. In the `/model` dialog it groups buckets by family and shows the worst one. In the fallback message
  "all Pro models" is used because Pro limits are shared (`useQuotaAndFallback.ts:107-109`). Whether the server
  shares quota across Pro models is UNVERIFIED; the client only infers it from this message.
- **How errors are classified** (`GQE:205-463`):
  - `limit: 0` appears in the message → terminal (`:255-267`).
  - `quotaId` contains `PerDay` or `Daily` → terminal: "You have exhausted your daily quota on this model." (`:326-337`).
  - Reason `MODEL_CAPACITY_*` → terminal. The retry loop then makes up to 3 silent retries in interactive mode, or
    retries indefinitely up to max attempts in headless mode, and only then falls back (`:347-359`,
    `retry.ts:351-395`).
  - Reason `INSUFFICIENT_G1_CREDITS_BALANCE` → terminal (`:361-369`).
  - Code Assist domain with `RATE_LIMIT_EXCEEDED`: retryable if `retryDelay` is 300 s or less, otherwise terminal.
    Without a delay it is terminal (`:371-397`). `QUOTA_EXHAUSTED` → terminal (`:398-405`).
  - Any `RetryInfo` delay: retryable up to 300 s, terminal above that (`:410-424`, `MAX_RETRYABLE_DELAY_SECONDS`
    at `:120`).
  - A message matching "Please retry in Xs" is handled the same way (`:276-290`).
  - Anything else on 429/499/503 → retryable (`:460-462`).
- **Retry.** Default policy: 10 attempts, 5 s initial delay, 30 s maximum, ±30% jitter. A server-provided delay sets
  the minimum, plus 0–20% jitter (`retry.ts:20`, `:42-47`, `:472-497`).
- **Availability state.** In the availability service a terminal failure marks the model `terminal` with reason
  `quota` or `capacity`. A capacity mark expires after 30 s; a quota mark lasts until the session resets. A
  transient failure in auto mode marks the model `sticky_retry`, retried once per turn
  (`packages/core/src/availability/modelAvailabilityService.ts:9-30`, `:52-68`;
  `packages/core/src/availability/policyHelpers.ts:343-355`; `packages/core/src/availability/policyCatalog.ts:41-69`).
- **Fallback chain.** The chain is Pro → Flash, and Flash is the last resort. The default action for every failure
  kind is `prompt`; in auto mode transient failures are `silent` (`policyCatalog.ts:41-123`).
- **Per auth type.**
  - OAuth (free, Standard, Enterprise, AI Pro/Ultra): gets buckets, tier, and credits for G1 subscribers.
  - `COMPUTE_ADC` also uses Code Assist (`CA/codeAssist.ts:21-37`).
  - API key and Vertex: `getCodeAssistServer()` returns undefined, so there are no buckets, no tier, no credits, and
    no footer meter (`CA/codeAssist.ts:42-63`, `CFG:2313-2317`). The errors and fallback still apply. 429 text adds
    "request a quota increase through AI Studio / Vertex" (`packages/core/src/utils/errorParsing.ts:12-33`).
  - OAuth and ADC users without `GOOGLE_CLOUD_PROJECT` get a "shared Google Cloud project" tip appended to quota
    errors (`retry.ts:215-249`).
- **Overage.** Overage here means spending Google One AI credits after the quota is exhausted. The flow is controlled
  by `overageStrategy` and the credits balance. Credits are a single scalar balance, not a window.
- **Relation to tokens and context.** These are fully separate:
  - Token usage per model (`input, prompt, candidates, total, cached, thoughts, tool`) is accumulated from
    `usageMetadata` (`packages/core/src/telemetry/uiTelemetry.ts:54-70`).
  - Context occupancy is `lastPromptTokenCount / tokenLimit(model)`, with a default of 1,048,576
    (`UI/components/ContextUsageDisplay.tsx:17-36`, `packages/core/src/core/tokenLimits.ts:20-23`). Its footer item
    is `context-used`, separate from `quota`.
  - A context-window overflow is also reported with the `RESOURCE_EXHAUSTED` status in the agent-event protocol.
    That status is the same one used for 429, but it means something different (`event-translator.ts:185-193`).

## Gaps and unverified items

- No public documentation for `retrieveUserQuota` was found. It is a `v1internal` Cloud Code Private API, and the
  types reference an internal google3 proto (`CA/types.ts:79-80`). The semantics of `tokenType`, the units of
  `remainingAmount` (requests or tokens), and whether more than one bucket can exist per model (for example, one per
  `tokenType`) are UNVERIFIED.
- Whether the server ever sends `remainingAmount`, or only `remainingFraction`, is UNVERIFIED. The code handles both
  cases (`CFG:2352-2362`).
- The client assumes the window is daily. The real window length per bucket is not on the wire.
- `/stats` session view: the quota data is fetched but not rendered (`HistoryItemDisplay.tsx:155-161`). This looks
  like a regression, or it is intentional, given the "Use /model" hint.
- Event `quota-changed` does not fire for a `resetTime`-only change (`CFG:868-880`).
- Gemini API (API key) and Vertex 429 responses may carry `QuotaFailure` details with `quotaId`, `quotaValue`, and
  similar fields (`googleErrors.ts:54-66`). Apart from `quotaId`, the client does not surface them. Their exact
  content is UNVERIFIED here.
- Not examined: ACP mapping of any of the above (see `gemini-acp.md`), IDE companion extensions, and Code Assist
  admin controls (`fetchAdminControls`).

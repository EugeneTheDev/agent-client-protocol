# Gemini CLI: can the agent tell which quota pool is being charged right now?

Scope: whether Gemini CLI can reliably set a per-pool `active` flag, meaning "current usage is charged against this
pool now". This note builds on `gemini-native.md` (buckets, polling, errors), `gemini-overage.md` (credits flow), and
`gemini-acp.md` (ACP boundary) and does not repeat them.

Source: https://github.com/google-gemini/gemini-cli at commit `fb972b2f87fe7d5b06d37eac711490162d98de2c`
(2026-10-02), the same checkout as the other Gemini notes. Paths are relative to the repo root. Permalink form:
`https://github.com/google-gemini/gemini-cli/blob/fb972b2f87fe7d5b06d37eac711490162d98de2c/<path>#L<line>`.

Abbreviations: `CA` = `packages/core/src/code_assist`, `CFG` = `packages/core/src/config/config.ts`,
`GC` = `packages/core/src/core/geminiChat.ts`, `CL` = `packages/core/src/core/client.ts`,
`FH` = `packages/core/src/fallback/handler.ts`, `B` = `packages/core/src/billing/billing.ts`,
`ACPS` = `packages/cli/src/acp/acpSession.ts`, `LCG` = `packages/core/src/core/loggingContentGenerator.ts`.

## Summary

- **Included buckets: the agent knows the concrete model of every request before sending it.** The model is not
  read from the response. In auto mode the router picks a concrete model for each prompt (`CL:779-812`). In
  `GC.apiCall`, every attempt calls `resolveModel(...)` and passes the concrete id as `req.model`
  (`GC:917-954`, `:1082-1090`). That id goes on the wire as `CAGenerateContentRequest.model` (`CA/converter.ts:135-136`).
  After a silent fallback, the next attempt reads the new `getActiveModel()` (`GC:934-943`), so the per-attempt model
  is always correct inside core. The "pooled" Pro+Flash meter in auto mode is a display construct only
  (`CFG:2056-2101`). It is not a server pool, and the router still sends each request to one model.
- **Mapping a model to a bucket is a client-side string match.** Bucket `modelId` values are normalized for Flash
  and Flash-Lite aliases (`CFG:2328-2347`), and the meter is looked up as `modelQuotas.get(resolvedModel)`
  (`CFG:2103-2117`). No wire field says "this request was charged to bucket X". Whether the server shares one quota
  across several Pro models is UNVERIFIED (see `gemini-native.md`).
- **More than one bucket is charged inside one prompt.** Utility calls run on other models: the routing classifier,
  summarizers, and edit-corrector use `flash-lite`; loop detection, next-speaker, and web-search use Flash
  (`packages/core/src/config/defaultModelConfigs.ts:173-176`, `:211-299`). So "the active pool" is really "the pool of
  the main chat model". At any moment several buckets can be charged.
- **Credits: the client cannot know before or during a request whether credits will be spent.** With `always`, the
  client adds `enabled_credit_types` to every eligible streaming request, whether or not the included bucket still
  has quota left (`CA/server.ts:99-125`). The server decides which pool to charge. The only per-response signal is
  `consumedCredits` (optional) in the response. The client reads it only to sum it for telemetry after the stream
  ends (`CA/server.ts:136-198`). Nothing in the wire type or the client says whether the field is sent on every
  response, only when credits were spent, or only when the request opted in. That is UNVERIFIED.
- **ACP loses the per-request model.** `_meta.quota.model_usage[].model` and `usage_update.size` take the model from
  the `ModelInfo` event. That event fires once per prompt sequence, before the fallback retry loop, and is not
  re-emitted on tool-loop iterations. A silent in-request fallback or an auto-mode continuation is therefore
  attributed to the wrong id (Pro instead of Flash), or to the alias `auto` (`ACPS:427`, `:493-496`, `:576-584`;
  `CL:809-812`). The core has the correct value. The ACP adapter does not use it.

## Signals

| Signal | Meaning | When absent | Source |
| --- | --- | --- | --- |
| `req.model` passed to `generateContentStream` / `generateContent` | Concrete model id for this attempt, after alias resolution, availability selection, fallback, and `BeforeModel` hook overrides | Never absent for a real request. A hook may block the call, and then nothing is sent | `GC:917-954`, `:1005-1029`, `:1082-1090`; `CA/converter.ts:135-136` |
| `GeminiEventType.ModelInfo` | Model picked by the router at the start of a prompt sequence | Not re-emitted while `currentSequenceModel` is set: tool-loop continuations with the same `prompt_id`, and in-request fallback retries. It resets on `ModelChanged` or a new `prompt_id` | `CL:786-812`, `:135-137`, `:935-942` |
| `Config.getActiveModel()` / `getModel()` | Configured or fallback model. `activateFallbackMode` calls `setModel(..., true)`, which emits `ModelChanged` and `quota-changed` | Always set. In auto mode `getModel()` is the alias (`auto`), not a concrete model | `CFG:1929-1951`, `:1982-1984`; `packages/core/src/config/models.ts:642-654` |
| `response.modelVersion` | Model version the server reports. Used for logging and per-model UI stats as `modelVersion \|\| req.model` | Optional (`CA/converter.ts:89`, `:159`). Whether its format matches bucket `modelId` is UNVERIFIED | `LCG:405`, `:554`, `:590` |
| `BucketInfo.modelId` + `remainingFraction` (poll) | Per-model included pool. A fall in `remaining` between polls is indirect evidence of charging | Absent for API key and Vertex users. The poll runs at most every 30 s after a response, fire-and-forget | `CFG:2313-2390`, `:2392-2400`; `LCG:421-423`, `:572-574` |
| `quota-changed` event | Pooled or primary meter changed, or the model changed | Not fired for a change to `resetTime` alone | `CFG:868-880`, `:1938-1940` |
| `enabled_credit_types: ['GOOGLE_ONE_AI']` on the request | Client opted in to credits for this request. This is intent, not consumption | Not sent when the strategy is not `always`, the balance is below 50, the model is not in `OVERAGE_ELIGIBLE_MODELS`, or the call is non-streaming | `CA/server.ts:99-125`, `:205-216`; `B:33-37`, `:137`, `:145-154` |
| "Using AI Credits for this request." (feedback `info`) | Shown once per user prompt when the opt-in is first sent | Shown in the TUI only. Not forwarded over ACP. It fires before the response, so it reports intent | `CA/server.ts:108-111`; `gemini-overage.md` |
| `CaGenerateContentResponse.consumedCredits[]` | Credits spent by this response or chunk | Optional. The client accepts absent, `null`, and empty values (`CA/server.test.ts:1036-1044`). The server's rule for when it sends the field is UNVERIFIED | `CA/converter.ts:80`; `CA/server.ts:163-169` |
| `CaGenerateContentResponse.remainingCredits[]` | Balance after this response. A drop is a second indicator of consumption | Optional. Also applied on non-streaming calls | `CA/server.ts:170-179`, `:234-236`, `:241-255` |
| Telemetry `credits_used {model, credits_consumed, credits_remaining}` | Emitted after the stream completes when the summed `consumedCredits` > 0 | Not emitted when the sum is 0 or the field is absent. `model` is `req.model` | `CA/server.ts:184-194`; `packages/core/src/telemetry/billingEvents.ts:25-201` |
| `TerminalQuotaError` with `INSUFFICIENT_G1_CREDITS_BALANCE` | The server refused to charge credits | Only on errors | `packages/core/src/utils/googleQuotaErrors.ts:361-369` |

## Deduction rules

An agent that wants to fill `active` with what this client knows can apply these rules:

1. **Main-chat included bucket.** On each attempt, `active` = the bucket whose (normalized) `modelId` equals
   `req.model` of the latest main-chat (`LlmRole.MAIN`) request. Use the per-attempt model from `GC.apiCall`, not
   `ModelInfo` or `getModel()`. After a fallback, the next attempt's `req.model` already names the fallback model
   (`GC:934-954`).
2. **Auto mode.** Do not mark the synthetic pooled Pro+Flash meter as one pool. Mark the bucket of the concrete model
   the router chose for this request. If the RFD wants an auto-mode aggregate, it is a derived view, not a pool.
3. **Utility calls.** The classifier and helper calls charge other buckets (`flash-lite`, Flash) during the same
   prompt. Either (a) mark only the main-chat pool `active`, or (b) let several pools be `active` at once. Gemini
   cannot give a single answer.
4. **Credits, before or during a request.** If the request carried `enabled_credit_types`, credits are *possibly*
   charged. The client does not know whether the server charged the included bucket or credits. Antigravity's
   public docs for the same credit product say credits are used only once the baseline quota for a model is
   exhausted, and that billing "automatically switches back to using the baseline quota once it refreshes"
   (https://antigravity.google/docs/plans/). That this rule holds for Gemini CLI's Code Assist path is UNVERIFIED.
5. **Credits, after a response.** If summed `consumedCredits` for `GOOGLE_ONE_AI` > 0, or `remainingCredits` fell
   compared with the cached balance, credits were spent on that response. Mark the credits pool `active` for that
   response and leave the model's included bucket inactive. Absent or 0 is weak evidence that no credits were spent.
   It is not proof, because the rule for sending the field is unknown.
6. **Included bucket exhausted, no credits opt-in.** After a `TerminalQuotaError` with no fallback taken, nothing is
   being charged. No pool is `active` until the next request.

## Failure modes

- **Before the first response.**
  - The model is known before the request is sent (`req.model`).
  - In auto mode the concrete model is known only after routing, which happens per prompt and itself spends a
    `flash-lite` classifier call (`packages/core/src/routing/strategies/classifierStrategy.ts:149-172`). Function
    responses bypass the classifier (`:149`).
  - Bucket data exists only after the startup poll (`CFG:1621-1623`), and only for Code Assist auth.
  - Whether credits will be spent is unknown until the first response that carries `consumedCredits`, if the server
    ever sends one.
- **Lag in credits.** `consumedCredits` is summed over the whole stream and logged only after the stream completes
  (`CA/server.ts:136-198`). Which chunk carries it on the real wire is UNVERIFIED. The tests put it on a metadata
  chunk next to `usageMetadata` (`CA/server.test.ts:1088-1096`), but those are mocks. Bucket `remaining` lags by up to
  30 s plus the poll round trip (`CFG:2392-2400`). The UI copy warns that "Newly purchased AI credits may take a few
  minutes to update" (`packages/cli/src/ui/components/EmptyWalletDialog.tsx`, see `gemini-overage.md`).
- **After a model switch.**
  - `setModel` (from `/model`, or `activateFallbackMode` on a `retry_always` fallback) resets `currentSequenceModel`
    through `ModelChanged`, re-emits `quota-changed`, and the next request routes again (`CFG:1938-1951`,
    `CL:135-137`, `FH:164-168`).
  - `retry_once` does not change the active model (`FH:171-175`). The retried model comes from availability state,
    so only `req.model` reveals it.
  - Inside one `sendMessageStream`, a silent fallback changes `req.model` without a new `ModelInfo` (`CL:809-812`),
    so ACP mislabels the tokens (see Summary).
  - A `BeforeModel` hook can also rewrite the model (`GC:1018-1029`).
- **API-key and Vertex users.** There is no Code Assist server, so there are no buckets, no credits, and no tier
  (`CA/codeAssist.ts:42-63`; `CFG:2313-2317`). The model per request is still known, but there is no pool to mark.
  At most the agent can describe one pay-as-you-go pool with no meters, and nothing on the wire supports that.
- **`ask` strategy pending consent.**
  - The failing request has already ended with a `TerminalQuotaError`, and the TUI shows the overage menu
    (`gemini-overage.md`). Nothing is in flight, so no pool is `active`.
  - After "Use AI Credits" the strategy becomes `always` until the next user prompt, and the retry carries the
    opt-in. From then on rule 5 applies.
  - In ACP no fallback handler is registered, so the error goes straight to the client (`FH:112-115`;
    `gemini-acp.md`). With `ask`, credits are never used in ACP. With `always` in settings they may be used, and
    the user is not told.
- **Retries in flight.** During silent `RetryableQuotaError` backoff (`packages/core/src/utils/retry.ts:440-470`) no
  request is being served. Whether rejected attempts count against a bucket is UNVERIFIED.
- **Credits charged while the bucket still has quota.** A user report says credits were deducted while the dashboard
  still showed 20% of the included quota left
  (https://discuss.ai.google.dev/t/google-deducted-my-ai-credits-despite-i-still-having-the-gemini-pro-quota/135887).
  This is anecdotal and concerns Antigravity, but it shows that rule 4 cannot be trusted. Only rule 5 (post hoc) is
  evidence.
- **Product context.** The Gemini CLI docs banner says that, for unpaid-tier and Google One users, "Gemini CLI was
  replaced by Antigravity CLI on June 18th, 2026" (https://geminicli.com/docs/resources/quota-and-pricing/). The
  credits code is still in this commit, but in practice the Google One credits path may now be reachable only for
  some account types. Which ones is UNVERIFIED.

## Verdict

| Pool kind | Verdict | Why |
| --- | --- | --- |
| Included per-model buckets (main chat) | **Reliable with caveats** | The concrete model of every attempt is known in core before the request is sent, including after silent fallback and in auto mode. Caveats: (1) mapping a model to a bucket is a client-side string match with alias normalization, and no wire field confirms the charged bucket; (2) server-side sharing across Pro models is unverified; (3) utility calls charge other buckets at the same time; (4) the ACP adapter currently attributes by `ModelInfo`, which is stale after in-request fallback and tool-loop continuations; (5) none of this exists for API key or Vertex users |
| AI credits pool | **Heuristic only** in real time. **Reliable with caveats** after the fact, as a positive signal | Opting in (`enabled_credit_types`) does not mean credits are spent, and the server decides which pool to charge. `consumedCredits > 0` on a response is a good positive signal once the stream ends. Its absence or 0 is ambiguous, because the rule for sending the field is undocumented. There is no signal before the response, and none for non-streaming calls, which never opt in |

Implication for the RFD: `active` should be optional and allow several pools at once, and the RFD should say whether
`active` describes the last completed request (post hoc) or the next request (predictive). Gemini can fill the
post-hoc meaning for included buckets well and for credits partially. It can fill the predictive meaning only for
included buckets.

## Gaps

- UNVERIFIED: when the Code Assist server sends `consumedCredits`: on every response, only when credits were spent,
  or only when `enabled_credit_types` was set. Also which stream chunk carries it. There are no public docs and no
  live capture.
- UNVERIFIED: whether the server charges credits only after the included bucket is exhausted. The docs that say so
  are for Antigravity (https://antigravity.google/docs/plans/), not Gemini CLI.
- UNVERIFIED: whether `response.modelVersion` uses the same id format as `BucketInfo.modelId` and the request `model`.
  If it does, it could confirm a server-side model substitution.
- UNVERIFIED: whether the server ever serves a request on a different model than `req.model`. No client code handles
  that case.
- UNVERIFIED: whether quota-rejected (429) attempts are counted against the bucket.
- The local clone is shallow, so the history of the credits fields could not be inspected.

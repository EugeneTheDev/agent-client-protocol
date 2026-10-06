# Codex: can an adapter tell which quota pool is being charged right now?

Scope: whether a Codex adapter can reliably set a per-pool `active` flag ("current usage is charged against this pool
now"). This note builds on `codex-native.md`, `codex-overage.md` and `codex-acp-adapter.md` and does not repeat them.

Source: `github.com/openai/codex` at commit `823ea830c0fd418b09ff02d36cad9a1fff66465b` (the same commit as the other
notes, a shallow clone). Paths are relative to `codex-rs/`. Public docs are cited by URL.

## Summary

- **No success-path signal names the charged bucket.** Codex has exactly one explicit "this bucket" signal, the
  `x-codex-active-limit` header. Codex reads it only on a 429 `usage_limit_reached`, where it names the bucket that
  blocked the request (`codex-api/src/api_bridge.rs:181-196, 275`). On successful responses, Codex receives only
  per-bucket snapshots:
  - SSE: header families `x-<family>-*` (`codex-api/src/rate_limits.rs:28-51`);
  - WebSocket: `codex.rate_limits` events labeled by `metered_limit_name` (`codex-api/src/rate_limits.rs:124-169`).

  Neither says "this request was billed here". It is UNVERIFIED whether the backend sends only the charged family or
  every family.
- **Codex itself maps model to bucket by name.** The TUI treats the bucket whose `limit_name` equals the current model
  slug, plus the always-relevant `codex` bucket, as the ones that matter (`tui/src/chatwidget/rate_limits.rs:195-201`).
  `ModelInfo` carries no quota field. The status line always shows the `codex` bucket, whatever the model
  (`tui/src/chatwidget/status_surfaces.rs:760-775`).
- **Both windows of a bucket seem to count at once.** Each snapshot carries `primary` and `secondary` together, and
  every consumer checks both. The 429 names the "server-selected window responsible for the limit"
  (`protocol/src/error.rs:685-686`). This is inferred. No contract or doc says it.
- **More than one bucket can be active at once.** An image-generation tool call is metered against a separate
  `image_gen` bucket during a model turn (`ext/image-generation/src/tool.rs:261-286`). The pricing page says image
  generation "also draws from credits" after included limits.
- **Credits have no per-response consumption signal.**
  - Responses carry the credits *state* (`has_credits`, `unlimited`, `balance`), not a charge.
  - The only authoritative input is `ordinaryUsageAllowed`. It is poll-only, it reflects only the `codex` bucket, and
    it is nulled unless the account identity matches.
  - Codex's own TUI says that "percentages, sparse notifications and reset timestamps cannot prove" recovery
    (`tui/src/chatwidget/backend_banners.rs:315-316`).
- **Verdict.**
  - Included buckets: heuristic only, except on a 429, which is exact.
  - Credits: heuristic. An adapter that polls gets "reliable with caveats". The current ACP adapter polls only on
    `/status`, so it gets heuristic only.

## Signals

| Signal | Meaning | When absent | Source |
|---|---|---|---|
| `x-codex-active-limit` header (429 only) | Metered `limit_id` that blocked this request. Codex parses that family's headers into the error's snapshot | Absent on success responses, or never read there. When absent on a 429, Codex falls back to `codex` | `codex-api/src/api_bridge.rs:188-201, 275`; tests `codex-api/src/api_bridge_tests.rs:643-700`; `app-server/tests/suite/v2/imagegen_extension.rs:594` (`image_gen`) |
| 429 body `error.limit_window_minutes` | "Server-selected window responsible for the limit", in minutes. Says which window (not which bucket) blocked | Optional in the body | `protocol/src/error.rs:685-686`; `codex-api/src/api_bridge.rs:208-214` |
| 429 `x-codex-rate-limit-reached-type` | Reason: base limit, credits depleted, or spend cap. Codex copies it onto the snapshot | Optional | `codex-api/src/api_bridge.rs:190-201`; `codex-api/src/rate_limits.rs:188-193` |
| SSE header families `x-<family>-primary-used-percent` etc. | One snapshot per family found on the response. `codex` is first, then the other families in sorted order | `codex` is **always synthesized**, even with no headers at all (all-null windows). Other families are kept only if they carry data | `codex-api/src/rate_limits.rs:28-51, 259-261`; `codex-api/src/sse/responses.rs:43, 78-80`; test `rate_limits.rs:370-381` |
| `x-<family>-limit-name` | Display name of the family, typically a model slug (test: `codex_bengalfox` → `gpt-5.2-codex-sonic`) | Optional. Never back-filled from `limit_id` | `codex-api/src/rate_limits.rs:84-88, 332-348`; `api_bridge_tests.rs:609-641` |
| WS event `codex.rate_limits.metered_limit_name` (or `limit_name`) | `limit_id` of this one snapshot. One event is one bucket | Defaults to `codex` when absent. The WS handshake headers are **not** parsed for rate limits | `codex-api/src/rate_limits.rs:124-169`; `codex-api/src/endpoint/responses_websocket.rs:527-546, 775-779` |
| WS event `rate_limits.allowed` / `limit_reached` | Appear in the test fixture, with the same shape as the usage endpoint. A per-response "ordinary usage allowed" would come from here | **Dropped** by Codex's parser (`RateLimitEventDetails` has only `primary` and `secondary`). Production presence UNVERIFIED | `core/tests/suite/client_websockets.rs:1660-1680`; `codex-api/src/rate_limits.rs:111-115` |
| `openai-model` header / WS response model → `ModelReroute` event | The model the server actually ran, which can differ from the requested one | Absent when the server does not report it | `codex-api/src/sse/responses.rs:49-53, 73-77`; `responses_websocket.rs:781-788`; `core/src/session/mod.rs:3922-3949` |
| `limit_name == current model` rule | Codex's own model-to-bucket rule. With no match, only `codex` applies | No bucket list until a poll. The rule needs `rateLimitsByLimitId` | `tui/src/chatwidget/rate_limits.rs:195-201` |
| `normalModelSlug` | Maps a quota-alias bucket (a reserve model) to the normal model's metadata. "Never a replacement for the request model" | Poll-only, alias buckets only | `protocol/src/protocol.rs:2352-2354`; `backend-client/src/client/rate_limit_resets.rs:46-56` |
| `account/rateLimits/updated` (push) | One snapshot. After core's single-slot merge, it is the **last** family parsed from the latest response | No `threadId`. Pushes reuse the cached snapshot when a response had no new data | `core/src/state/session.rs:75, 327-332`; `core/src/session/turn.rs:2957-2962`; `app-server/src/bespoke_event_handling.rs:1579-1605` |
| `ordinaryUsageAllowed` (poll) | Backend `rate_limit.allowed` for the **`codex` bucket only**. False means included usage is blocked | Null for API-key auth (read rejected), when the backend omits it, or when `account_id`/`user_id` do not match the active auth | `backend-client/src/client/rate_limit_resets.rs:38-44`; `app-server/src/request_processors/account_processor.rs:1270-1285` |
| `credits.{hasCredits, unlimited, balance}` (headers, WS, poll) | Credit pool state. Not prefixed by family, so every header family carries the same values | `balance` can be null (hidden). Absent when the headers are missing | `codex-api/src/rate_limits.rs:219-231`; `backend-client/src/client.rs:621-653` |
| `rawResponse/completed.usageMetadata.amount` | Opaque decimal string per upstream response ("exact upstream usage"). Unit and pool are unspecified | Optional. The notification is marked "internal-only" | `app-server-protocol/src/protocol/common.rs:1977-1979`; `.../v2/thread.rs:1903-1919`; `protocol/src/response_usage.rs:9-14`; `codex-api/src/sse/responses.rs:106-114, 436-454` |
| API-key turn cost (`ApiKeyTurnCost.total_usd`) | USD per turn, polled from the backend every 150 s | Only with API-key auth and an OTLP exporter. Telemetry only, never sent to clients | `app-server/src/turn_cost_worker.rs:25, 102-110, 470-499`; `backend-client/src/client/turn_usage.rs:24-33` |

## Deduction rules

These are the rules an adapter can apply, from strongest to weakest.

1. **On a 429 `usage_limit_reached`, the blocked bucket is exact.**
   - `UsageLimitReached.rate_limits.limit_id` comes from `x-codex-active-limit`. Mark that pool exhausted.
   - `limit_window_minutes` tells which window blocked.
   - `rateLimitReachedType` tells whether the cause was the included allowance or the credits or spend cap.
   - This describes the *blocked* pool, not the pool that will be charged next. Source: `api_bridge.rs:188-214`;
     `turn.rs:1720-1726`.
2. **Model to included bucket by name (what Codex itself does).**
   - Take the server-reported model, which follows `ModelReroute`, and otherwise the requested model.
   - If `rateLimitsByLimitId` has a bucket whose `limitName` equals that slug, that bucket is the model's bucket.
     Otherwise the model draws on `codex`.
   - It is UNVERIFIED whether a model-specific request is also charged to `codex`. The TUI watches both
     (`rate_limits.rs:195-196`). The reserve comment "Reserve is a fallback for exhausted ordinary usage"
     (`protocol/src/error.rs:694-695`) suggests the reserve bucket is used *instead of* `codex`.
3. **Per-response family hint (weak).**
   - A non-`codex` family on a response, or a WS event with a non-`codex` `metered_limit_name`, suggests that this
     request touched that bucket.
   - This is valid only if the backend sends model-specific families only for that model's requests. That is UNVERIFIED,
     because `parse_all_rate_limits` takes whatever arrives.
   - A `codex` snapshot proves nothing, because it is synthesized even when no headers arrive.
4. **Both windows of the active bucket are active together.** This is inferred:
   - The two windows always arrive in one snapshot.
   - The memories guard requires both to be under the threshold (`memories/write/src/guard.rs:51-59`).
   - The 429 names one responsible window.
   - The docs say "Weekly limits may also apply" (https://learn.chatgpt.com/docs/pricing).
5. **Tool-metered buckets are concurrent.** `image_gen` is charged by the image tool alongside the model bucket. So
   `active` has to allow several pools at once (`ext/image-generation/src/tool.rs:261-286`).
6. **Credits active, inferred:**
   - the condition is `ordinaryUsageAllowed == false && (hasCredits || unlimited) && spendControlReached != true &&
     rateLimitReachedType == null`;
   - it is valid only for the `codex` bucket's included allowance;
   - it holds only as of the last poll.

   This is the negation of the TUI's own recovery predicate (`backend_banners.rs:322-327`). The overage note gives
   the rationale.
7. **Credits active, heuristic only.** A bucket window at 100% with usable credits, or a falling `balance` between
   responses. Codex states that percentages and reset times cannot prove included-usage state
   (`backend_banners.rs:315-316`; v2 doc on `ordinaryUsageAllowed`, `app-server-protocol/src/protocol/v2/account.rs:331-334`).
   A falling balance has further problems:
   - `balance` can be hidden, and it means nothing when `unlimited` is set;
   - no code compares balances;
   - the timing of balance updates is UNVERIFIED;
   - with a shared workspace pool, other members' spending could also lower it (UNVERIFIED).

## Failure modes

- **Before the first response.**
  - There is no push data. Only `account/rateLimits/read` can name the buckets and give `ordinaryUsageAllowed`.
  - The TUI prefetches at startup (`tui/src/app_event.rs:218-219`).
  - The ACP adapter polls only on `/status`. Its rate-limit cache stays empty until a push arrives
    (`codex-acp-adapter.md`).
  - Rules 2 and 6 cannot run until a poll completes. Rule 2 can then mark the model's bucket active before any request.
- **After a model switch.**
  - The last push still reflects the previous model's last family.
  - Rule 2 re-evaluates immediately, but only against the cached bucket list. "The new model has no dedicated bucket"
    and "the bucket list is stale or missing" cannot be told apart.
  - A server reroute (`ModelReroute`) changes the model, and so the bucket, without any user action.
  - Concurrent threads or subagents on different models share one account-wide push stream with no `threadId`, so
    pushes cannot be attributed to a session (`bespoke_event_handling.rs:1579-1605`; adapter broadcast in
    `codex-acp-adapter.md`).
- **Core keeps one snapshot slot.**
  - With several header families on one response, only the last family survives into `TokenCount` and the push
    (`session.rs:75, 327-332`; the deferral at `turn.rs:2957-2962`). The `codex` update from the same response is lost.
  - On SSE, a response with no headers overwrites the windows with null.
  - On WS, a response with no `codex.rate_limits` event leaves the old snapshot, which is then re-pushed as if fresh.
  - These are inferences from code, with no test.
- **After a window reset.**
  - Nothing clears the cached `usedPercent` when `resetsAt` passes. I found no such code in the TUI or the adapter.
  - `ordinaryUsageAllowed` stays false until the next poll, so rule 6 keeps reporting "on credits" after included
    usage is available again.
  - TUI staleness is bounded by the poll interval: 60 s, or 5 s once a watched window is at 99% or more
    (`rate_limits.rs:207-216`). That is about 5 s right after exhaustion.
  - In the current ACP adapter it is unbounded.
  - Reset-credit redemption triggers an explicit refresh (`tui/src/app_event.rs:227-228`).
- **Credits poll caveats.**
  - `ordinaryUsageAllowed` is null when the backend omits identity or the identity does not match the active login
    (`account_processor.rs:1270-1285`).
  - It covers only `codex`. A per-model bucket's own `allowed` is dropped, so rule 6 says nothing about credits behind
    per-model buckets. The overage note's UNVERIFIED claim is that credits do not cover them.
- **API-key users.**
  - `account/rateLimits/read` is rejected, so there is no bucket list, no `ordinaryUsageAllowed` and no credits.
  - Whether `x-codex-*` headers arrive is UNVERIFIED. Without them, pushes are a synthesized `codex` snapshot with all
    fields null (`codex-native.md`, Gaps).
  - Quota failures arrive as `QuotaExceeded` (`insufficient_quota` and related codes) with no bucket
    (`api_bridge.rs:221-233`).
  - Billing is "standard API rates" (https://learn.chatgpt.com/docs/pricing). There is no pool to mark `active`,
    except a single synthetic API-billing pool. The USD turn cost is telemetry-only.

## Verdict

- **Included buckets: heuristic only (exact only at a 429).**
  - Codex has no success-path field that says which bucket was charged. Codex's own clients map the model to a bucket
    by `limit_name == model slug` and otherwise assume `codex`.
  - For RFD purposes, "reliable with caveats" is reachable only if two things hold, both UNVERIFIED:
    - the backend sends model-specific families or WS events only for that model's requests;
    - per-model requests are not also charged to `codex`.
  - Marking both windows of the chosen bucket as active is a safe inference.
- **Credits: heuristic, or reliable with caveats when polled.**
  - There is no per-response credit-consumption signal.
  - An adapter that polls `account/rateLimits/read` can apply rule 6, with these caveats:
    - it is accurate only to the poll interval;
    - it holds for the `codex` bucket only;
    - it fails after a reset until the next poll;
    - it is unavailable for API keys and on identity mismatch.
  - Without polling, which is the current ACP adapter's behavior, it is heuristic only.

## Implications for the RFD

- `active` must be allowed on several pools at once: a model bucket, `image_gen`, and credits after exhaustion.
- `active` should be optional, with an explicit "unknown" state. The RFD should not imply that agents always know it.
- An "as of" timestamp, or a source (`response`, `poll`, `inferred`), on `active` would let clients judge staleness.
- A pool that blocked a request (exact on a 429) is a different fact from a pool being charged. A separate
  `exhausted` or `blocked` state carries the exact data without overloading `active`.

## Gaps

- UNVERIFIED: whether production responses carry only the charged bucket's family or WS event, or every bucket. No
  repo test sends `metered_limit_name`. The shallow clone and PR searches gave no rationale.
- UNVERIFIED: whether a request on a model with its own bucket also consumes `codex` windows.
- UNVERIFIED: whether production WS `codex.rate_limits` events carry `allowed` and `limit_reached`. If they do, Codex
  could get a per-response `ordinaryUsageAllowed`, but it drops them today.
- UNVERIFIED: the meaning and unit of `usageMetadata.amount`, and whether it is the credits charged for the response.
- Not checked: whether compaction, memories or guardian background requests draw on the same bucket as the turn model.
- Model names in some fields (reserve aliases) are product-specific. The RFD should not copy them.

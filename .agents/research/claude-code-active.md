# Claude Code: can an adapter know which quota pool is being charged "now"?

Scope: whether an ACP adapter built on the Claude Agent SDK can fill a per-pool `active` flag ("current usage is
charged against this pool now"). This note builds on `claude-code-native.md` (field inventory, units, cadence) and
`claude-code-overage.md` (overage state machine, disabled reasons) and does not repeat them.

Versions: `@anthropic-ai/claude-agent-sdk@0.3.289` (`/tmp/ccsdk/package/sdk.d.ts`) and the CLI binary
`~/.local/share/claude/versions/2.1.289`. `bin@N` is a byte offset into that Bun bundle. Bundle code is undocumented
internal behavior and can change in any release.

Doc source: ERR = https://code.claude.com/docs/en/errors (fetched 2026-10-06).

## Summary

- **No signal names "the pool this request was charged to".** The push path names the *binding* (most limiting)
  window (`rateLimitType`, from header `representative-claim`), not the charged one. The CLI's own schema says the
  top-level fields "describe the currently limiting window" (bin@180756636). The poll path's `limits[].is_active` is
  "the server's headline pick: the row a single-value indicator shows" (sdk.d.ts:6202-6205), a display choice, not
  billing.
- **Included pools are charged at the same time.** ERR: "Usage counts against the session and weekly allowances at
  the same time." The Opus and Sonnet limits "each apply only to requests to that model family" (ERR). So for one
  request, `active` is multi-valued: 5h + 7d, plus the model-family weekly window when the request's model has one.
  Whether a model-family request *also* counts toward session and weekly is not stated (ERR does not say; UNVERIFIED,
  though nothing suggests it is excluded).
- **Overage has two usable signals, both after the fact.** `isUsingOverage` (CLI-derived: the binding window is
  rejected and overage is allowed) and `overageInUse` (server header: this response was billed to credits). Each
  reflects the **last completed API response of any query source** (main loop, subagent, side call) and reaches the
  SDK only when the derived state changes. `overageInUse` is the better "now" signal but is dropped for usage-based
  billing orgs.
- **Verdict:** included pools are heuristic only (deduced from model plus docs), and overage is reliable with
  caveats (last-response granularity, absent before the first response, flaps under mixed-model concurrency).

## Signals

| signal | meaning | when absent | source |
|---|---|---|---|
| `SDKRateLimitInfo.isUsingOverage` | CLI-derived `status==='rejected' && overageStatus ∈ {allowed, allowed_warning}` from the **same response's** headers: the binding window is exhausted and credits carried this response | Always set by the parser (default `false`). The whole event is absent for API key, Bedrock, and Vertex sessions, and before the first response | `Lke` bin@187243849; sdk.d.ts:5655 |
| `SDKRateLimitInfo.overageInUse` | Server header `anthropic-ratelimit-unified-overage-in-use: true` on this response, meaning the response was billed to credits. Can be true while `status==='allowed'` (credit-only model or feature) | Absent when the header is absent or not `"true"` (only `true` is ever set, never `false`). **Stripped by the SDK serializer for orgs with `billingType==='usage_based'`** (`Srt` passes `includeOverageInUse: !tF()`; `tF` = `billingType==="usage_based"`). Gateway behavior UNVERIFIED | `Lke` bin@187243849; `ka` bin@194692500; `Srt` bin@205837506; `tF` bin@186172409 |
| `SDKRateLimitInfo.rateLimitType` | Binding window: header `representative-claim` when the status is `rejected` or the header status is `allowed` with no derived warning. Otherwise the window that triggered a CLI-derived `allowed_warning` (5h, 7d, 7d_oi, overage only) | Absent when the server omits `representative-claim`. Whether the server sends it on ordinary `allowed` responses is UNVERIFIED | `Lke`, `yvn`, `deriveTrackedLimits` bin@187250755 |
| `unifiedWindows` (runtime-only, not in `.d.ts`) | Per-window `{utilization, resetsAt}` for `five_hour`, `seven_day`, `seven_day_overage_included`, from the **latest** non-stale response only. "both windows are tracked on every observation" | A window the latest response did not carry is absent (no carry-over from earlier responses). Never carries `seven_day_opus` / `seven_day_sonnet` | `is` bin@205837506; `Dw` bin@187241611; zod description bin@180756636 |
| header `{5h,7d,7d_oi,overage}-utilization` | Per-window readings. The header-window table `Dw` has only these four keys. Opus and Sonnet windows appear on push **only** as a `representative-claim` value, never with their own utilization | Per-window: absent when the response does not carry it | `Dw` bin@187241611; `ivn` |
| `rateLimitGraceActive` (runtime-only) | A "grace zone" past a limit in which requests still run. "overageStatus allowed / allowed_warning means paid extra usage covers the overflow" | Only for requests that send `anthropic-usage-limit: extended` (experiment-gated). A refused 429 is not read, so the value persists while refused | zod description bin@180756636; `Mke` class |
| poll `limits[].is_active` | Server's headline row for a single-value indicator | `limits` null when the server predates rows or the fetch fails | sdk.d.ts:6202-6205 |
| poll `extra_usage.is_enabled` | "false while extra usage cannot cover sends". This is availability, not "in use" | `extra_usage` absent when the plan has none or the fetch failed | sdk.d.ts:6207-6209 |

## Deduction rules

What an adapter can compute from the last `rate_limit_event` (`info`) and the model of the last request.

1. **Overage active (credits are paying now):** `info.overageInUse === true || info.isUsingOverage === true`.
   - Use `overageInUse` as primary. It is the server's billing fact and covers both causes: an exhausted window, and a
     credit-only model or feature.
   - Fall back to `isUsingOverage` when `overageInUse` is suppressed (usage-based billing orgs). That fallback misses
     credit-only requests made while plan windows still have room.
   - The CLI itself treats either one as sufficient. Its grace logic marks a zone "covered" when `overage-status` is
     allowed or `overage-in-use === "true"` (`qwn`, bin@187230000).
2. **Overage scope:** when `isUsingOverage && rateLimitType === 'seven_day_overage_included'`, credits pay only for
   the overage-included model family. The CLI's allowlist defaults to `Fable`, `Fable 5`, `Fable 5.1` and can be
   overridden by a remote flag (`Ure`). Other models still draw on the plan. The CLI branches on exactly this case
   before latching "Fable credits required" (bin@187255960). So "overage active" is model-conditional, not global.
3. **Included pools active:** these have no wire signal and must be deduced.
   - If overage is not active, mark `five_hour` and `seven_day` active.
   - Also mark the model-family weekly window active when the current model belongs to one: `seven_day_opus` for
     Opus, `seven_day_sonnet` for Sonnet, and `seven_day_overage_included` for an allowlisted family.
   - The model-to-window mapping is the adapter's own knowledge. Push carries no model id per window. Poll carries
     only display labels (`model_scoped[].display_name`, `limits[].scope.model.display_name`).
   - Caveat on `seven_day_sonnet`: on Pro and Enterprise the CLI labels it "weekly limit" rather than "Sonnet limit"
     (`rDt`, bin@187230000), which suggests the plan may fold it into the general weekly window. UNVERIFIED.
4. **Included pools while on overage:** whether plan windows keep accumulating while credits pay is not stated.
   Utilization above 1 "occur[s] when usage legitimately runs past a window's cap" (bin@180756636), which suggests
   windows keep counting in some modes. UNVERIFIED. The RFD should allow an included pool and an overage pool to
   both be `active`, or should leave this unspecified.
5. **Binding vs. active:** never map `rateLimitType` to `active`. It names the window most likely to stop you, which is
   at most a "limiting" hint, a different attribute from `active`.

## Failure modes

| situation | effect on a deduced `active` | source |
|---|---|---|
| Before the first API response of a session | No data at all. The startup quota probe (a 1-token `"quota"` request, `pvn`) returns early in non-interactive mode (`Ce()`), so SDK hosts, the ACP adapter included, get nothing until the first real request completes. It is also skipped in essential-traffic mode (`Tt()`) and under `--bare`. The first request's pool is therefore always unknown in advance | `probeQuotaStatus` bin@187253410; startup bin@196565669 |
| Granularity is per API response, after completion | Headers are applied when a request's stream completes (`if(zl)…$ke(zl.headers…)` bin@189378611), so `active` describes the **previous** request. A transition into overage is learned from the first response that was already billed to credits | bin@189378611 |
| Emission only on change | `rate_limit_event` fires from `statusChanged`, after a deep compare, or when a 5h/7d/7d_oi rounded percentage or reset moves. Unchanged state produces no event, so the adapter must hold the last value | `extractQuotaStatusFromHeaders` bin@187249290+; `Xh` bin@206166635 |
| Mixed models and concurrency | One global state is fed by every query source: main loop, subagents, side calls. The latest non-stale response wins (`isStaleObservation`). A Sonnet subagent or Haiku side call that finishes after a credit-billed Fable response clears `overageInUse`, and the main loop's next response sets it again. `active` can flap within a single turn. This follows from the code; not observed live | `$ke` call sites bin@189274023, bin@189378611; `isStaleObservation` |
| Model switch | No reset happens on `/model`. `resetCurrentLimits` is called on account change and remote detach only. After a switch, the last state, which may be `rejected` on `seven_day_opus` with `isUsingOverage: true`, persists until the next response under the new model. The `/model` path itself was not traced (UNVERIFIED) | `Kan` callers bin@200570393, bin@203676900 |
| Window reset | At a reset, a TUI timer forgets that window and locally rewrites a `rejected` state to `allowed` with `rateLimitType`/`resetsAt` cleared, but **leaves `isUsingOverage` unchanged**. It then re-probes, which is a no-op in non-interactive mode. Whether this timer runs in SDK mode is UNVERIFIED. If it does not, the SDK state stays `rejected` / on-overage until the next response | `vr` bin@207449079, `$r` bin@207459196 |
| Credit-only features | Fable on some plans, fast mode, 1M context, `/ultrareview`: credits pay while `status==='allowed'`. Only `overageInUse` shows it, so `isUsingOverage` alone reports "not on overage". Whether fast mode and 1M context set `overage-in-use` is UNVERIFIED | `claude-code-overage.md`; ERR |
| Usage-based billing orgs | `overageInUse` is never forwarded. Only `isUsingOverage` remains, and it is blind to credit-only billing. Wording changes to "usage allocation" | `Srt` bin@205837506 |
| API key, Bedrock, Vertex | No unified headers. The CLI resets state to `allowed` and emits nothing useful. No pool exists to mark | `extractQuotaStatusFromHeaders` (`mF` gate) |
| Refusals (429) | A refused request charges nothing. The 429 headers are applied (status `rejected`, `isUsingOverage` false), so the adapter sees "no pool active, blocked". Rejections re-emit at most once per 30 s per `resetsAt` | `extractQuotaStatusFromError`; `ndo` bin@205837506 |
| Grace and low-priority continuation | Requests can continue past a limit in an internal grace zone (`rateLimitGraceActive`) or a "lower priority" mode (`slow-*` headers) without overage. Neither is a public pool. If they are funded by neither the plan window nor credits, a pool model has nowhere to put them. UNVERIFIED and experiment-gated | bin@180756636; `CXe` |
| Account switch | State resets (`accountEpoch++`), and in-flight responses from the old account are discarded | `resetCurrentLimits` |

## Verdict

- **Overage pool: reliable with caveats.**
  - `overageInUse || isUsingOverage` correctly says whether credits paid for the **most recent completed response**.
  - Caveats:
    - it is retrospective (one response behind)
    - it is unknown before the first response
    - it can flap when models are mixed or requests run concurrently
    - it can be model-scoped (`seven_day_overage_included`)
    - it loses credit-only detection for usage-based-billing orgs
    - it can go stale across a window reset until the next response
- **Included pools: heuristic only.**
  - No field reports which included window a request counted against.
  - Docs say the session and weekly windows count simultaneously, and model-family windows count only for their family.
  - An adapter can deduce a multi-valued `active` set from the request's model, but the model-to-window mapping is
    out-of-band.
  - Push carries no utilization for the Opus and Sonnet windows.
  - `representative-claim` and `is_active` are "binding window" and "display pick", not "charged".
- **RFD implications:**
  - `active` must allow several pools at once.
  - It should be documented as "best effort, as of the last response".
  - It is likely worth separating from a "limiting / binding" marker, which Claude Code does report reliably.
  - Agents should be allowed to omit `active` for included pools.

## Gaps

- No live traffic was captured. Header presence per response (especially `representative-claim` on `allowed`, and
  `7d_oi` on non-Fable models) is inferred from parser code. The 30-minute merge of windows seen across responses
  (`currentWindows`, bin@187250584, flag `tengu_sharded_moonbeam`) hints that responses can carry different window
  subsets, but the reason is UNVERIFIED.
- Whether model-family usage (Opus, Sonnet, Fable) also counts toward session and weekly is not documented.
- Whether plan windows keep accruing while overage pays is not documented.
- Whether the reset timer (`vr`/`$r`) and its local state rewrite run in SDK (non-interactive) mode is not traced.
- Whether `overage-in-use` is set for fast mode, 1M-context, and gateway spend-cap requests is UNVERIFIED.
- The model-switch path was not traced. The "no reset" claim rests only on the `resetCurrentLimits` caller list.

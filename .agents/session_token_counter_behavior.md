## Description

Part of the ACP UI extension protocol (IJAI-873): split three models that are currently conflated — session token usage, current context occupancy/cost, and account quota window with reset time. `usageUpdated` in `AcpSessionEventMapper` puts `usage_update.used` — which core defines as "tokens currently in context", i.e. window occupancy — into the `totalTokens` field of `SessionUsage`. A counter that reads as "tokens this session spent" actually shows what is in the context right now, so after compaction the number *drops* and the session looks like it spent less than a minute ago, while cost keeps rising.

Relative priority within this umbrella: P1.

## Context

- **`PromptResponse.usage` is not read at all.** `AcpSession` takes only `stopReason` from the response and discards `usage` and `_meta` wholesale, so `inputTokens`/`outputTokens` in `SessionUsage` are always empty for ACP sessions — they are filled only when parsing imported JSONL sessions. No input/output, cache or reasoning breakdown is visible for either provider.
- **Account quota is absent from the chat although the data arrives.** Codex receives `account/rateLimits/updated` carrying `RateLimitSnapshot`/`RateLimitWindow` (`usedPercent`, `windowDurationMins`, `resetsAt`) and renders it *only as text inside a slash command* (`CodexCommands.ts`). Claude puts `rate_limit_info` into `_meta["_claude/rateLimit"]`, and `plugins/air` reads that key nowhere. The user hits the limit with no warning.
- **The one visible quota widget is unrelated.** `JbCentralQuotaStatusBarWidget` shows `JbCentralQuotaInfo` (`usedUsd`, `totalUsd`, `remainingUsd`, `percentUsed`, `resetDateText`) — money over a CLI, not Codex or Claude token limits, and not tied to the current session. Mixing the two models into one widget is forbidden; its `resetDateText` is free text, not RFC 3339.
- **Correction to the parent spec:** it claims Codex's legacy `_meta.quota` becomes an alias for `quotaSnapshot`. It does not — `_meta.quota` is `{token_count, model_usage}`, per-model *usage* with no window, limit or reset time, so it aliases the usage model instead. The real `quotaSnapshot` source is `account/rateLimits/updated`.
- **Core contradicts itself:** the `Usage` KDoc and the `PromptResponse.usage` field description say "for a prompt turn" while individual field descriptions say "across session"/"across all turns". So AIR must not sum `PromptResponse.usage` as a delta; the adapter normalises provider semantics into a replacement snapshot and proves it with a multi-turn test.

## Expected Behavior

Standard `usage_update` stays the occupancy/cost snapshot and `PromptResponse.usage` the replacement usage snapshot (both `@UnstableApi`). Account-level `_meta.jetbrains.air.quotaSnapshot` carries required `id`, `revision`, `scope`, `window`, `updatedAt` and optional `usedPercent`, `used`, `limit`, `remaining`, `resetAt`, `status`. Absolute values are optional on purpose: neither provider knows them, so what the provider actually reported is filled in and the rest is never computed or synthesised.

Three independent UI elements that never add up: a context indicator next to the composer ("31k / 200k", plus cost when available) which is the only one that drops after compaction; a session token counter broken down into input / output / cached / reasoning which grows monotonically; and an account quota block ("5-hour limit — 42% used, resets at 18:00"). Claude adds a status (`allowed`, warning, rejected) and shows its overage window as a separate line with its own `id`; Codex shows `primary` and `secondary` as separate lines. Expanding the quota block shows all known windows at once and explains that account quota is unrelated to context or session tokens. With the capability off, the quota block is absent entirely rather than showing zeroes or dashes.

## Open questions

- The object carries both `scope: "account"` and `id: "account:five_hour"`, so the prefix duplicates the field — drop the prefix or declare `scope` derived.
- Claude's `SDKRateLimitInfo` carries a parallel overage window (`overageStatus`, `overageResetsAt`, `isUsingOverage`, `overageDisabledReason`) that the spec does not describe beyond "separate id"; the rule for displaying two windows at once needs a decision.

## Acceptance Criteria

- [ ] Occupancy never lands in a session-totals field; compaction reduces only occupancy and leaves session totals and quota untouched.
- [ ] `PromptResponse.usage` is read and surfaced with the cache and reasoning categories preserved; a multi-turn test proves it is not double-counted.
- [ ] Quota reaches the session UI structurally for both providers, with a stable window id (Codex: `limitId ?? limitName`, separately for `primary` and `secondary`; Claude: `rateLimitType`).
- [ ] A quota reset replaces the correct window; a stale `revision` is ignored so no indicator jumps backwards.
- [ ] A snapshot without absolute values renders as percent plus reset time, never as zeroes.
- [ ] `updatedAt` is displayed ("as of 13:10") but is not used as an ordering key; after replay quota is shown as the last known snapshot rather than passed off as current.
- [ ] A quota fetch error blanks the quota block only, leaving the context indicator and session counter intact.
- [ ] Legacy Codex `_meta.quota` and Claude `_meta["_claude/rateLimit"]` are read for one compatibility cycle, the first deduplicated against the canonical usage model and the second against `quotaSnapshot` by `window`.
- [ ] Covered by `AcpSessionEventMapperTest`, `SessionEventStoreTest` and `ThreadGuiSnapshotMapperTest`.

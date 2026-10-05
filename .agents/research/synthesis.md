# Synthesis: rate limits and quotas across agents

Sources: every other note in this directory. All claims are traceable to those notes.

## 1. Comparison: agent x field x semantics

### 1.1 What a single limit ("window" / "bucket") carries

| Concept | Claude Code | Codex | Gemini CLI | Copilot CLI | Kimi CLI | Cline | Cursor |
|---|---|---|---|---|---|---|---|
| Unit of reporting | window per `rateLimitType` (push) / `limits[]` row (poll) | `RateLimitSnapshot` per `limitId`, with `primary`/`secondary` windows | bucket per `modelId` | snapshot per category (`chat`, `completions`, `premium_interactions`) | `usage` row + `limits[]` rows | spend-limit error; balance | two monthly pools |
| Stable id | `five_hour`, `seven_day`, `seven_day_opus`, `seven_day_sonnet`, `overage`, ...; poll rows have `kind` | `limitId` (`codex`, per-model ids) x position | `modelId` | category key | none (label only) | none | pool name |
| Human label | none on push; `display_name` on model-scoped poll rows | `limitName` (optional) | none | none | `name`/`title` | none | n/a |
| Used share | `utilization` fraction 0..1 (push); `percent` 0..100 (poll) | `usedPercent` 0..100 (int in app-server, f64 in core) | (derived) | (derived) | (derived) | (derived) | percent (undocumented endpoint) |
| Remaining share | — | — | `remainingFraction` 0..1 | `remainingPercentage` | — | — | — |
| Absolute used | — (only overage spend, minor currency units) | — (only spend control: `used`, decimal string) | — | `usedRequests` | `used` | `spent_usd` | — |
| Absolute limit | — (overage `monthly_limit`) | — (spend control `limit`) | — (client derives) | `entitlementRequests` (`-1` = unlimited) | `limit` | `limit_usd` | on-demand limit |
| Absolute remaining | — | — | `remainingAmount` (int64 string) | (derivable) | `remaining` | — | on-demand remaining |
| Unit of amounts | percent only; money for overage | percent; credits for spend control | opaque server units (requests? `tokenType` unread) | requests | unknown | USD | USD / requests |
| Reset time | `resetsAt` epoch s (push), `resets_at` ISO 8601 (poll); optional | `resetsAt` Unix s; optional | `resetTime` timestamp; optional | `resetDate` ISO 8601; optional | `reset_at` ISO or `reset_in` seconds | `resets_at` ISO, nullable | billing cycle dates |
| Window length | implicit in id | `windowDurationMins`; optional | none (UI assumes daily) | implicit (monthly) | `duration` + `timeUnit` | `budget_period` daily/monthly | implicit (monthly) |
| Scope | account; per model (`seven_day_opus`, `model_scoped[]`); per surface (poll `scope`) | account; per model via `limitId` | per model | per feature category | account | user/org (`limit_scope`) | per model family pool |
| Status / severity | push `allowed` / `allowed_warning` / `rejected`; poll `severity` open string (`normal`/`warning`/`critical`); `is_active` | `rateLimitReachedType` enum (reason); `spendControlReached`; `ordinaryUsageAllowed` | none (errors only) | `has_quota`, `usageAllowedWithExhaustedQuota` | none (client colors) | error implies exceeded | none |
| Overage / credits | separate overage window + `overageStatus`, `overageDisabledReason` (13 values), `isUsingOverage`, `extra_usage` | `credits {hasCredits, unlimited, balance}` | credits balance pushed per response | `overage`, `overageAllowedWithExhaustedQuota` | — | account balance (micro-USD) | on-demand spend |
| Plan | `subscription_type` (poll) | `planType` enum (18 values) | tier `id`/`name` | SKU (raw) | — | plan period | — |

### 1.2 Delivery

| Agent | Push | Poll | API-key / BYOK users |
|---|---|---|---|
| Claude Code | `rate_limit_event` on change, derived from headers on every API response | `/usage`, SDK `get_usage` (experimental) | nothing except 429 errors and retry events |
| Codex | `TokenCount.rate_limits` (core); sparse `account/rateLimits/updated`, one bucket per notification (app-server) | `account/rateLimits/read`; TUI polls every 60 s, 5 s near exhaustion | read is rejected; nothing |
| Gemini CLI | credits balance on each response; quota errors | `retrieveUserQuota` at startup, `/stats`, after a response if cache > 30 s | nothing except 429 errors |
| Copilot CLI | `session.error` with `errorType` `rate_limit`/`quota` + `errorCode`; `retryAfterSeconds` on auto-switch | `account.getQuota` | n/a |
| Kimi CLI | — | `/usage` command polls REST | n/a |
| Cline | spend-limit 429 error payload | balance at startup / dialog | provider errors only |
| Cursor | — | undocumented dashboard endpoint | n/a |
| OpenCode | gateway 429 with `limitName` (flattened) | — | header details parsed, unused |
| Amp, Goose, Qwen Code | errors only (Goose: `credits_exhausted` error data) | Amp: `amp usage` text | — |

### 1.3 What reaches an ACP client today

| Agent | Quota over ACP | Limit-hit error over ACP | Token/context fields |
|---|---|---|---|
| Claude Code adapter | raw SDK object in a vendor `_meta` key, piggybacked on `usage_update` (dropped if no occupancy reading yet) | `internalError` with `{errorKind}` | `usage_update` = occupancy; `PromptResponse.usage` = per-turn |
| Codex adapter | none (cached, only `/status` Markdown) | category, no reset time or window | everything from the "last request" breakdown; no session total |
| Gemini CLI | none | JSON-RPC code `429`, text only, retry delay dropped; silent model fallback | `PromptResponse.usage` re-counts history per call; `_meta.quota` = tokens only |
| Copilot, Kimi, Cline, OpenCode, Amp, Goose, Qwen, Cursor | none | free-text errors (Goose: structured `credits_exhausted`) | varies |

## 2. Common ground

1. **Percent + reset time is the lowest common denominator.** Every agent with structured data gives a share (used or
   remaining) and, usually, a reset time. Absolute amounts are the exception, and their units differ (requests, USD,
   credits, opaque units).
2. **A limit is identified by an id that is stable within an agent but not across agents.** Ids encode window length
   (Claude), position plus bucket (Codex), model (Gemini) or feature category (Copilot). Ids are opaque to clients.
3. **Several limits apply at once**: rolling short window + weekly (Claude, Codex, Kimi), per-model windows (Claude,
   Codex, Gemini), monthly allowance (Copilot, Cursor, Cline), and an overage/credits pool (Claude, Codex, Gemini,
   Copilot, Cline).
4. **Every field except a share is optional in practice.** Reset time, window length, label, absolute amounts, plan
   and status are each missing in at least one agent or regime.
5. **Quota is account-level, not session-level.** It is shared across sessions and unaffected by compaction.
6. **API-key users usually get no quota data at all**, so the feature must degrade to "absent", never to zeroes.
7. **Both push and poll exist natively.** Push is per API response or on change; poll is on demand or periodic.
8. **Limit-hit errors are the most common signal and the most lossy over ACP today**: reset time, retry delay, window
   and reason are dropped everywhere.
9. **Token usage and context occupancy are separate concepts** that adapters currently conflate with quota (quota rides
   on `usage_update`; a `_meta.quota` key holds token counts).

## 3. Differences that the design must absorb

- Fraction (0..1) vs percent (0..100); used vs remaining.
- Epoch seconds vs ISO 8601 vs relative seconds for reset.
- Severity: agent-graded three-state (Claude push), server-graded open string (Claude poll), reason enum (Codex),
  boolean (Copilot), or nothing.
- Overage is a separate window in Claude, a credits block in Codex, a balance in Gemini/Cline, a counter in Copilot.
- Some agents publish sparse per-bucket updates (Codex); others publish full snapshots (Claude poll, Copilot, Gemini).

## 4. Open design decisions

1. Delivery: push notification, poll method, or both.
2. Scoping: session update (`session/update`) vs a connection-level (account) notification.
3. Update semantics: full snapshot replacement vs per-limit upsert; staleness handling.
4. Value model: which of percent / absolute values are required; fraction vs percent; used vs remaining.
5. Window identity and description: opaque id + label; window duration; scope (model, category); open enums.
6. Status/severity: agent-graded open enum vs client-derived.
7. Overage, credits and balances: same "limit" shape vs a separate shape.
8. Limit-hit errors: structured error data / new error code / notice / out of scope.
9. Capability negotiation and protocol version (v1 unstable, v2).
10. Timestamp format.

## 5. Overage (milestone 3)

Sources: `*-overage.md` notes for Claude Code, Codex, Gemini CLI, Copilot CLI, Cursor, Cline.

### 5.1 Comparison

| Concept | Claude Code | Codex | Gemini CLI | Copilot CLI | Cursor | Cline |
|---|---|---|---|---|---|---|
| What it is | extra usage billed in money after plan windows | account-wide credits after both windows | AI credits for 3 eligible models | premium requests beyond entitlement | on-demand spend beyond included pools | none (plan and credits are independent providers) |
| Native id | `overage` / `extra_usage` | none | `creditType` | bucket key | none | none |
| Allowed | `overageStatus` allowed/allowed_warning | `hasCredits \|\| unlimited` | strategy + balance >= 50 + eligible model | `overageAllowedWithExhaustedQuota` | `enabled` | — |
| Active | `isUsingOverage` (inferred) and `overageInUse` (server) | inferred only (poll) | inferred (opted in on request) | inferred (`overage > 0`) | inferred | — |
| Unavailable reason | 13 values in 4 groups: exhausted / not available / not set up / indeterminate | depleted / cap reached (owner vs member) / not enabled | proposed: user disabled / insufficient balance / not eligible / model not eligible | none | inferred only | error-time only |
| Balance (remaining, no cap) | not exposed | `credits.balance` (decimal string) | `remainingCredits` (int64 string) | — | — | `current_balance` |
| Count used (no cap) | — | — | per-response `consumedCredits` | `overage` (requests or AI credits) | — | — |
| Cap (limit + used + reset) | `monthly_limit` / `used_credits` (minor units), `overageResetsAt` | `individualLimit` spend cap (monthly) | — | budget exists, not exposed | `onDemand.limit/used/remaining`, `billingCycleEnd` | `SPEND_LIMIT_EXCEEDED` (error only) |
| Unlimited | `monthly_limit: null` | `credits.unlimited` | — | (`entitlement: -1` on base) | `limit: null` | — |
| Units | currency minor units + ISO 4217 | credits (no currency) | credits (no currency) | requests or AI credits | probably USD cents | USD / micro-USD |
| Coverage | all plan windows + credit-only features; not listed | `codex` bucket windows (implicit) | eligible models | the bucket carrying the flags | both pools | n/a |
| Consent | org settings; per-model consent dialog for one model | never asks | `ask` / `always` / `never`; choice lasts one prompt | never asks | opt-in (individual) / admin | never asks |

### 5.2 Findings for the field set

1. **Every agent with overage can state whether it is usable.** "Active" is explicit only in Claude Code; everywhere
   else it is inferred.
2. **Meters come in three shapes**: a balance (remaining only), an uncapped count (used only), and a cap
   (limit + used + reset). Codex has both a balance and a cap at once, so one meter per overage is not enough.
3. **A cap has exactly the shape of a base limit** (used/limit/share/reset/period). Cline's spend limit is already a
   base limit; Claude's monthly cap, Codex's spend cap and Cursor's on-demand limit have the same shape.
4. **Reasons cluster into a few categories**: exhausted (out of funds or cap reached), not enabled (not opted in, not
   set up, user disabled), not permitted (org/seat/plan policy), not eligible (model), unknown.
5. **"Unlimited" must be explicit.** `null` already means "clear" under the patch rule, so it can't also mean unlimited.
6. **Coverage is rarely reported.** It is implied by plan knowledge (Claude, Codex), models (Gemini), or the bucket
   (Copilot). It must be optional.
7. **Units differ** (currency minor units, decimal credit strings, requests, micro-dollars). An explicit unit or
   currency is needed. ACP's `Cost` already uses `amount: number` + ISO 4217 `currency`.
8. **Consent and purchase are interactions, not state.** Only Gemini asks per use; Claude has one per-model dialog.
   Purchase/upsell fields (`credits_required`, `canUserPurchaseCredits`) and Codex reset credits are actions.

### 5.3 Open decisions

- Where caps live: `cappedBy` limit references vs `meters[]` inside overage.
- State model: required `state` + optional `reason`.
- Amount representation (follow `Cost`?).
- Consent and purchase actions: in or out of scope.
- Removal of limits and overages from the keyed list.
- "Unlimited" representation.

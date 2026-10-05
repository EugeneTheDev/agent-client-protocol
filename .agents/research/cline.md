# Cline: quota / credit / rate-limit reporting (triage)

Source: github.com/cline/cline, shallow clone of `main` at commit `68b24a92b71ee98c3e8ea4aca7a7a1c0f97a43b8`
(2026-10-05). Paths below are relative to the repo root. Components: `apps/cli` (Cline CLI, includes ACP mode),
`apps/vscode` (VS Code extension), `sdk/packages/*` (shared runtime).

## Summary

- The Cline ACP mode (`apps/cli/src/acp/`) reports NO quota, credit, rate-limit, or usage data to the ACP client. Agent
  `usage` events are explicitly dropped (`apps/cli/src/acp/session-updates.ts:35-40`), and there is no `usage_update`
  or balance code anywhere in `apps/cli/src/acp/` (grep for `usage` in `acpAgent.ts` returns nothing). Provider errors
  surface only as a failed prompt with a flat message string (`describeAgentError`, `session-updates.ts:~85`).
- Outside ACP, Cline has real quota features, but they are specific to the Cline-hosted provider and are mostly
  poll-on-demand HTTP calls or error-body parsing, not a protocol-level stream:
  1. Cline account credit balance (REST poll; shown in the CLI welcome line and `/account` dialog, and in the VS Code
     account view).
  2. Structured errors from the Cline backend: `insufficient_credits` (402), `SPEND_LIMIT_EXCEEDED` (429, with
     budget period, limit, spent, reset time), ClinePass limit, and free-model daily limit.
  3. OpenRouter key info (`GET /key`) shown as a balance link in VS Code settings.
  4. Generic transient-error retry (429/5xx) with a retry notice event that no UI consumes.
- No usage/remaining/reset windows from other providers (Anthropic, OpenAI, and so on) are read from response headers.
  Header handling is limited to honoring `retry-after` inside the AI SDK and in a banner fetch.

Verdict for the RFD: Cline is a weak data source for the field model, but its spend-limit error payload
(`budget_period`, `limit_usd`, `spent_usd`, `resets_at`) is a useful example of a windowed monetary limit.

## Fields

### Cline account balance (REST)

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `ClineAccountBalance.balance` | number | no | Credit balance of the user; raw unit is micro-dollars (divided by 1,000,000 for display as `$x.xx`) | `sdk/packages/core/src/account/types.ts:31-34`; scaling at `apps/cli/src/utils/output.ts:222-227` |
| `ClineAccountBalance.userId` | string | no | Owner | same |
| `ClineAccountOrganizationBalance.balance` | number | no | Active organization's balance (same unit) | `types.ts:77-80` |
| `ClineAccountOrganizationBalance.organizationId` | string | no | Organization | same |
| `displayedBalance` (CLI-local) | number | no | Org balance if an org is active (falls back to personal), else personal | `apps/cli/src/tui/cline-account.ts:178-189` |
| `UserCurrentPlan.currentPeriodStart` / `currentPeriodEnd` / `cancelAt` / `canceledAt` | string (ISO, presumably) | yes | Subscription period of the current plan; type has an index signature `[key: string]: unknown`, so the real shape is loose | `types.ts:100-110` |
| `ClineAccountUsageTransaction` (`costUsd`, `creditsUsed`, `promptTokens`, `completionTokens`, `totalTokens`, `aiModelName`, `createdAt`, ...) | per-request ledger row | no | History of per-request spend | `types.ts:36-62` |

Endpoints (`sdk/packages/core/src/account/cline-account-service.ts`): `GET /api/v1/users/{id}/balance` (line ~117),
`GET /api/v1/organizations/{id}/balance` (~185), `GET /api/v1/users/{id}/usages`, `GET /api/v1/users/me/plan`.

### Structured error payloads from the Cline backend

Reshaped for the VS Code webview in `apps/vscode/src/sdk/message-translator.ts:2770-2860`; classified in
`apps/vscode/src/services/error/ClineError.ts:160-260` (enum `ClineErrorType` at lines 11-21).

| Error | Fields | Optional? | Meaning | Source |
|---|---|---|---|---|
| `insufficient_credits` (HTTP 402) | `code`, `current_balance: number`, `total_spent`, `total_promotions`, `message`, `buy_credits_url` | only `current_balance` is required for classification as `Balance` | Out of credits | `message-translator.ts:2838-2852`, `ClineError.ts:184` |
| `SPEND_LIMIT_EXCEEDED` (HTTP 429) | `limit_scope`, `budget_period` (`"daily"`/`"monthly"`), `limit_usd: number`, `spent_usd: number`, `resets_at` (ISO 8601, may be `null` for monthly), `message` | all optional; a "minimal" variant carries only `code` and `message` | Org/user-enforced budget cap with window and reset | `message-translator.ts:2854-2868`; `apps/vscode/webview-ui/src/components/chat/SpendLimitError.tsx:26-37`; stories at `ErrorRow.stories.tsx:156-218` |
| `INFERENCE_CAP_ERROR` | `details.message` only | n/a | Maps to `QuotaExceeded`; free text | `ClineError.ts:228-230`; `ErrorRow.tsx:~108` |
| ClinePass limit | free text "You have reached your ... ClinePass limit ... please try again later." | no structured fields | Subscription-plan usage cap, detected by string markers | `sdk/packages/llms/src/providers/errors.ts:10-12,134-141` |
| Free model daily limit | free text containing "free limit reached on model" and "try again in <duration>" | reset time is extracted by substring | Daily free-model cap | `errors.ts:13-14,143-164`; CLI message at `apps/cli/src/utils/cline-pass-errors.ts:50-60` |

### OpenRouter key info (VS Code settings only)

`GET https://openrouter.ai/api/v1/key`, zod schema in `apps/vscode/webview-ui/src/components/ui/hooks/useOpenRouterKeyInfo.ts:7-19`:
`label: string|null`, `usage: number`, `is_free_tier: boolean`, `is_provisioning_key: boolean`,
`rate_limit: {requests: number, interval: string}`, `limit: number|null`. UI uses only `limit` and `usage`:
remaining = `limit - usage`, hidden when `limit === null` (`OpenRouterProvider.tsx:~20-50`). Cached 30 s in the webview
(`CACHE_DURATION_MS`, line 4). `rate_limit` is parsed but not displayed.

### Rate-limit errors from other providers

No structured fields. Detection is by HTTP status 429 plus message regexes (`rate limit`, `per minute`):
`sdk/packages/llms/src/providers/error-classification.ts:33-37,170-175`; VS Code regexes `status code 429`,
`rate limit`, `too many requests`, `quota exceeded`, `resource exhausted` at `ClineError.ts:75`. The `ProviderErrorClass`
type is only `"context_window_exceeded" | "auth" | "unknown"`; `rate_limit` and `billing` are listed as future classes
(`sdk/packages/shared/src/agent.ts:285-300`).

### Token and cost usage (not quota, but adjacent)

- `AgentEvent {type:"usage", usage: Partial<AgentUsage>}` with `AgentUsage = {inputTokens, outputTokens,
  cacheReadTokens, cacheWriteTokens, reasoningTokenCount?, totalCost?}` (`sdk/packages/shared/src/agent.ts:100-120,
  335-337`).
- CLI TUI status bar shows `(totalTokens) $cost`; cost is hidden for subscription providers such as ClinePass
  (`apps/cli/src/tui/components/status-bar.tsx:47-73`). `maxInputTokens`/context window are resolved separately
  (`status-bar.tsx:~100-115`). No percent-of-context or remaining-quota figure.

## Delivery mechanism

- Balance: pull. CLI loads it via `loadClineAccountSnapshot` at interactive startup (welcome line
  `"<email> | Credits: $x.xx | <org>"`, `apps/cli/src/tui/interactive-welcome.ts:170-195`) and when the `/account`
  dialog opens (`apps/cli/src/tui/components/dialogs/account-dialog.tsx:~534-545`). VS Code calls
  `AccountServiceClient.getUserCredits` from the account view with a manual refresh button
  (`apps/vscode/webview-ui/src/components/account/AccountView.tsx:128`, `CreditBalance.tsx`). No periodic polling, no
  push, and not refreshed after each request.
- Limit errors: delivered as ordinary error strings. The CLI TUI string-matches the message and renders a "Cline Credits
  depleted" panel with dashboard links (`apps/cli/src/tui/cline-account.ts:54-68`, `components/chat-entry.tsx:~290-315,
  712`); a code comment calls this temporary until structured error types are passed through (`cline-account.ts:44-45`,
  FIXME). VS Code reshapes the error into JSON in an `api_req_failed` ask message (`message-translator.ts:1985-2030`).
- Retry: the agent runtime emits `{type:"status-notice", message:"provider error — retrying (attempt n/3)",
  metadata:{kind:"provider_error_retry", phase, iteration, attempt, maxRetries, delayMs, providerError}}` before an
  exponential backoff of 1 s doubling, capped at 15 s (`sdk/packages/agents/src/agent-runtime.ts:76-91,1195-1222`). The
  AI SDK separately retries request-start failures up to 5 times honoring `retry-after`
  (`sdk/packages/llms/src/providers/ai-sdk.ts:1075-1089`). Neither `apps/cli/src` nor `apps/vscode/src` consumes
  `status-notice` or `provider_error_retry` (grep, non-test files), and the legacy VS Code `api_req_retried` rows are
  dropped (`apps/vscode/src/sdk/legacy-state-reader.ts:193-196`). So there is no retry UI in practice.
- ACP: nothing. `translateEvent` returns `[]` for `usage`, `error`, `done` (`apps/cli/src/acp/session-updates.ts:28-44`);
  stop reasons map only `completed/aborted/max_iterations/mistake_limit` (`acpAgent.ts:~940-957`). The only account
  call in ACP is organization listing (`apps/cli/src/acp/organizations.ts:69-125`, `fetchUserOrganizations`), not
  balance. The ACP SDK dependency is `@agentclientprotocol/sdk ^0.16.1` (`apps/cli/package.json:71`).
- Banner service: honors `Retry-After` (seconds or HTTP date) for its own 429s on banner fetches only
  (`apps/vscode/src/services/banner/BannerService.ts:366-390`); unrelated to inference quota.

## Windows and semantics

- Credits: a single prepaid balance (no window, no reset). Personal vs organization balance; the organization balance
  wins when an org is active.
- Spend limit: the only explicit window. `budget_period` is `"daily"` or `"monthly"`, `limit_usd`, `spent_usd`,
  `resets_at` (ISO 8601; `null` observed in the monthly story), `limit_scope` (a value `"user"` appears in stories;
  other values UNVERIFIED). Only available after the limit is hit (error), never proactively.
- ClinePass and free-model limits: window not exposed structurally; reset only as free text ("try again in ...").
- OpenRouter: `limit`/`usage` are absolute dollars (key-level cap), `rate_limit.{requests,interval}` is parsed but unused.
- No 5-hour / weekly / per-model windows, no overage flag, no severity or status field, no reading of provider
  rate-limit headers (`x-ratelimit-*`: grep found none).
- Relation to tokens/context: none. Cost (`totalCost`) and token counts are cumulative session stats; the credit balance
  is a separate account-level number never joined to them.

## Gaps and unverified items

- UNVERIFIED: the exact JSON of the Cline backend `/balance` response beyond the TypeScript type; the
  micro-dollar unit is inferred from the `/ 1_000_000` normalization.
- UNVERIFIED: the full set of `limit_scope` and `budget_period` values the backend can send (only stories and
  pass-through code seen).
- UNVERIFIED: the real contents of `UserCurrentPlan` (loose index signature) and whether the plan period is shown to the
  user.
- Not checked: the CLI's non-interactive/JSON output modes for usage or cost fields (only TUI and ACP examined), and
  the `apps/cline-hub` account view (it mirrors the same balance/transactions display,
  `apps/cline-hub/src/webview/src/components/views/settings/account-view.tsx:341,537-542`).
- The retry `status-notice` exists in the runtime but is unconsumed in the CLI and VS Code apps examined, so the
  retry UI claim above is a negative result from grep, not an exhaustive trace.

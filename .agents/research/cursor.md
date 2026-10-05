# Cursor (Cursor CLI `cursor-agent` / `agent`, ACP mode) - quota and rate-limit reporting

Researched 2026-10-05 from public docs and third-party issues. The CLI was not installed locally, so nothing was run.

## Summary

- The Cursor CLI and its ACP mode (`agent acp`) do **not** surface quota, usage-limit, remaining-usage, reset-time or
  rate-limit information, neither to the user nor programmatically. This is documented-absence plus third-party
  confirmation, not a locally run test.
- Plan usage is shown only in the Cursor editor settings and on the web usage dashboard. There the data comes from an
  **undocumented** HTTP endpoint (`GET https://cursor.com/api/usage-summary`) that works with the CLI's stored token.
  It is the only machine-readable quota source found, and it is not part of the CLI or ACP.
- The one usage-adjacent signal in the CLI is a per-turn **token count** (`usage` object) in `--output-format json`
  output. It is observed in a real capture but is not in Cursor's docs. It is not quota.
- A usage-limit error shape for the CLI is unknown. Nobody has captured one (`UNVERIFIED`).
- Recommendation for the RFD: classify Cursor as "no in-protocol quota reporting". It is useful only as evidence that
  an agent may offer a quota source outside its protocol surface.

## Fields

### In CLI / ACP surfaces (no quota fields)

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `usage.inputTokens`, `usage.outputTokens`, `usage.cacheReadTokens`, `usage.cacheWriteTokens` | integer | Observed only with `--output-format json` on the final `result` object. Not in the documented schema. | Token counts for the run. Not quota. | https://github.com/jonathanong/auto-harness/issues/795 (real capture from `cursor-agent 2026.09.10-fd3934a`) |
| `duration_ms`, `duration_api_ms` | number | no | Timing only. Docs say `duration_api_ms` currently equals `duration_ms`. | https://cursor.com/docs/cli/reference/output-format |
| `stopReason` on `session/prompt` result | string | no | Only field shown in the ACP docs' prompt response. | https://cursor.com/docs/cli/acp |
| `agent status` / `whoami` (`--format text\|json`) | text or JSON | n/a | Authentication status, account info, endpoint. No usage. | https://cursor.com/docs/cli/reference/parameters, https://cursor.com/docs/cli/reference/authentication |
| `agent about` (`--format text\|json`), `/about` | text or JSON | n/a | Version, system, account info. A third-party report says it prints email and subscription tier, with no usage or quota. | https://cursor.com/docs/cli/reference/parameters, https://github.com/Dicklesworthstone/coding_agent_account_manager/issues/109 |

The documented `stream-json` event types are `system/init`, `user`, `assistant`, `tool_call` and `result`. None of them
carries usage, cost, quota or rate-limit fields. Docs say consumers "should ignore unknown fields", so fields may be
added later (https://cursor.com/docs/cli/reference/output-format).

Documented ACP Cursor extensions are `cursor/ask_question`, `cursor/create_plan`, `cursor/update_todos`, `cursor/task`
and `cursor/generate_image`. None is quota-related. The docs define no `_meta` or usage fields and no error codes
(https://cursor.com/docs/cli/acp).

The CLI slash-command list has no usage, quota or limits command. Only `/max-mode` (legacy request-based plans) and
`/about` come near it (https://cursor.com/docs/cli/reference/slash-commands).

### Out-of-band: undocumented dashboard endpoint (not CLI, not ACP)

Source for all rows: https://github.com/Dicklesworthstone/coding_agent_account_manager/issues/109. The endpoint is
`GET https://cursor.com/api/usage-summary`, observed with CLI 2026.09.28 on a Team plan. The response is
`UNVERIFIED` by me and may change without notice.

| Name | Type | Optional? | Meaning |
|---|---|---|---|
| `billingCycleStart`, `billingCycleEnd` | ISO-8601 string | unknown | Billing-cycle window. `billingCycleEnd` acts as the reset time. |
| `membershipType` | string (e.g. `enterprise`) | unknown | Plan tier. |
| `limitType` | string (e.g. `team`) | unknown | Whether the limit is per user or per team. |
| `isUnlimited` | bool | unknown | |
| `autoModelSelectedDisplayMessage`, `namedModelSelectedDisplayMessage` | string | unknown | Human-readable text, e.g. "You've used 31% of your included total usage". |
| `individualUsage.plan.{used,limit,remaining}` | number | unknown | Included usage (units unclear, possibly cents). |
| `individualUsage.plan.breakdown.{included,bonus,total}` | number | unknown | |
| `individualUsage.plan.{autoPercentUsed,apiPercentUsed,totalPercentUsed}` | number (percent) | unknown | Per-pool percent used. The pools are Auto/Cursor models vs named API models. |
| `individualUsage.onDemand.{enabled,used,limit,remaining}` | bool / number / null | `limit` and `remaining` are `null` when there is no cap | Pay-as-you-go usage. |
| `teamUsage.onDemand.{enabled,used,limit,remaining}` | same | unknown | Team-level pooled on-demand spend limit. |

Auth for this endpoint: the `accessToken` JWT from `~/.config/cursor/auth.json`, sent as the cookie
`WorkosCursorSessionToken=<userId>%3A%3A<accessToken>`.

## Delivery mechanism

- CLI/ACP: none (nothing pushed, nothing to poll). The user must open the editor settings or the web dashboard.
- Dashboard endpoint: poll-only HTTP GET with no documented cadence. The dashboard also reads it.
- Errors: no documented structured error for usage exhaustion. In stream-json, failure can end the stream without a
  terminal event, and the message goes to stderr, with no structured error fields documented
  (https://cursor.com/docs/cli/reference/output-format). A real usage-limit error has not been captured
  (https://github.com/jonathanong/auto-harness/issues/795).

## Windows and semantics (documented plan model)

- Included usage is split into two pools, "Cursor Models" (first-party) and "Other Models" (third-party, charged at API
  price). Both "reset with your monthly billing cycle"; there are no 5h or weekly windows in the current docs
  (https://cursor.com/docs/account/pricing, https://cursor.com/docs/models-and-pricing).
- After included usage is exhausted the user can enable on-demand (pay-as-you-go) billing or upgrade. On-demand "is
  billed monthly at the same rates" and "Requests are never downgraded in quality or speed" (same pages).
- Team plans: a Premium seat "adds 5x the Standard limits on Agent". The Start plan (India) has no Other Models pool
  and no on-demand usage.
- Rate limits are mentioned only per model in notes (for example "4x rate limits compared to GPT-5.1 Codex"). There is
  no limits table or numeric value in the official docs. The pages for rate limits (/docs/account/rate-limits) and
  spend limits (/docs/account/spend-limits) returned 404, so I could not read a dedicated official page.
- Older third-party descriptions mention "burst" and "local" compute-based limits that refill over hours. These are
  third-party and possibly outdated; treat as `UNVERIFIED`.
- Token usage and context occupancy are not tied to quota in any CLI surface. The `/summarize` command exists to reduce
  context, but nothing reports context-window occupancy.

## Gaps and unverified items

- Not tested on a live CLI: `agent about --format json` and `agent status --format json` field lists are unknown.
- The exact error returned (CLI print/stream-json, ACP JSON-RPC error, HTTP status) when included usage or a rate limit
  is hit is unknown. Cursor docs say requests are not downgraded when on-demand is enabled, but behavior without
  on-demand is not documented in the pages read.
- The `usage` object in `--output-format json` is not in official docs. It is shown only in a third-party capture of
  one CLI version, and it may be absent in `stream-json`.
- `/api/usage-summary` is undocumented, and its units, optionality and stability are unknown. Cursor also has Admin and
  Analytics APIs for Teams/Enterprise. I did not review them (`UNVERIFIED`).
- ACP-mode `agent acp` is marked hidden in the parameters reference, so the surface may change.

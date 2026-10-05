# GitHub Copilot CLI: overage (paid usage beyond the included allowance)

Builds on `.agents/research/copilot-cli.md` (not repeated). Re-verified against `@github/copilot-sdk` 1.0.16
(`npm pack`, `dist/generated/rpc.d.ts`, `dist/generated/session-events.d.ts`; paths below are `rpc.d.ts:N` and
`session-events.d.ts:N` in that tarball). The `copilot` CLI itself is closed source, so everything about CLI behavior is
inferred from SDK types, GitHub docs, or GitHub issues.

## Summary

- The CLI-visible overage data is small: two numbers/booleans per quota bucket, in two shapes of the same data.
  - Typed `account.getQuota` (camelCase, `@experimental`): `overage` (number), `overageAllowedWithExhaustedQuota`
    (boolean), plus `usageAllowedWithExhaustedQuota` (boolean).
  - Raw `quota_snapshots` passthrough (snake_case, all optional): `overage_count` (number), `overage_permitted` (boolean).
- There is NO field for the overage cap (budget), spend in USD, price, or an "overage is active right now" state. A budget
  is a GitHub billing setting (user, cost center, org, or enterprise). Nothing in the SDK types exposes it. `overage`
  counts usage; it is not compared against any exposed limit.
- State is not an enum. "Allowed" is `overage_permitted` / `overageAllowedWithExhaustedQuota`. "Active" must be derived
  (quota exhausted AND `overage > 0`, or `has_quota == false` AND permitted). "Blocked" is only observed after the fact as
  a `session.error` with `errorType: "quota"`, `errorCode: "quota_exceeded"` (HTTP 402).
- Unit is ambiguous in the types: the descriptions say "requests" / "requests/units". Under legacy billing it is premium
  requests (priced $0.04 each); under usage-based billing (since 2026-06-01) the billing unit is AI Credits
  (1 credit = $0.01) and budgets are in USD. The types have a `token_based_billing` flag to tell which applies, but
  whether `overage`/`entitlement` are then in credits is UNVERIFIED.
- Coverage is per bucket. `overage*` fields exist on every `quota_snapshots` entry (`chat`, `completions`,
  `premium_interactions`, plus an open index signature). In practice only `premium_interactions` is expected to have a
  meaningful overage (UNVERIFIED; docs say completions/chat on paid plans are unlimited). Billing-side budgets cover a
  bundle (all AI credit SKUs) or a single SKU, not an individual bucket as seen by the CLI.
- The CLI does not ask the user to consent to overage. Enablement happens outside the CLI (GitHub billing: budget for
  individuals, "premium request / AI credit paid usage" policy plus budgets for orgs and enterprises). The only in-CLI
  spend prompt found is the user-configured per-session AI-credit cap (`session_limits_exhausted.requested`), which is
  unrelated to GitHub overage.

## Fields

### A. `account.getQuota` per-bucket snapshot (`AccountQuotaSnapshot`, rpc.d.ts:6343-6381)

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `overage` | number | required | "Number of additional usage requests made this period" (count beyond entitlement; no unit beyond "requests") | rpc.d.ts:6370-6373 |
| `overageAllowedWithExhaustedQuota` | boolean | required | "Whether additional usage is allowed when quota is exhausted" (paid overage permitted) | rpc.d.ts:6374-6377 |
| `usageAllowedWithExhaustedQuota` | boolean | required | "Whether usage is still permitted after quota exhaustion" (broader: also covers fallback to included/free models; relation to the overage flag not specified) | rpc.d.ts:6362-6365 |
| `isUnlimitedEntitlement` | boolean | required | unlimited bucket, so overage is moot | rpc.d.ts:6350-6353 |
| `entitlementRequests` | number | required | included amount, `-1` = unlimited | rpc.d.ts:6354-6357 |
| `usedRequests` | number | required | used "so far this period" (whether it includes overage is not stated) | rpc.d.ts:6358-6361 |
| `remainingPercentage` | number | required | percent of entitlement remaining (no negative/overage semantics stated) | rpc.d.ts:6366-6369 |
| `resetDate` | string (ISO 8601) | optional | next reset of the quota | rpc.d.ts:6378-6381 |

Container: `AccountGetQuotaResult.quotaSnapshots: { [k: string]: AccountQuotaSnapshot | undefined }` (rpc.d.ts:6330-6340).
Both interfaces are tagged `@experimental`.

### B. Raw `quota_snapshots` entries (`CopilotUserResponseQuotaSnapshots*`, rpc.d.ts:5860, 5946-6140)

All fields optional; entries themselves may be `null`/absent (index signature `... | null | undefined`, rpc.d.ts:5946-5963).
Identical field sets for `chat` (5967-6021), `completions` (6024-6078), `premium_interactions` (6081-6135).

| name | type | optional? | meaning | source (premium_interactions) |
|---|---|---|---|---|
| `overage_count` | number | yes | "Count of additional pay-per-request usage consumed this period beyond the entitlement." | rpc.d.ts:6094 |
| `overage_permitted` | boolean | yes | "Whether usage may continue at pay-per-request rates once the entitlement is exhausted." | rpc.d.ts:6098 |
| `has_quota` | boolean | yes | "Whether the user currently has quota available; when `false` and not unlimited, further requests are blocked until the quota resets." | rpc.d.ts:6126 |
| `entitlement` | number | yes | "requests/units" included; `-1` unlimited | rpc.d.ts:6090 area |
| `unlimited` | boolean | yes | unlimited bucket | rpc.d.ts:6118 area |
| `percent_remaining`, `quota_remaining`, `remaining` | number | yes | remaining amounts at snapshot time | rpc.d.ts:6102-6114 |
| `quota_reset_at` | number | yes | Unix epoch seconds of next reset | rpc.d.ts:6130 area |
| `timestamp_utc` | string | yes | snapshot capture time | rpc.d.ts:6122 area |
| `token_based_billing` | boolean | yes | "Whether this category uses usage-based (token/AI-credit) billing rather than a fixed premium-request count." | rpc.d.ts:6134 |

Account-level (on `CopilotUserResponse`, not per bucket):
- `token_based_billing?: boolean`: "Whether the account is on usage-based (token/AI-credit) billing rather than a fixed
  premium-request quota." (rpc.d.ts:5873-5876)
- `quota_reset_date?: string` (raw) and `quota_reset_date_utc` (normalized) (rpc.d.ts:5857-5859).
- `session.model.list` may carry `quotaSnapshots?: {[k]: JsonValue}` "returned alongside the model list" (rpc.d.ts:21787-21791);
  contents untyped, so overage fields there are UNVERIFIED.

Not present anywhere in the SDK types (grep for `overage|budget|spending` in all `.d.ts`/`.js`: only the fields above):
budget amount, spend in USD, price per unit, currency, "overage active" flag, overage reset (shares the quota reset),
blocked-reason enum.

### C. Error / push signals related to overage

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `session.error.errorType` | string | required | `"quota"` for quota failures | session-events.d.ts:1618-1621 |
| `session.error.errorCode` | string | optional | for `errorType: "quota"` "the CAPI quota error code (e.g., `quota_exceeded`, `session_quota_exceeded`, `billing_not_configured`)" | session-events.d.ts:1616 |
| `session.error.statusCode` | number | optional | upstream HTTP status | session-events.d.ts:1643 |
| (HTTP) 402 body | `{"error":{"message":"You have no quota","code":"quota_exceeded"}}` | n/a | observed in a user report from CLI v1.0.49, when over-quota billing was reportedly enabled | https://github.com/github/copilot-cli/issues/3431 |

The meaning of `session_quota_exceeded` and `billing_not_configured` is not documented anywhere I found (names only).

### D. Not overage: user-configured session AI-credit cap

`session_limits_exhausted.requested` `{ requestId; maxAiCredits: number; usedAiCredits: number }`
(session-events.d.ts:11146-11165). The user answers with `{ action: "add" | "set" | ...; additionalAiCredits?; maxAiCredits? }`
(rpc.d.ts:5389-5400, session-events.d.ts:11195-11220). It is a local, per-session spending guard set via
`--max-ai-credits`, not GitHub billing overage. It is the only in-CLI "may I spend more" consent flow and is a structural
precedent for a user-set cap (`max`/`used`, unit AI Credits), but separate from account overage.

## State and coverage semantics

State derivation (all inferred, since no state enum exists):

| RFD state | How the CLI/SDK data indicates it | Confidence |
|---|---|---|
| allowed | `overage_permitted == true` / `overageAllowedWithExhaustedQuota == true` | field meaning documented in type comments |
| active | quota exhausted (`has_quota == false` or `remainingPercentage == 0`) and `overage` / `overage_count` > 0 | derived; UNVERIFIED that servers populate it this way |
| disabled | `overage_permitted == false` (or absent) while exhausted; later a `quota_exceeded` / 402 error | derived; no reason code. Reason (policy off vs budget exhausted vs $0 budget) is NOT distinguishable in the types |
| blocked by budget | Not exposed. A budget cap hit surfaces only as a 402 `quota_exceeded`. Community report of a hidden `$0` org budget with `prevent_further_usage: true` producing `quota_exceeded` while credits remained: https://github.com/orgs/community/discussions/197813 | observed only via issue reports |

- `usageAllowedWithExhaustedQuota` vs `overageAllowedWithExhaustedQuota`: two separate booleans. The description of the
  first ("usage still permitted after quota exhaustion") likely covers the legacy free-model fallback ("you can still
  use Copilot with one of the included models for the rest of the month",
  https://docs.github.com/en/copilot/concepts/billing/copilot-requests); the second is paid usage. Relation UNVERIFIED.
- Cap: GitHub billing budgets (not visible to the CLI).
  - Individuals (usage-based billing): "Your additional usage budget is set in US dollars, and your usage is shown in
    GitHub AI Credits" (e.g. "$10 budget covers 1,000 AI credits"); additional usage "may be capped"
    (https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-individuals). GitHub may also limit total
    additional credits by "usage patterns, billing history, and verification status"
    (https://github.blog/changelog/2026-06-01-updates-to-github-copilot-billing-and-plans/). So the effective cap can be
    the lesser of a user budget and a GitHub-imposed limit, neither exposed.
  - Orgs/enterprises: user-level budget (ULB, always a hard stop, "caps how many AI credits a single user can consume in a
    billing cycle"), cost center, org, enterprise budgets; "lowest remaining headroom wins"; "Stop usage when budget limit
    is reached" is off by default for non-ULB budgets
    (https://docs.github.com/en/copilot/concepts/billing/budgets-for-usage-based-billing). Budget REST object:
    `budget_type`, `budget_amount` (integer USD), `prevent_further_usage`, `budget_scope`, `budget_product_sku`,
    `consumed_amount`, `expires_at` (https://docs.github.com/en/rest/billing/budgets); org-admin API only, not used by the CLI.
- Reset: shared with the quota. Legacy: 1st of month 00:00:00 UTC, no rollover. Usage-based: "00:00:00 UTC on the first
  day of each calendar month", fixed regardless of subscription date
  (https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-individuals). Budget cap decreases take
  effect next billing cycle; increases apply immediately (budgets doc). In the SDK there is no separate overage reset
  field; use the bucket's `resetDate` / `quota_reset_at`.
- Coverage: per-bucket flags/counters in the CLI view (key = quota type). `premium_interactions` is the only bucket where
  paid overage is documented (legacy: "Purchase additional premium requests at $0.04/request"). Billing budgets cover
  either a bundle of all AI-credit SKUs (`BundlePricing`, `ai_credits`) or one SKU. Code completions and next-edit
  suggestions on paid plans stay unlimited and are not charged against credits
  (third-party summary of the GitHub changelog; changelog page itself says only that all plans bill by AI credits), so
  `completions` overage is expected to be zero/irrelevant (UNVERIFIED).
- Who enables:
  - Individual: sets an additional-usage budget in GitHub billing settings (no budget means credit-consuming features
    stop, completions continue; no automatic switch to a cheaper model). Not purchasable if subscribed via GitHub Mobile
    iOS/Android (legacy page).
  - Org/enterprise: "Premium request paid usage" policy (legacy; "enabled by default" per search summary of
    https://docs.github.com/en/enterprise-cloud@latest/copilot/how-tos/manage-and-track-spending/manage-request-allowances)
    / "AI credit paid usage" policy (usage-based; metered usage needs it, otherwise "usage is blocked when the shared pool
    is exhausted"), then budgets as the cap.
  - Consent in the CLI: none found. The CLI docs page on `/usage` lists only session stats (AI Credits used this session,
    duration, lines edited, per-model tokens) and no budget/overage prompts
    (https://docs.github.com/en/copilot/how-tos/use-copilot-agents/use-copilot-cli). The CLI never asks "continue into
    paid usage?"; the request either proceeds as paid or errors with `quota_exceeded`.

## Mapping to the draft overage entity

| Draft field | Copilot native source | Notes |
|---|---|---|
| `id` | none; use the quota bucket key (`premium_interactions`) or `quota_id` (raw, optional) | one overage per bucket, no independent id |
| `state` = allowed / active / disabled | `overageAllowedWithExhaustedQuota` / `overage_permitted` (allowed vs disabled); `overage` > 0 with exhausted quota (active) | no enum, active is derived |
| state reason | none | cannot tell policy-off vs budget-exhausted vs `$0` budget vs GitHub-imposed cap; only `errorCode` after a failed request (`quota_exceeded`, `billing_not_configured`) |
| `meter.used` | `overage` / `overage_count` | count; unit "requests" (legacy) or presumably AI credits (UNVERIFIED) |
| `meter.limit` / `remaining` | none | the budget (USD) is not exposed; cannot compute remaining overage |
| `meter.unit` / currency | none explicit; `token_based_billing` hints requests vs AI credits. Budget currency is USD, credit = $0.01 (docs) | unit must be inferred |
| `meter.share` | none | |
| `meter.reset` | bucket `resetDate` (ISO 8601) / `quota_reset_at` (epoch s) | same as base quota; monthly UTC |
| covered base limits | implicit: the bucket carrying the flags (usually `premium_interactions`) | no explicit list |

Takeaways for a common field set: required minimum from Copilot is only `allowed: boolean` and `used: number`; everything
else (limit, remaining, unit, reason, covered limits) must be optional. A `reason` and `limit` would be new information
that Copilot cannot supply today but that billing-side budgets could.

## Gaps and unverified items

- UNVERIFIED: what the live Copilot API returns for `overage_count` / `entitlement` under AI-credit billing (credits vs
  requests vs absent). The SDK descriptions say "requests/units". No captured payload; I could not authenticate.
- UNVERIFIED: how `account.getQuota` derives `usedRequests`, `usageAllowedWithExhaustedQuota`, `overage` from the raw
  snapshot (client-side mapping lives in the closed CLI); raw-to-typed mapping is a guess from names only.
- UNVERIFIED: difference between `usageAllowedWithExhaustedQuota` and `overageAllowedWithExhaustedQuota`.
- UNVERIFIED: whether `overage_permitted` reflects the user's budget being set/non-zero, org policy, or both; whether it
  flips to false when a budget is exhausted. Issue #3431 (402 despite "over-quota billing enabled", CLI 1.0.49, closed,
  no maintainer comment, no quota payload) suggests the flags and server enforcement can disagree.
- UNVERIFIED: meaning of `session_quota_exceeded` and `billing_not_configured` error codes (names only in a doc comment).
- UNVERIFIED: whether the CLI shows any overage line in `/usage` or in `/user`-style screens; the docs list none.
- Not checked: Copilot REST usage endpoints (`/users/{u}/settings/billing/premium_request/usage`, etc.) as a source of
  overage cost, and ACP mode (still no documented quota/overage surface; see the main note).
- The "default: enabled" claim for the org-level premium request paid usage policy comes from a search-result summary of
  the GitHub docs; the page fetch did not reproduce it. Treat as UNVERIFIED wording. The "no budget set means features
  stop" and "$0 budget stops usage" statements for individuals come from a secondary summary plus the budgets doc ("Any
  budget set to $0 USD stops usage immediately").
- The legacy page https://docs.github.com/en/copilot/concepts/billing/copilot-requests applies only to annual plan
  holders who stayed on request billing after 2026-06-01; most accounts are now on AI credits, so both unit systems must
  be representable.

# Claude Code: overage ("usage credits", formerly "extra usage")

Scope: only the overage bucket of claude.ai subscription accounts, plus the gateway spend cap that reuses it. This
builds on `claude-code-native.md` (field inventory, delivery cadence) and `claude-code-acp-adapter.md` (ACP
forwarding), and does not repeat them.

Versions: `@anthropic-ai/claude-agent-sdk@0.3.289` `sdk.d.ts` (npm tarball), and the CLI binary
`~/.local/share/claude/versions/2.1.289`. `bin@N` is a byte offset into that Bun bundle. The minified JS read there
is undocumented internal behavior and can change in any release.

Doc sources (fetched 2026-10-05):

- COSTS = https://code.claude.com/docs/en/costs (sections "Check your usage-credits spend", "Add usage credits to your
  subscription", "Claude for Teams and Enterprise", "When a developer asks about a limit")
- ERR = https://code.claude.com/docs/en/errors ("Usage limits" section)
- MODEL = https://code.claude.com/docs/en/model-config ("Fable and usage credits")
- CMDS = https://code.claude.com/docs/en/commands (`/usage-credits`, `/rate-limit-options`)
- SUP = https://support.claude.com/en/articles/12429409-extra-usage-for-paid-claude-plans
- SL = https://code.claude.com/docs/en/statusline

Naming: the product now calls it "usage credits". The wire and type names still say `overage` (headers, push) and
`extra_usage` (poll). `/extra-usage` was renamed `/usage-credits` (CMDS).

## Summary

- **Overage is its own bucket with its own state, reset, and meter, next to the plan windows.** On push it appears
  as a second status (`overageStatus`), a reset (`overageResetsAt`), a disabled reason, a scope (`limitScope`), and two
  "in use" booleans that mean different things. On poll it appears as an `extra_usage` spend meter in currency minor
  units. Neither surface has an id other than the literal `overage` / `extra_usage`.
- **Funding is prepaid credits under a monthly spend cap.** The user or org buys credit (a balance, optional
  auto-reload, and in some jurisdictions expiry) and sets a monthly spend limit, which may be unlimited (SUP).
  The meter Claude Code shows is spend against the monthly cap. The prepaid balance comes from a separate billing
  endpoint, which the SDK does not expose.
- **What it covers:** overage backs every plan window. It takes over when the representative window (5h, 7d, Opus,
  Sonnet, or the per-model `seven_day_overage_included`) is rejected. It also pays for credit-only entitlements, which
  bill to credits even when every window has room: Fable models on some plans and seat tiers, fast mode, and 1M
  context on some plans. No field lists the windows it covers; coverage is implicit.
- **Exhausted versus on overage.** A client tells "included exhausted, now on overage" from
  `status === 'rejected' && overageStatus ∈ {allowed, allowed_warning}`, which the CLI pre-computes as
  `isUsingOverage: true`. `overageInUse` is a different signal: the server says the request was billed to credits.
  That can happen while `status` is `allowed`, for example on a credit-only model.
- **Consent is a CLI-side, per-model gate.** Enabling overage happens on claude.ai, by the user (Pro/Max) or by an org
  admin (Team/Enterprise). The CLI never asks consent for generic overage. It does show a **Fable usage-credits
  consent prompt** before a Fable request bills credits. That prompt is an SDK `request_user_dialog` of kind
  `fable_overage_consent_prompt`.

## Fields

### A. Push: `SDKRateLimitInfo` (stream `rate_limit_event`) and its source headers

Public type: `sdk.d.ts:5646-5668`. The CLI's own zod schema with descriptions is at bin@180754900-180758900. The
serializer `ka` (bin@194693426) maps internal state to SDK fields. The header parser `Lke` is at bin@187244077.

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `overageStatus` | `'allowed' \| 'allowed_warning' \| 'rejected'` | opt; absent when header `anthropic-ratelimit-unified-overage-status` is absent | State of the overage bucket. `allowed_warning` means near its cap: the toast "You're close to your usage credit limit" shows when `isUsingOverage` is true (`bXe`, bin@187236000). `rejected` means overage cannot cover the request | sdk.d.ts:5652; bin@187244077 |
| `overageResetsAt` | int, Unix epoch **seconds** | opt; absent when header `...-overage-reset` is absent | End of the current overage (spend) period | sdk.d.ts:5653; bin@187244077 (`Math.round(Number(...))`) |
| `overageDisabledReason` | 13-value enum (sub-table below) | opt | Why overage cannot be used, from header `...-overage-disabled-reason`. On a `credits_required` 429 it can instead come from body `error.details.disabled_reason` (`Fke`, bin@187246100). The serializer rewrites raw `org_spend_cap_reached` to `org_level_disabled_until` (bin@194694000), so SDK consumers never see the raw value | sdk.d.ts:5654 |
| `isUsingOverage` | boolean | opt in the type; the CLI always sets it (default `false`) | **Derived by the CLI**, not sent by the server: `status==='rejected' && overageStatus ∈ {allowed, allowed_warning}`, i.e. the representative window is exhausted and credits carry the request | sdk.d.ts:5655; bin@187244077 |
| `overageInUse` | boolean | opt; set only when header `anthropic-ratelimit-unified-overage-in-use: true`; **omitted for orgs with `billingType === 'usage_based'`** (`includeOverageInUse: !tF()`, bin@205837737) | Server flag: this response was billed to usage credits. Can be true while `status==='allowed'` (credit-only model). Drives the toast "<model> is now using usage credits instead of your plan limits" (bin@209881800) | sdk.d.ts:5656; header name `D$t` |
| `limitScope` | `'service' \| 'channel' \| 'group_pool'` | opt; "Absent on a plain member denial and from older CLIs" | Which spend cap blocked the request when it is not the member's own: `group_pool` = a pooled group budget, `channel` = one Slack channel's monthly cap, `service` = a service-level monthly cap (the CLI's internal schema places `service` on the Claude-in-Slack surface; UNVERIFIED for other surfaces). From header `...-overage-scope` (`ZXe`) | sdk.d.ts:5661-5664; bin@180758900 |
| `rateLimitType: 'overage'` | enum member | opt | Representative claim is the overage meter itself. Warning text: "You've used N% of your usage credits" (`Zwn`, bin@187238000) | sdk.d.ts:5649 |
| `utilization` (when `rateLimitType==='overage'`) | number, **fraction** of the monthly spend cap | opt | Spend-cap utilization, from header `...-overage-utilization` | bin@187244077 (`ivn`, `Dw` table) |
| `errorCode` | `'credits_required'` | opt; only on a 429 whose body has `error.details.error_code === 'credits_required'` | The request needs credits (credit-only model/feature or exhausted window) and credits cannot be used | sdk.d.ts:5665; `Fke` bin@187246100 |
| `canUserPurchaseCredits` | boolean | opt; only with `errorCode` | Body `details.can_user_purchase_credits`: the user may buy credits themselves | sdk.d.ts:5666; `Fke` |
| `hasChargeableSavedPaymentMethod` | boolean | opt; only with `errorCode` | Body `details.has_chargeable_saved_payment_method`: an inline purchase is possible | sdk.d.ts:5667; `Fke` |
| (internal) `overagePeriodMonthly {utilization}` | fraction of cap | opt; not in public `.d.ts` | "Monthly service spend-cap telemetry for the Claude-in-Slack surface" (header `...-overage-period-monthly-utilization`) | bin@180758632 |
| (internal) `overagePeriodChannel {utilization}` | fraction of cap | opt; "Absent when the channel has no individual cap" | Per-Slack-channel cap (header `...-overage-period-channel-utilization`) | bin@180758829 |
| (internal) `upgradePaths` | `string[]` (comma-split header `...-upgrade-paths`) | opt; **dropped by the SDK serializer** `ka` | Upsell options the server allows. Values seen in UI code: `upgrade_plan`, `overage` (bin@207541945) | bin@187244077 |

`overageDisabledReason` values (sdk.d.ts:5654). The meaning of each is inferred from CLI message mapping (`Qwn`
bin@187234000, `Mto` bin@188299200, fast-mode `yx` bin@181954200) and is not server-documented. The CLI groups them as
follows:

| value | CLI group | user-facing meaning (CLI text) |
|---|---|---|
| `out_of_credits` | exhausted (`Dvn`) | prepaid balance empty: "You're out of usage credits" / "Your org is out of usage · add funds" |
| `org_level_disabled_until` (raw `org_spend_cap_reached`) | exhausted | monthly spend cap hit until the period resets: "You've hit your org's monthly spend limit" / "individual spend limit" |
| `org_level_disabled` | not available | "usage credits turned off by your organization" |
| `org_service_level_disabled` | not available | "This service is disabled for your org" |
| `seat_tier_level_disabled`, `seat_tier_zero_credit_limit` | not available | "Your seat type doesn't include usage credits" |
| `member_level_disabled`, `member_zero_credit_limit` | not available | "Your usage allocation has been disabled by your admin" |
| `group_zero_credit_limit` | not available | "Your group's usage limit is set to $0" |
| `overage_not_provisioned`, `no_limits_configured` | not set up | "requires usage credits · /usage-credits to turn them on" (fast-mode mapping) |
| `fetch_error`, `unknown` | indeterminate | The CLI does **not** retire its overage reading for these two (`applyWindowReadings`, bin@187252000), so they read as "server could not determine eligibility" (inferred) |

### B. Poll: `extra_usage` (`get_usage` reply and `/usage` `usage_report`)

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `extra_usage` | object \| `null` | opt + nullable; absent when the plan has no extra usage or the fetch failed | "Extra-usage (overage) spend for the billing period, when the plan has it" | sdk.d.ts:4336-4342 (get_usage), 6208-6218 (`SDKUsageReport`) |
| `.is_enabled` | boolean | req | "false while extra usage cannot cover sends" (off, not provisioned, or blocked) | sdk.d.ts:6209 |
| `.monthly_limit` | number (**currency minor units**, cents for USD) \| `null` | req, nullable | Monthly spend cap. `null` is rendered as "Unlimited" (bin@217083300, bin@209319800) | sdk.d.ts:6209 |
| `.used_credits` | number (minor units) \| `null` | req, nullable | Spend so far this period. `null` is rendered as "—", or the row is hidden (bin@217083300) | sdk.d.ts:6209 |
| `.utilization` | number **percent 0-100** \| `null` | req, nullable | used / limit. When null and a limit exists, the CLI computes `used/limit*100` clamped to 0-100, and shows 100 for a limit of 0 (bin@217083300) | sdk.d.ts:4341; bin@217083300 |
| `.currency` | string \| `null` | opt + nullable | ISO 4217 code. The CLI defaults to `USD`. The gateway variant validates `^[A-Za-z]{3}$` (bin@188363000) | sdk.d.ts:4342 |
| (endpoint only) `.disabled_reason` | string, nullish | opt | Same vocabulary as `overageDisabledReason`. Read from `/api/oauth/usage` (schema `wvn`, bin@187246700) but **not passed through** to the SDK types | bin@187246700, `nzn` bin@188309800 |

`extra_usage` carries **no reset time**. The `/usage` row renders its reset as the first day of the next local
calendar month (`new Date(y, m+1, 1)`, bin@217083300 and bin@209319800). `limits[]` rows carry no overage meter;
their `kind` values are `session`, `weekly_all`, `weekly_scoped` (sdk.d.ts:6168-6205).

Separate billing endpoints read by `/usage-credits` are not exposed to the SDK: prepaid balance `{amount, currency,
auto_reload_settings.enabled, expiry_policy_months}` (`u_e`, bin@188305500), bundles, payment method, purchase, and
`PUT .../overage_spend_limit {is_enabled, monthly_credit_limit, currency}` → `{disabled_until, used_credits}`
(bin@188303400). These show the server-side model: an enable flag, a monthly cap in minor units, a `disabled_until`
timestamp when capped, and a separate prepaid balance.

### C. Status line `rate_limits.spend_limit` (gateway only)

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `spend_limit.used_percentage` | number, percent; can exceed 100 | opt | Built from the **header `overage` window**, `ovt(utilization)` = fraction×100, only when `He()==='gateway'` | bin@188452600; SL |
| `spend_limit.resets_at` | epoch s | opt | Header `...-overage-reset` | bin@188452600 |
| `spend_limit.used_usd` / `limit_usd` | number, USD (major units) | opt; only when the gateway meter's currency is `USD` and its `resetsAt` matches the header reset | Gateway meter `used/100`, `limit/100`; the meter body is `{extra_usage:{monthly_limit, used_credits, utilization, currency}, limits[]}` in minor units | bin@188452600, bin@188363000 |
| `spend_limit.period` | `'daily' \| 'weekly' \| 'monthly'` | opt | `group` of the active `limits[]` row of the gateway meter (`mso`) | bin@188363000 |

So a gateway spend cap travels as the overage bucket on the wire. The status line omits overage for direct claude.ai
subscribers (SL: "per-model and overage windows are not included").

## State and coverage semantics

### State machine (push path)

The flags are listed by precedence in the CLI's message logic (`bXe`/`Qwn`, bin@187236000).

1. **On overage:** `status==='rejected'` and `overageStatus ∈ {allowed, allowed_warning}` → `isUsingOverage:true`.
   The TUI shows the transition toast once per entry (`limit-reached` notification, bin@209882814): "You're now using
   usage credits · Your session limit resets …", or "Now using usage credits for <model>" for a model-scoped window
   (`Z0r`, bin@187239528). `allowed_warning` while on overage → "You're close to your usage credit limit".
   The `status` / `resetsAt` / `rateLimitType` fields still describe the **exhausted base window**, so a client knows
   when included usage returns.
2. **Overage cannot help:** `status==='rejected'` and (`overageStatus==='rejected'` or `overageDisabledReason` set).
   `overageDisabledReason` + `limitScope` choose the message. If both resets are known, the CLI shows the earlier of
   `resetsAt` and `overageResetsAt` (bin@187234000).
3. **Credit-billed but not exhausted:** `status==='allowed'` and `overageInUse===true` (`isUsingOverage` false). This
   happens with credit-only models. The CLI latches "Fable credits required" and shows the model toast (bin@187254000,
   bin@209881800).
4. **Normal:** `status` allowed or allowed_warning, with no overage fields or `overageStatus==='allowed'`.

`isUsingOverage` and `overageInUse` compared:

- `isUsingOverage` is the CLI's inference: "base window rejected, credits permitted".
- `overageInUse` is the server's per-response billing fact. The SDK drops it for `usage_based` billing orgs, and
  those orgs see "usage limit" wording instead of "spend limit" (ERR).
- Either one may be true without the other.

### Coverage

- **Base windows:** overage takes over when any representative window is rejected. Qwn treats `five_hour`,
  `seven_day`, `seven_day_opus`, `seven_day_sonnet`, and `seven_day_overage_included` alike (bin@187234000).
  - ERR: credits pay "when one of your plan's usage windows has run out, or when the request is one that only usage
    credits pay for".
  - SUP mentions session limits only; it says nothing about weekly or per-model limits.
  - No field states coverage per window.
- **`seven_day_overage_included`:** the zod description calls it the "overage-included weekly (per-model bucket;
  present only for accounts whose responses carry that window)" (bin@180755900). Its display name is "Fable limit"
  (`Vye`, bin@187233000). Header key: `7d_oi`.
  - The CLI's "overage-included models" allowlist defaults to `["Fable","Fable 5","Fable 5.1"]`, overridable by a
    remote flag (bin@186532800). It filters `model_scoped[]` rows (sdk.d.ts:4326).
  - Reading: some plans include a weekly allowance for this model family, and usage past it bills to credits ("Fable
    limit reached · continuing on Fable 5.1 uses usage credits", ERR).
  - So overage **can be scoped to one model**. The scope is implied by the window, not by a field on the overage.
- **Credit-only entitlements (no base window involved):**
  - Fable on some plans and seat tiers (MODEL)
  - fast mode (`FastModeDisabledReason 'extra_usage_disabled'`, sdk.d.ts:773)
  - 1M context on some plans ("Usage credits required for 1M context", ERR)
  - `/ultrareview` after the free runs (CMDS)

  For these, `errorCode:'credits_required'` signals a refusal.
- **Budget hierarchy (`limitScope`):**
  - The overage pool has nested caps: org monthly cap, group pool, member cap, Slack channel cap, service cap
    (COSTS "set spend limits at the organization, group, or individual member level"; ERR).
  - `limitScope` names only the cap that blocked, and only on denial. The public SDK does not expose a meter per cap.
    The internal `overagePeriodMonthly/Channel` are Slack-only.

### Reset and period

- **Push:** `overageResetsAt` comes from the server, in epoch seconds.
- **Poll:** has no reset. The client assumes the first of the next local calendar month (bin@217083300). SUP says
  "monthly spend limit" and "month-to-date" but **does not say calendar month or billing cycle** (UNVERIFIED).
- The prepaid balance does not reset. It can expire, for example after 6 months in Japan (SUP), and the CLI can show
  `expiry_policy_months` (bin@209308000).

### Who enables it and consent

- **Pro/Max:** the user enables it, sets the monthly cap (or unlimited), and buys credits on claude.ai Settings > Usage.
  `/usage-credits` opens that page. When the spend limit is reached with credits still available, the CLI prompts
  inline to raise or remove the limit (COSTS).
- **Team/Enterprise:** admins enable it and set org, group, or member caps. Members without billing access send a
  request through `/usage-credits`, after a confirmation dialog (COSTS, CMDS).
- **No generic overage consent in the CLI.** Once enabled, crossing into overage only produces a toast
  (`USAGE_TRANSITION_PREFIXES`, sdk.d.ts:9819: "Toast only; these never arrive as API errors").
- **Fable consent:** the CLI asks once before a Fable request bills credits (MODEL). Enterprise members on organization
  billing are not asked.
  - The dialog is the SDK `request_user_dialog` of kind `fable_overage_consent_prompt`:
    - payload `{overagesEnabled: boolean, modelName?: string, balanceCents?: number|null, currency?: string|null}`
    - result `'consent' | 'switch_default' | 'cancelled'`, default `cancelled` (bin@188308600)
  - The dialog is emitted only when the host declares the kind in `supportedDialogKinds` (sdk.d.ts:4524-4528).
    In `-p` mode, or when the host does not show the prompt, the CLI bills without asking (MODEL).
  - Consent is stored per org or account in local config `fableOverageConsentV2` (bin@186133790).
  - On fallback, the CLI emits an internal system message `model_consent_fallback` with fields `choice`,
    `original_model`, `fallback_model`, `persisted_as_default` (bin@194692900). This message is not in the public
    `.d.ts`.

### Purchase and upsell fields

- `errorCode`, `canUserPurchaseCredits`, and `hasChargeableSavedPaymentMethod` belong to overage. They appear only on
  a `credits_required` refusal and describe whether the user can **fund** the overage bucket. They are not meter data.
- `upgradePaths` (`upgrade_plan`, `overage`) is a general "what can you do about this limit" hint shared by plan
  upgrade and credits. It is internal, and `ka` drops it from SDK output.
- All of these are actions or affordances, not state.

## Mapping to the draft overage entity

| draft field | Claude Code source | notes |
|---|---|---|
| `id` | literal `overage` (push `rateLimitType`, header prefix) / `extra_usage` (poll) | Only one overage bucket is visible per account. Per-cap ids (group, channel) are not exposed |
| `state: allowed` | `overageStatus ∈ {allowed, allowed_warning}`, `isUsingOverage:false`; poll `is_enabled:true` | `allowed_warning` needs a severity or warning flag on the entity, or on its meter |
| `state: active` | `isUsingOverage:true` (base window exhausted) **or** `overageInUse:true` (credit-billed request) | Two causes. The entity may need to say whether "active" is due to a covered limit being exhausted or to a credit-only resource. `overageInUse` is absent for usage-based-billing orgs |
| `state: disabled + reason` | `overageDisabledReason` (13 values); poll `is_enabled:false` (reason not passed through) | Claude Code splits these into **exhausted** (`out_of_credits`, `org_level_disabled_until`), **not available** (org/seat/member/group/service), **not set up** (`overage_not_provisioned`, `no_limits_configured`), and **indeterminate** (`fetch_error`, `unknown`). `overageStatus:'rejected'` with no reason is a fourth shape. The draft's `disabled` would need a reason category, or a separate `exhausted` state |
| `reason` scope | `limitScope` (`group_pool`, `channel`, `service`; absent = member's own cap) | Fits as "which cap" on the reason, not as a separate entity |
| `meter.used` | `used_credits` (minor units); gateway `used_usd` (major USD) | Poll only, plus status line for gateways |
| `meter.limit` | `monthly_limit` (minor units); `null` = unlimited | The draft needs "unlimited" distinct from "unknown". Claude Code uses `null` for both, depending on context: in the `/usage` row, null limit = Unlimited |
| `meter.remaining` | **absent** | Derivable as limit-used. The prepaid balance is a different quantity, not exposed |
| `meter.unit/currency` | `currency` (ISO 4217, optional, default USD) with **minor units** | The draft should state the minor-unit scale explicitly, or carry a decimal amount |
| `meter.share` | push: fraction 0..1 (`utilization` when `rateLimitType==='overage'`); poll: percent 0-100 | Same unit mismatch as the base windows |
| `meter.reset` | push `overageResetsAt` (epoch s); poll none (client assumes the 1st of the next calendar month) | Optional in practice |
| `covers[]` | **not reported.** Implied: all plan windows that can be representative claims, plus credit-only resources (Fable, fast mode, 1M context) | A `covers` list would have to be filled by the agent from out-of-band knowledge. Claude Code would most honestly report "all" or omit it. Per-model coverage exists only through the `seven_day_overage_included` window |
| purchase / upsell (not in draft) | `errorCode:'credits_required'`, `canUserPurchaseCredits`, `hasChargeableSavedPaymentMethod`, `upgradePaths` | Better modelled as actions on a refusal than as overage state |
| consent (not in draft) | `request_user_dialog` `fable_overage_consent_prompt` | Per-model consent before credit billing. This is an interaction, not quota state |

## Gaps and unverified items

- The server does not document the meaning of each `overageDisabledReason`. The meanings above come from CLI message
  text. `no_limits_configured`, `fetch_error`, and `unknown` are the least certain.
- Whether the overage period is a calendar month or the billing cycle is UNVERIFIED. The poll surface has no reset;
  the CLI assumes a calendar month.
- Whether overage covers `seven_day_opus` / `seven_day_sonnet` exhaustion for every plan is UNVERIFIED. The CLI code
  handles those windows the same way, and SUP mentions only session limits.
- `limitScope: 'service'` beyond the Slack surface is UNVERIFIED.
- `overagePeriodMonthly`, `overagePeriodChannel`, `upgradePaths`, the `fable_overage_consent_prompt` dialog payload,
  and the `model_consent_fallback` message come from the bundle only. They are not in the public `.d.ts`.
- The relationship between the header `overage-utilization` (fraction) and poll `extra_usage.utilization` (percent)
  is assumed to be the same quantity on different scales. Not verified against live traffic.
- The prepaid balance, auto-reload, and expiry are visible only in the `/usage-credits` UI via private endpoints.
  No SDK field carries them.
- The ACP adapter forwards push overage fields unchanged inside `_meta` and validates, but does not render, poll
  `extra_usage` (see `claude-code-acp-adapter.md`).

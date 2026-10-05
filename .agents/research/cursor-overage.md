# Cursor - overage ("on-demand" / usage-based charges)

Researched 2026-10-05 from official docs/help pages and third-party captures of the undocumented dashboard endpoint.
Builds on `.agents/research/cursor.md` (not repeated: CLI/ACP expose no quota fields). Nothing was run locally.

## Summary

- Cursor's overage is "on-demand usage": pay-as-you-go billing at API rates after included usage is exhausted. Official
  docs describe it only in prose; there is no documented machine-readable schema.
- The only structured source is the undocumented dashboard endpoint `GET https://cursor.com/api/usage-summary`. It
  carries `onDemand` meters shaped `{enabled, used, limit, remaining}` in two places: `individualUsage.onDemand` and
  `teamUsage.onDemand`. Units are most likely cents (UNVERIFIED for the individual meter; stated as cents for team
  meter by one source).
- State is only partly derivable: `enabled` plus `used`/`limit`/`remaining` can express disabled, enabled-idle, and
  "limit reached" (`remaining == 0`). There is no reason code (admin-disabled vs user-disabled vs plan-excluded), no
  "currently active" flag, and no currency field.
- Enabling: individuals opt in explicitly; Teams default to enabled; team admins (or members, depending on a toggle)
  set limits. No consent prompt is documented.
- Coverage: official docs never say whether on-demand backs both pools. They say it applies after "included monthly
  usage" is exhausted, and for Teams the order is third-party pool -> Cursor Models pool -> on-demand. Treat as one
  shared overage that covers both pools (documented for Teams, implied for individuals).

## Fields

### A. `GET https://cursor.com/api/usage-summary` (undocumented; UNVERIFIED by me, community captures only)

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `individualUsage.onDemand.enabled` | boolean | Block may be absent ("only appear when enabled / on team plans"); present as `false` on a free plan in another capture | Whether on-demand is turned on for this user | https://github.com/noamsto/tmux-og/issues/778 , https://github.com/tddworks/ClaudeBar/issues/303 |
| `individualUsage.onDemand.used` | number | present when block present | On-demand spend so far this cycle; sample `2309`, printed as cents by the gist script | https://gist.github.com/dmwyatt/1e9359b1862e7cbfe1e754fe4c8db764 |
| `individualUsage.onDemand.limit` | number or `null` | nullable | Per-user spend cap. `null` = no cap | caam issue #109 (`"limit": null, "remaining": null`), https://github.com/Dicklesworthstone/coding_agent_account_manager/issues/109 |
| `individualUsage.onDemand.remaining` | number or `null` | nullable | `limit - used`; `null` when `limit` is `null` | same |
| `teamUsage.onDemand.{enabled,used,limit,remaining}` | bool / number / number-or-null | `teamUsage` only on team plans | Team-wide pooled on-demand meter. Capture: `used 91272, limit 100000, remaining 8728` (so remaining = limit - used). Gist describes it as team-wide spend limits and usage "in cents" | caam #109; gist; tmux-og #778 (`limit: 1000000`) |
| `individualUsage.overall.{enabled,used,limit,remaining}` | bool / number / number-or-null | Enterprise payloads only; `plan` absent there | Per-user overall (included + on-demand under pooled/enterprise) limit. `limit` is the per-user limit in cents (`15000` = $150 in one capture). `enabled:false`, `limit:null` in another | https://github.com/getagentseal/codeburn/issues/1546 , https://github.com/ClearMeasureLabs/Cursor-Usage-Status/pull/2 |
| `isUnlimited` | boolean | top-level | `true` = no limit at all (no team default fallback) | Cursor-Usage-Status PR #2 |
| `limitType` | string (`"team"` seen) | top-level | Whether the limit is team- or user-scoped | caam #109 |
| `membershipType` | string | top-level | Plan tier (`enterprise`, `pro`, ...) | gist |
| `billingCycleStart`, `billingCycleEnd` | ISO-8601 string (with or without fractional seconds) | top-level | Monthly cycle. `billingCycleEnd` is the reset for both included usage and on-demand | tmux-og #778; gist |
| `individualUsage.plan.{used,limit,remaining,breakdown,*PercentUsed}` | number | absent on Enterprise; `used`/`limit` stay 0 on free plans | Included-usage meter (not overage). Percent fields are 0-100; `autoPercentUsed` = Cursor-models pool, `apiPercentUsed` = third-party pool | ClaudeBar #303, tmux-og #778 |

Unit evidence is mixed: gist says cents; tmux-og #778 says "dollars (units to verify: cents or USD)"; codeburn #1546
sample values are placeholders. Treat unit as cents, `UNVERIFIED`. No currency field exists; the currency is implicit
(USD for the US; Start plan is billed in INR but has no on-demand).

### B. Admin API (official, Teams/Enterprise admin, API-key auth): https://cursor.com/docs/account/teams/admin-api

| Name | Type | Optional? | Meaning | Source |
|---|---|---|---|---|
| `POST /teams/spend` -> `teamMemberSpend[].spendCents` | number | no | On-demand spend this cycle, in cents; included usage not counted. Extra decimal precision since 2026-06-04 | admin-api |
| `overallSpendCents` | number | no | On-demand plus included, cents | admin-api |
| `monthlyLimitDollars` | number or `null` | nullable | Per-user monthly limit in dollars; `null` = none | admin-api |
| `hardLimitOverrideDollars` | number | no | Per-user override in dollars; `0` = no override | admin-api |
| `effectivePerUserLimitDollars` | number | no | Limit actually enforced; derived, may come from a team default not shown in the other two fields | admin-api |
| `subscriptionCycleStart` | epoch ms | top-level | Cycle start | admin-api |
| `POST /teams/user-spend-limit` `{userEmail, spendLimitDollars}` | string, integer-or-`null` | required | Set/remove per-user limit; whole dollars only; Enterprise only | admin-api |

Unit asymmetry to note: spend in cents, limits in dollars.

### C. Not found

No on-demand fields in `agent` CLI output, stream-json, or ACP (see `cursor.md`). Hard-limit RPCs (`GetHardLimit`,
`GetAggregatedUsageEvents`, `perUserMonthlyLimitDollars`) are mentioned only by a third-party extension PR
(Cursor-Usage-Status PR #2) and are `UNVERIFIED`.

## State and coverage semantics

State a client can derive from `usage-summary` (all inferred, `UNVERIFIED`; no official doc defines `enabled`):

| Observable | Reading |
|---|---|
| `onDemand` block absent, or `enabled:false` | Disabled (individual default; or plan excludes it, e.g. Start). Reason not distinguishable |
| `enabled:true`, `used == 0` | Allowed, not yet used |
| `enabled:true`, `used > 0` and included meter exhausted (`plan.remaining == 0` or percent 100) | Active (overage being consumed). Active-ness must be inferred from the base meter, not a flag |
| `enabled:true`, `limit` non-null, `remaining == 0` (or `used >= limit`) | Blocked by spend limit |
| `limit == null` | Uncapped |
| `isUnlimited:true` | No cap at all |

Block behaviour (official, https://cursor.com/help/account-and-billing/spend-limits and
https://cursor.com/help/account-and-billing/overages): at the limit "AI features stop working for that specific user",
on-demand "stops until you increase the spend limit or a new billing cycle starts", user sees a notification.
Enforcement is not instant, so spend can briefly exceed the limit (becomes a temporary spend-limit credit). Included
usage after the limit is not addressed directly. Without on-demand, behaviour at exhausted included usage is not
documented in the pages read; docs only offer "add on-demand or upgrade".

Teams: one team-level limit; when hit, "all members consuming on-demand usage lose access to AI features". Enterprise:
team-level and member/group-level limits, "highest applicable limit" wins; for pooled Enterprise, member limits "apply
to total usage, not just on-demand". Per-user on-demand limits on Teams were deprecated 2025-12-05 and replaced by
email alerts that do not block (https://forum.cursor.com/t/team-admins-now-can-control-spend-alerts-and-limits/145259).

Reset: on-demand spend and limits reset with the billing cycle (`billingCycleEnd`); raising a limit mid-cycle can make
previously credited overage billable.

Coverage (which base limits it backs):
- Teams: "Once a member uses up their third-party allowance, Cursor switches them to the Cursor Models pool ... If they
  keep using third-party models, or exhaust the Cursor pool too, they continue with on-demand usage if it is enabled"
  (https://cursor.com/docs/account/teams/pricing). So on-demand backs both pools, in sequence.
- Individuals: "When you exceed your included monthly usage ... Add on-demand usage"; the pools page names the extra
  payment option only for Other Models and explicitly does not say what happens when Cursor Models runs out
  (https://cursor.com/docs/models-and-pricing). Whether Cursor-models-only usage can spill into on-demand for individuals
  is `UNVERIFIED`.
- Start plan (India): no Other Models pool and no on-demand at all.
- Metering is a single shared meter, not per pool: one `onDemand` object, no per-pool breakdown. Billing is "at the
  same rates"; Teams/Enterprise add the $0.25/M-token Cursor Token Rate on third-party models, first-party exempt.

Who enables and sets limits:
- Individuals (Pro/Pro+/Ultra): off by default; "must be explicitly enabled in your settings"; user sets own monthly cap
  in dashboard Spending tab. This explicit enable is the only consent-like step documented.
- Teams: "enabled by default"; admins can disable or set the team limit. "Only Admins Can Edit Usage Settings"
  permission toggle (Spending tab): off = any member may change, on = admins only. Enterprise member/group overrides are
  always admin-only. Changes audited as `admin_only_usage_pricing` on Enterprise.
- No consent prompt/interstitial is documented (spend-limits help page: "says nothing about consent prompts").
- Optional: "Dynamic Spend Limits" auto-scale the team limit with seat count; email spend alerts at user/team level.

## Mapping to the draft overage entity

| Draft field | Cursor native | Notes |
|---|---|---|
| `id` | none | Synthesize (e.g. `on-demand`; team vs individual meter as separate entities if both exposed) |
| `state: allowed` | `enabled:true`, `used == 0` (or `enabled:true` with base meter not exhausted) | Inferred |
| `state: active` | `enabled:true` and base meters exhausted / `used > 0` | No native flag |
| `state: disabled` + reason | `enabled:false` or block absent -> disabled. Reasons from docs only: user-not-opted-in (individual default), admin-disabled (Teams), plan-excluded (Start), spend-limit-reached (`remaining == 0`) | Reason NOT in payload; client must infer from plan/`membershipType`, and "limit reached" from meter |
| `meter.used` | `onDemand.used` | number, cents (UNVERIFIED) |
| `meter.limit` | `onDemand.limit` | `null` = uncapped; map to absent |
| `meter.remaining` | `onDemand.remaining` | `null` when uncapped; equals `limit - used` |
| `meter.unit/currency` | none | Implicit USD cents; need protocol-level `unit: "cents"` + `currency` supplied by adapter, or omit |
| `meter.share` | none native | Could derive `used/limit`; team vs individual scope is expressed by which object (`teamUsage` vs `individualUsage`) |
| `meter.reset` | top-level `billingCycleEnd` | Shared with the base limits, not on the onDemand object |
| covers (base limits) | all included pools: Cursor Models (`autoPercentUsed`) and Other Models (`apiPercentUsed`), in sequence | One overage entity covering both pool ids; Teams doc is explicit, individual is implied |

Design-relevant observations: (1) the meter is optional by nature (disabled/absent block, nullable limit/remaining), so
`meter` and each of its fields must be optional; (2) scope (user vs team) is a real distinction that can both be present
at once, so overage may need a `scope` or multiple entities; (3) the "blocked" state is derived from the meter, so an
explicit `state` with reason is more informative than anything Cursor itself exposes; (4) spend and limits use
different units in the Admin API (cents vs dollars), so an explicit `unit` is warranted.

## Gaps and unverified items

- `usage-summary` is undocumented and community-captured; field optionality, units (cents vs dollars), and meaning of
  `enabled` are not confirmed by Cursor. One Enterprise capture shows `teamUsage.onDemand` with `limit:1, remaining:1`
  that may be placeholder values.
- Per-user vs team meters: not confirmed whether `individualUsage.onDemand.limit` reflects the Enterprise member limit.
  In Enterprise captures the per-user limit appears under `individualUsage.overall.limit` instead (cents).
- Whether individuals' Cursor-Models-only usage can spill into on-demand is undocumented.
- Exact behaviour/error returned when included usage is exhausted and on-demand is off, or on-demand limit is hit, in
  the CLI/ACP (no structured error documented; see `cursor.md`).
- No official doc defines "active" vs "enabled"; the table above is inference.
- Hard-limit RPCs on `api2.cursor.sh` (`GetHardLimit`, `/auth/usage-summary` with Bearer token) are known only from one
  third-party PR: https://github.com/ClearMeasureLabs/Cursor-Usage-Status/pull/2 (`UNVERIFIED`).
- Pages https://cursor.com/docs/account/spend-limits and /rate-limits returned 404 earlier; the authoritative text is the
  help-center page https://cursor.com/help/account-and-billing/spend-limits.

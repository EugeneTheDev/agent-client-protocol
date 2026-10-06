# Cursor - can an agent know which quota pool is being charged right now?

Researched 2026-10-06 from official docs, Cursor staff forum replies, and third-party captures. Nothing was run locally.
Builds on `.agents/research/cursor.md` and `.agents/research/cursor-overage.md` (not repeated: CLI and ACP expose no
quota fields; `usage-summary` is undocumented; `onDemand` has no "active" flag).

## Summary

- Pool assignment is by model, not by request metadata. Named third-party model -> Other Models / "API" pool.
  Composer and Grok 4.x -> Cursor Models / "Auto + Composer" pool. `Auto` -> not documented; routing is decided by
  Cursor server-side, and neither the CLI nor ACP documents any signal that says which model Auto picked.
- The mapping is only partly official. Models-and-pricing lists which models are in the Cursor Models pool and says a
  chosen third-party model draws from Other Models. The spillover rules (Cursor pool -> API pool -> blocked/on-demand)
  come from staff forum replies. Official pages on Teams and individuals appear to disagree on direction (see Failure
  modes).
- There is no per-request pool signal in the CLI or ACP. The pool-level meters exist only in the polled, undocumented
  `GET https://cursor.com/api/usage-summary`. A per-event ledger exists out-of-band (`kind`, `isChargeable`,
  `chargedCents`, `model`), but it is a different, authenticated, hourly-aggregated or cookie-based API and has no pool
  field.
- On-demand: no per-request signal reachable from the agent. The only practical signals are `onDemand.used` rising
  between polls, or per-event `kind == USAGE_BASED` / `isChargeable` in the out-of-band usage-events API.
- Verdict: included pools "reliable with caveats" when the user picks an explicit model, "heuristic only" under Auto
  or after the first pool is exhausted. On-demand "heuristic only".

## Signals

| Signal | Meaning | When absent | Source |
|---|---|---|---|
| Selected model id (user choice) | Determines pool for explicit models: Grok 4.x / Composer -> Cursor Models; any third-party model -> Other Models, "at that model's API rate" | Under `Auto` the selected id is the router, not the model charged. In ACP the docs show no model selection or reporting mechanism (UNVERIFIED whether `agent acp` exposes `models`/config options at runtime) | https://cursor.com/docs/models-and-pricing, https://cursor.com/docs/cli/acp |
| CLI `system/init.model` (stream-json) | "model display name" at session start | Not documented to show the resolved model for Auto. No other event type has a model field. Absent in ACP | https://cursor.com/docs/cli/reference/output-format |
| `usage-summary` `individualUsage.plan.autoPercentUsed` | Cursor Models / "Auto + Composer" pool, % used (0-100) | Cursor can omit `autoPercentUsed` or `apiPercentUsed`. Start and Express plans carry only `totalPercentUsed`. Enterprise payloads have no `plan` | https://github.com/tddworks/ClaudeBar/issues/303, https://github.com/sendbird/stave/pull/592, `cursor-overage.md` |
| `usage-summary` `individualUsage.plan.apiPercentUsed` | Other Models / "API" pool, % used | Same as above | same |
| `usage-summary` `totalPercentUsed` | Combined gauge. Can read low while the API pool is near exhausted (sample 39.9 / 97.2 / 44.7) | Always the only value on plans without the split | https://github.com/Dicklesworthstone/coding_agent_account_manager/issues/112, ClaudeBar #303 |
| `usage-summary` display messages `autoModelSelectedDisplayMessage` / `namedModelSelectedDisplayMessage` | Text the dashboard shows for the Auto-selected and named-model cases ("... of your included total usage" / "... of your included API usage"). The names hint that the dashboard picks the pool by "Auto selected" vs "named model selected" | May be absent. Free text, not machine-stable | https://gist.github.com/dmwyatt/1e9359b1862e7cbfe1e754fe4c8db764 |
| `usage-summary` `individualUsage.onDemand.{enabled,used,limit,remaining}` | On-demand meter. `used` in cents (UNVERIFIED) | Block absent or `enabled:false` when not enabled. `limit`/`remaining` null when uncapped | `cursor-overage.md` |
| Usage-events API (dashboard `POST /api/dashboard/get-filtered-usage-events`, session cookie + `Origin` header, undocumented; official Teams-admin `POST https://api.cursor.com/teams/filtered-usage-events`) | Per-event ledger: `timestamp`, `model`, `kind` (`USAGE_EVENT_KIND_USAGE_BASED` = on-demand, `USAGE_EVENT_KIND_INCLUDED_IN_BUSINESS` = included; Admin API: `Usage-based` / `Included in Business`), `isChargeable`, `chargedCents`, `requestsCosts`, `tokenUsage`, `cursorTokenFee` | No pool field, so the Cursor Models vs Other Models split must come from `model`. Admin API: team admin API key only, data "aggregated at the hourly level", poll "at most once per hour", 60 req/min. Since 2026-07-31 dollar fields are zeroed on self-serve plans (including Teams) in the dashboard endpoint, including history. Enterprise keeps dollars | https://gist.github.com/dmwyatt/1e9359b1862e7cbfe1e754fe4c8db764, https://cursor.com/docs/account/teams/admin-api, https://www.developersdigest.tech/blog/cursor-removes-dollar-costs-usage-page |
| Dashboard usage tab per-request list | "lists each individual request, the model used, and the cost" (staff) | Not machine-accessible except via the unofficial endpoint above. Whether a row names the pool is not stated | https://forum.cursor.com/t/billing-of-auto-and-composer-1-5-usage-in-paid-plans/153544 |
| Dashboard text "Consumed by Auto. Additional usage consumes API quota." | Staff quote of the dashboard wording for spillover | Display only | https://forum.cursor.com/t/does-composer-2-5-fall-back-to-api-usage-after-auto-composer-reaches-0-on-individual-pro/163366 |
| Errors / blocked requests | When both pools are exhausted and on-demand is off, "requests just get blocked", "nothing extra will be charged" (staff) | No structured error documented for CLI or ACP (`cursor.md`) | same forum thread |

Admin API `kind` values are listed only as examples ("e.g."). No other values are documented.

## Deduction rules

An adapter could fill `active` as follows. All rules are inferred, not defined by Cursor.

1. Known model (user picked it explicitly, not Auto):
   - Model in {Composer *, Grok 4.x} -> Cursor Models pool is the intended charge target.
   - Any other model -> Other Models pool is the target.
   The model-family list changes over time (docs list Grok 4.7/4.6/4.5, Composer 2.5 today) and must be updated or
   fetched. Unknown ids default to Other Models in community code (oh-my-pi #13200), which is a guess.
2. Auto selected: pool unknown. Mark both included pools as possibly active, or mark none. Community tools treat Auto
   as "must see both pools" (https://github.com/sendbird/stave/pull/592). Teams docs say Auto bills "at the list price
   of the model each request is routed to" and add the Cursor Token Rate "when Auto routes to a third-party model"
   (https://cursor.com/docs/account/teams/pricing), which implies Auto can land on the third-party pool.
3. Spillover (individuals, staff statement): if `autoPercentUsed >= 100` and the current model is Composer, Grok or
   Auto, the charge goes to the API pool. If `apiPercentUsed >= 100` and the model is a named third-party model, the
   request is either on-demand (if enabled) or blocked. The user has no choice about spillover (forum "Why does Auto
   silently fall back to API usage" shows users surprised by it; replies not read).
4. On-demand `active`: set when `onDemand.enabled` and the pool that the current model charges to is exhausted (rule 3),
   or when `onDemand.used` increased since the previous poll, or when the latest usage event has `kind == USAGE_BASED`
   / `isChargeable == true`. The first two are heuristics. The third is direct but out-of-band.
5. Use the transition of `*PercentUsed` between two polls around a prompt as a confirmation signal. Only the pool whose
   percent moved was charged. It cannot attribute usage to a single request if other sessions run concurrently on the
   same account.

## Failure modes

- Auto is opaque. No documented signal says what Auto resolved to, in the CLI, ACP or usage-summary. The only post-hoc
  route is the per-event ledger `model` field.
- Spillover direction is contradictory across sources.
  - Staff on individual plans: Auto + Composer pool spills one-way into the API pool; named models "charged to the API
    quota right away"; never back (forum 163366, 164677, June to July 2026).
  - Teams docs (https://cursor.com/docs/account/teams/pricing): "Once a member consumes all of their included
    third-party Other Models usage, Cursor switches them to the Cursor Models pool", then on-demand. This reads as the
    reverse direction, or as a plain-language gloss. Not resolved (UNVERIFIED which is current for Teams).
  - A staff reply in June 2026 said Team has "a single shared pool", which the Teams June 2026 blog and docs have since
    replaced with two pools (https://cursor.com/blog/teams-pricing-june-2026). That blog says nothing about spillover.
- Display bugs. Staff acknowledged "the in-chat usage indicator shows wrong pool percentage after model switch"
  (forum 164677). Another bug report claims Composer 2.5 drains the API limit instead of the Auto + Composer limit
  (referenced in forum 163366, not read). So even Cursor's own indicator can name the wrong pool.
- Staleness. Community note: "some of these usage meters are heavily delayed" (forum 153544, community reply, not staff).
  No documented cadence for `usage-summary`. The Admin API is hourly-aggregated. Treat the pool percentages as
  eventually consistent, so "which pool moved during this prompt" can be wrong or empty for a short turn.
- Pool fields can be missing. `autoPercentUsed` or `apiPercentUsed` can be omitted, and plans without the split give
  only `totalPercentUsed`. Falling back to checking every bucket caused wrong blocking in a community tool
  (stave #592).
- The user-level and team-level views differ. Teams: allowance per seat ("does not transfer between team members");
  pooling across a team is Enterprise only. Enterprise payloads lack `individualUsage.plan`.
- On-demand `used` only changes at billing-granularity updates. It cannot be tied to a request, and the Admin API
  (hourly) is too coarse. Concurrent sessions on the same account confound attribution.
- Cost fields are now unreliable. Since 2026-07-31 `chargedCents` is zeroed for self-serve plans, so an event ledger
  must rely on `kind` and `isChargeable`, not on cents. `isChargeable` is also not a clean test: an Enterprise capture
  has an `INCLUDED_IN_BUSINESS` event with `isChargeable: true`
  (https://github.com/getagentseal/codeburn/issues/1545, via search summary).
- Both-pools-exhausted behaviour without on-demand: staff say requests are blocked. Official docs do not say this and
  document no error shape.
- Older plans (unlimited Auto, request-based) behave differently per staff ("dashboard is the source of truth").

## Verdict

| Target | Verdict | Why |
|---|---|---|
| Included pools, explicit model selected, both pools below 100% | Reliable with caveats | Mapping by model family is documented in official pricing text. Caveats: model list changes, no in-protocol signal, usage-summary is undocumented, polled and can be stale, and Cursor's own UI is known to mislabel the pool |
| Included pools, Auto selected | Heuristic only | Auto's routed model and pool are not exposed. The best guess is "both pools possible", or a poll-delta |
| Included pools, after one pool hits 100% | Heuristic only | Spillover rules are staff-stated, not in official docs, and conflict between individuals and Teams |
| On-demand | Heuristic only | No native flag and no per-request signal in CLI/ACP. Inferences: pool exhaustion plus `enabled`, or `onDemand.used` increase between polls. A per-event `kind`/`isChargeable` signal exists but only via unofficial or admin-only, hourly APIs |

Implication for the RFD: `active` should be optional per pool and not assumed to be filled. Cursor can supply at best a
heuristic `active` for included pools when the model is explicit, and none for on-demand. If the RFD needs an
explicit-confidence marker, Cursor is an example of "inferred by the adapter, not reported by the agent".

## Gaps

- No live test. Whether `agent acp` exposes model selection, or reports the resolved model for Auto, is unknown.
  The ACP docs page lists no model-related fields apart from `model?` in `cursor/task` notifications.
- Spillover direction for Teams and for Enterprise is unresolved (above). Individual-plan spillover comes from forum
  staff replies only, not from Cursor docs.
- Pool membership of `Auto` is not stated in official text. "Auto + Composer" is the dashboard/forum label for
  `autoPercentUsed`, while the docs call it "Cursor Models". UNVERIFIED that these are the same meter on every plan.
- Update cadence of `usage-summary` and of its `onDemand.used` is undocumented.
- Whether the Admin-API `kind` field distinguishes pools (it appears not to) and whether the per-event dashboard
  endpoint still returns `kind` after the 2026-07-31 change (the article only says dollar fields are zeroed) is
  UNVERIFIED.
- The forum threads "Composer 2.5 drains API limits" and "Why does Auto silently fall back to API usage" (replies) were
  not read.
- The codeburn #1545 `isChargeable` observation is from a search summary, not read directly (UNVERIFIED).

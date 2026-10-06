# GitHub Copilot CLI: can the agent know which quota pool is being charged now?

Builds on `.agents/research/copilot-cli.md` and `.agents/research/copilot-cli-overage.md` (not repeated). Re-checked against
`@github/copilot-sdk` 1.0.16 (latest on npm on 2026-10-06; `dist/generated/rpc.d.ts` = `rpc.d.ts`,
`dist/generated/session-events.d.ts` = `se.d.ts`). The `copilot` CLI is closed source, so all runtime behavior is inferred
from SDK types and GitHub docs.

## Summary

- There is NO field anywhere in the SDK types that says "this request was charged to pool X". Neither `assistant.usage`
  nor any other event carries a quota key, a `quota_id`, an overage flag, or a pre/post quota snapshot.
  `assistant.usage` is described as "tokens, costs, quotas, and billing information" (se.d.ts:5795) but has no quota field.
- What exists per request: model id, `cost` (multiplier, experimental), `copilotUsage.totalNanoAiu`, `isAuto`, `isByok`,
  `initiator`. What exists per model: `Model.billing.multiplier`. What exists per account: `quota_snapshots` by category,
  pulled via `account.getQuota` (polling only).
- Category is therefore a deduction, not a signal. For chat-style requests the category is `premium_interactions` by
  convention (UNVERIFIED in the types; consistent with docs: completions are not billed in AI credits, chat/CLI are).
  `chat` and `completions` buckets are not drawn on by CLI model calls under any documentation found.
- Under usage-based billing (`token_based_billing`, the default for most accounts since 2026-06-01) every chat/CLI model
  consumes AI credits; no model is "included/free" (0x). The base/flex/additional-budget ordering is server-side and
  not exposed per request. So "included vs premium model" is only a distinction under legacy request billing.
- Overage cannot be seen per request. It is only observable as a change in `overage`/`overage_count` between two polls, or
  as a failure (`session.error` `quota`/`quota_exceeded`).

## Signals

| signal | meaning | when absent | source |
|---|---|---|---|
| `assistant.usage.model` (required) | model id used for the API call | never absent | se.d.ts:5879 |
| `assistant.usage.cost?` (experimental) | "Model multiplier cost for billing purposes" (legacy premium-request weight) | optional; likely absent/0 under token billing (UNVERIFIED) | se.d.ts:5825-5829 |
| `assistant.usage.copilotUsage?.totalNanoAiu` | "Total cost in nano-AI units for this request" from CAPI `copilot_usage` response field (usage-based billing cost; 1 AI credit = $0.01 per docs) | optional; absent when the server does not return `copilot_usage` (legacy plans, BYOK) UNVERIFIED | se.d.ts:5823, 5925-5935 |
| `assistant.usage.isAuto?` | Auto mode selected for this call (auto is billed at a discount; resolved concrete `model` is separate) | optional | se.d.ts:5863 |
| `assistant.usage.isByok?` | call used a bring-your-own-key provider: NOT charged to any Copilot pool | optional; absent should be read as "not BYOK" (UNVERIFIED) | se.d.ts:5867 |
| `assistant.usage.initiator?`, `interactionType?` | what started the call (sub-agent, sampling, compaction, user). Legacy billing counts only user-initiated prompts; interactionType is documented "Non-billing" | optional; absent for user-initiated | se.d.ts:5847, 5853 |
| `assistant.usage.rte?` | `X-GitHub-Copilot-Request-TE` treatment/eligibility header; `false` also when header absent. Not documented as quota-related | `false` when absent | se.d.ts:5913 |
| `assistant.usage` quota field | does not exist | always | grep of se.d.ts: "quota" appears only at 1616, 1620, 5765, 5795 |
| `Model.billing.multiplier?` | "Billing cost multiplier relative to the base rate" per model in `session.model.list` `list` | optional; for token-billing users, `tokenPrices` (AI credits per batch) and `modelPickerPriceCategory` (`low`/`medium`/`high`/`very_high`) instead | rpc.d.ts:15403-15413, 3524-3535 |
| `Model.billing.discountPercent?`, `promo?` | auto-mode discount; promo discount with `endsAt` | optional; absent for concrete models | rpc.d.ts:15411-15413, 15513-15540 |
| `Model.policy.state` | `enabled`/`disabled`/`unconfigured` (availability, not billing) | optional | rpc.d.ts:15389-15392 |
| `session.model.list.quotaSnapshots?` | "Per-quota snapshots returned alongside the model list, keyed by quota type"; values untyped `JsonValue` | optional, untyped; contents UNVERIFIED | rpc.d.ts:21787-21791 |
| `account.getQuota.quotaSnapshots[type]` | per-category entitlement/used/remaining/overage/reset (pull only, `@experimental`) | keys optional (`AccountQuotaSnapshot \| undefined`); raw passthrough entries may be `null` | rpc.d.ts:6336-6381, 5946-5963 |
| snapshot `token_based_billing?` (per category and account) | category/account uses AI-credit billing rather than premium-request counts | optional | rpc.d.ts:5876, 6020, 6077, 6134 |
| snapshot `has_quota?`, `unlimited?`, `overage_permitted?`, `overage_count?`, `percent_remaining?` | state inputs for deduction | all optional in raw form | rpc.d.ts:6002-6016, 6094-6126 |
| `session.usage.getMetrics` `totalPremiumRequestCost`, per-model `requests.cost`, `totalNanoAiu?` | session-cumulative consumption, per model | `totalNanoAiu` optional | rpc.d.ts:26101, 26226 |
| `session.model_change` (`newModel`, `previousModel`, `cause?`, `source?`, `autoTier?`) | model switch, including automatic ones (`rate_limit_auto_switch`, `refusal_fallback`) | `cause`/`source` optional | se.d.ts:2085-2118 |
| `session.send` `billable?` | "If false, this message will not trigger a Premium Request Unit charge" (default billable) | optional; default billable | rpc.d.ts:19315, 20668 |
| `session.error` `errorType: "quota"`, `errorCode` (`quota_exceeded`, `session_quota_exceeded`, `billing_not_configured`) | request refused for quota reasons; says nothing about which pool | only on failure | se.d.ts:1616-1621 |

## Deduction rules

1. Category. For a Copilot (non-BYOK) chat/CLI model call, treat the charged category as `premium_interactions` (the
   only category whose docs describe model-priced consumption). `chat`/`completions` buckets back IDE chat and inline
   suggestions; completions "are not billed in AI credits and remain unlimited for all paid plans"
   (https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-individuals). Do not mark `chat` or
   `completions` active from CLI model calls. Heuristic: not in any type or doc as an explicit mapping.
2. BYOK. If `assistant.usage.isByok == true`, mark nothing active (no Copilot pool is charged). Confidence: high, from the field's
   own description.
3. Legacy request billing (account `token_based_billing` false/absent). Per-request weight is `cost` (or
   `Model.billing.multiplier` of the used model). A multiplier of 0 means an included model that does not consume the
   allowance; docs: after the allowance runs out "you can still use Copilot with one of the included models for the rest
   of the month" (https://docs.github.com/en/copilot/concepts/billing/copilot-requests). Rule: `multiplier > 0` and
   `has_quota != false` -> `premium_interactions` active; `multiplier == 0` -> no premium pool consumed (the included
   allowance for chat is `chat`, if it is a finite bucket; otherwise nothing finite is active). Only user-initiated
   prompts count (docs: "only the prompts you send are counted", not autonomous tool calls), so per-`assistant.usage`
   attribution is wrong for agentic loops; use `initiator` absent / `interactionType: "conversation-user"` as the
   charge-bearing call. `billable: false` on a send also suppresses the charge.
4. Usage-based billing (`token_based_billing == true`). Every model call carries a nonzero cost, so
   `premium_interactions` (the AI-credit bucket) is active whenever `copilotUsage.totalNanoAiu > 0`. No multiplier applies
   (the pricing page says multipliers appear only in the legacy request-based note,
   https://docs.github.com/en/copilot/reference/copilot-billing/models-and-pricing). Inside that bucket the draw order is
   base credits, then flex allotment, then additional budget
   (https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-individuals). The SDK has no
   base/flex split; whether `entitlement` includes flex is UNVERIFIED.
5. Overage active (derived, polling only). Poll `account.getQuota` (e.g. after each turn / on `assistant.usage`) and
   mark overage active when for the bucket: not unlimited AND (`has_quota == false` OR `remainingPercentage == 0`) AND
   `overage_permitted` AND (`overage` increased since the previous poll). A single snapshot with `overage > 0` only proves
   "overage was used earlier this period", not "now". Without a second poll the correct state is "allowed" or "unknown".
6. Rate-limit / quota errors are not a source of "active": a `quota_exceeded` failure means no pool currently
   serves requests (all inactive), not that a specific pool is active.

## Failure modes

- Before the first request: `assistant.usage` has not fired, so rules 3-4 have no per-call input. Only `account.getQuota`
  (and `session.model.list.quotaSnapshots`, untyped) can be read, and "active" must be predicted from the selected model
  (`session.model.getCurrent` + `Model.billing`). Reasonable default: `premium_interactions` as the expected pool, marked
  as predicted, not observed. Absent entirely in ACP, where no quota surface is documented (see main note).
- Model switch: model change events (`session.model_change`, se.d.ts:2085) fire for user and automatic changes
  (`cause: "rate_limit_auto_switch"`, `"refusal_fallback"`). A switch to a multiplier-0 / included model on legacy billing
  moves "active" off the premium pool; on token billing nothing changes (all models consume credits). The auto model
  (`isAuto`, `billing.discountPercent`) is a synthetic id: the concrete model that served the call is in
  `assistant.usage.model`, but `Model.billing` for the concrete model must be looked up separately; `getCurrent`
  alone may return `auto`. Free/Student plans only have auto selection, so model-based deduction is moot there.
- Monthly reset: 1st of month 00:00:00 UTC (both billing models). Snapshots are not pushed, so the client holds stale
  `usedRequests`/`overage`/`has_quota` until the next poll. Compare `timestamp_utc` / `quota_reset_at` against now and
  refetch after the reset. A stale pre-reset `has_quota == false` would wrongly mark overage active or mark a pool blocked.
- Unlimited plans: `isUnlimitedEntitlement` / `unlimited == true` / `entitlement == -1` mean the bucket is never
  exhausted and overage is moot. Meters can be reported without any "active" meaning. Completions are unlimited on all
  paid plans, so `completions` is never usefully active.
- Absent or partial snapshots: all raw fields optional and entries may be `null`; the typed `AccountQuotaSnapshot` makes
  most fields required but is derived client-side from a closed mapping (UNVERIFIED). `getQuota` is `@experimental` and
  may change shape.
- Policy/org-managed: for Business/Enterprise, allowances are pooled per billing entity
  (https://docs.github.com/en/copilot/reference/copilot-billing/models-and-pricing); user-level snapshots may not show the
  shared pool the draw actually hits. Budgets (cost center, org, enterprise, user-level) are invisible to the CLI.
- Server/client disagreement: `quota_exceeded` 402 reported while flags suggested paid usage was allowed
  (https://github.com/github/copilot-cli/issues/3431), so deductions from flags can be wrong.
- Sub-agents and background work: `assistant.usage` fires for every call (agentId on the event envelope); under legacy
  billing only user-initiated prompts consume premium requests, under token billing all calls consume credits.
- Known counter bug: `/usage` premium counter undercounts (https://github.com/github/copilot-cli/issues/1582), so session
  metrics should not be used to verify the pool.

## Verdict

- Included categories (`premium_interactions`/credits, `chat`, `completions`): HEURISTIC ONLY (leaning "reliable with
  caveats" only for the narrow case of excluding things). Specifically:
  - Reliable: BYOK calls charge no pool (`isByok`); unlimited buckets are never exhausted; completions never draw
    from CLI model calls.
  - Reliable with caveats: under usage-based billing, `premium_interactions` is the pool for every non-BYOK model call,
    confirmed per call by `copilotUsage.totalNanoAiu > 0` when present. The caveat is the missing explicit category in any
    event and that base/flex are not separable.
  - Heuristic only: legacy billing "premium vs included" decisions; they depend on `Model.billing.multiplier` being
    populated and on user-initiated vs agentic attribution.
  - Net: an agent can fill `active` for the main pool with moderate confidence, but only by inference. It must not claim
    it is authoritative; recommend `active` be optional and never required for a pool.
- Overage: HEURISTIC ONLY, and never real-time. There is no per-request overage signal. The best available is the
  delta of `overage`/`overage_count` between two `account.getQuota` polls, stale by the poll interval and by server-side
  aggregation lag (UNVERIFIED how fresh). A failed request (`quota_exceeded`) is a reliable "blocked", not "active".
  Flags like `overage_permitted` indicate "allowed", not "charged now".
- Implication for the RFD: `active` as a boolean is fillable but low-confidence; consider semantics "best effort, may be
  absent" and keep overage's `active` derivable only from state transitions the agent itself observes.

## Gaps

- UNVERIFIED: contents of `session.model.list.quotaSnapshots` (typed `JsonValue`); it may carry the same
  `quota_snapshots` as the user response, which would let model-list refreshes serve as a poll.
- UNVERIFIED: whether `assistant.usage.cost` is populated under token billing; whether `copilotUsage` is absent on legacy
  plans; whether `Model.billing.multiplier` is populated under token billing (docs say no multiplier there).
- UNVERIFIED: which bucket key holds AI credits under usage-based billing (`premium_interactions` assumed from the
  `token_based_billing` flag being present on that entry), and the units of `entitlement`/`overage_count` then. No authenticated
  session was available to capture a payload.
- UNVERIFIED: how the CLI maps raw snapshot to `account.getQuota` (closed source) and how often snapshots refresh server side.
- UNVERIFIED: whether CLI calls ever charge the `chat` bucket (e.g., on Free plan). The Free plan "model access through auto
  model selection only" and the 50/month legacy chat counters were not checked against the CLI.
- UNVERIFIED: the `rte` header semantics (docs/types say only "treatment/eligibility").
- ACP mode: none of these signals is exposed over ACP (`copilot --acp`) per the main note; an ACP-side `active` flag from
  Copilot would require the adapter to use the SDK JSON-RPC (a different surface) or poll `account.getQuota` itself.
- The legacy multiplier table page ("Model multipliers for annual plans on request-based billing") was not fetched; multiplier
  values and the list of 0x models are therefore not recorded.
- Docs read: https://docs.github.com/en/copilot/concepts/billing/copilot-requests,
  https://docs.github.com/en/copilot/concepts/billing/usage-based-billing-for-individuals,
  https://docs.github.com/en/copilot/reference/copilot-billing/models-and-pricing,
  https://docs.github.com/en/copilot/reference/ai-models/supported-models (no billing data). Fetch tool summaries only, so the
  wording of quoted fragments is as returned by the summarizer.

# ACP conventions: collapsed state enum vs boolean plus reason

Scope: this repository only. Paths are relative to the repo root. `S/` = `agent-client-protocol-schema/src/`.
Task: inform the choice for a quota pool availability field between (A) one open enum
`state: available | exhausted | not_enabled | not_permitted | not_eligible | _custom` and (B) `available: boolean` plus
optional open-enum `unavailableReason`.

## Summary

- ACP's dominant idiom for "outcome or status with success and failure values" is ONE open string enum. Success and
  failure values share a field: `StopReason` (`end_turn` next to `refusal`, `cancelled`, `error`), `ToolCallStatus`
  (`completed` next to `failed`, `cancelled`), `PlanEntryStatus`, `CompactionStatus` (`completed`/`failed`/`cancelled`),
  `RequestPermissionOutcome` (`selected`/`cancelled`), `ElicitationAction` (`accept`/`decline`/`cancel`),
  `NesRejectReason`. This favors candidate A.
- ACP also has a boolean-plus-detail idiom, but it is used for plain yes/no facts: `ProviderInfo.required: bool`,
  `SessionConfigBoolean.currentValue`, the `auth/status` RFD (`authenticated: boolean` plus `message?`). The repo has no
  schema example of `bool` plus a separate open-enum reason. Where a reason is attached to a state, it is attached to
  a specific enum VALUE (a sibling field present only for that value, e.g. `error` beside `stopReason: "error"`), not to a boolean.
- Unknown-value handling is uniform: v2 makes every enum-like field open. `_`-prefixed values are implementation
  extensions. Unknown non-underscore values are reserved for future ACP. Receivers SHOULD preserve the value and fall back
  generically. The spec does NOT say "treat unknown as X" for status-like enums. The strongest classification rules are
  negative and safety-biased: "MUST NOT treat as approval" (permission outcome), "MUST NOT treat as a known action"
  (elicitation), "present a generic state without inferring lifecycle or control behavior" (compaction status, notice
  severity). The only "treat as X" is for `format` ("treat it as an annotation"). So with A, a client cannot infer
  "unknown means unavailable" from existing text; the RFD would need to state it, and the safe default is conservative.
- Conditional presence is expressed in the schema by tag-flattening (`#[serde(tag = ...)]` with a payload struct,
  producing JSON Schema `anyOf` branches with `const` discriminators) and in docs by prose such as "belongs to the
  `error` stop reason; beside any other stop reason it is an unknown field".
- Recommendation: A matches ACP conventions better. It needs an explicit rule that only the named usable value means
  usable. B is cleaner for "is it usable?" but is novel here and has a split-brain risk.

## Findings

### 1. Enums that mix success and failure values in one field

All are open in v2 (`Other(String)` or `Other(Payload)` with `#[serde(untagged)]`). In v1 most are closed; the
`NoticeSeverity`, `CompactionStatus`, `IdleStopReason` and elicitation ones are open in v1 draft features.

- `StopReason`, v2: `S/v2/agent.rs:3346-3382`. Success and failures in one tagged field `stopReason`:
  ```rust
  #[serde(tag = "stopReason", rename_all = "snake_case")]
  pub enum StopReason {
      /// The active work ended successfully.            EndTurn,
      MaxTokens, MaxTurnRequests, Refusal, Cancelled,
      /// The active work ended because something failed. Error(ErrorStopReason),
      /// Custom or future stop reason. ... #[serde(untagged)] Other(OtherStopReason),
  }
  ```
  v1 is a closed `Copy` enum with the five non-error values: `S/v1/agent.rs:3176-3205`. The idle snapshot variant that
  adds `Error` and `Other` in v1: `IdleStopReason`, `S/v1/client.rs:1347-1375`, whose doc says "The known values are those
  of StopReason plus `error`".
- `ToolCallStatus`, v2: `S/v2/tool_call.rs:342-372`: `Pending` (default), `InProgress`, `Completed` ("completed
  successfully"), `Failed` ("failed with an error"), `Cancelled`, `Other(String)`. v1 closed with four values:
  `S/v1/tool_call.rs:509-531`. Doc: `docs/protocol/v2/tool-calls.mdx:373`.
- `PlanEntryStatus`, v2: `S/v2/plan.rs:519-541`: `Pending`, `InProgress`, `Completed`, `Cancelled`, `Other(String)`.
  v1 closed with three: `S/v1/plan.rs:525-537`. Migration note: `docs/protocol/v2/migration.mdx:606`.
- `CompactionStatus`, v1 and v2: `S/v1/client.rs:765-790`, `S/v2/client.rs:743-760`: `InProgress`, `Completed`
  ("finished successfully"), `Failed` ("finished unsuccessfully"), `Cancelled`, `Other(String)`.
- `RequestPermissionOutcome`, v2: `S/v2/client.rs:2827-2855`, tag `outcome`: `Cancelled`, `Selected(SelectedPermissionOutcome)`,
  `Other(OtherRequestPermissionOutcome)`. v1 closed with the first two: `S/v1/client.rs:2380-2396`.
- `ElicitationAction`, v2: `S/v2/elicitation.rs:1874-1898`, tag `action`: `Accept(ElicitationAcceptAction)`, `Decline`, `Cancel`,
  `Other(OtherElicitationAction)`. Also open in v1: `S/v1/elicitation.rs:1909-1935`.
- `NesRejectReason`, v2: `S/v2/nes.rs:2934-2958`: `rejected`, `ignored`, `replaced`, `cancelled`, `Other(String)`.
  This is a reason-only enum, with no success value.
- `NoticeSeverity`, v1 `S/v1/client.rs:648-668` and v2 `S/v2/client.rs:625-642`: `Info`, `Warning`, `Error`,
  `Other(String)`. v1 doc text: "Values beginning with `_` are reserved for implementation-specific extensions. Other
  unknown values are reserved for future ACP severities."
- `StateUpdate`, v2: `S/v2/client.rs:1466-1494`, tag `state`: `Running`, `Idle`, `RequiresAction`, `Unknown` (unstable
  `unstable_subagents`, "cannot currently determine foreground activity", struct at `S/v2/client.rs:1714`), `Other`.
  `Unknown` is an explicit, known value that means "cannot tell"; it is separate from the open-enum `Other` fallback.
  Doc at `S/v2/client.rs:1700-1706`: "The Client MUST stop presenting the previous state as confirmed current activity,
  but may retain it as last known".
- Auth: the stable `AuthMethod` is a tagged union of method types (`S/v2/agent.rs:517-540`, `Other(OtherAuthMethod)`)
  and has no state/outcome enum. Auth STATE exists only in the RFD `docs/rfds/get-auth-state.mdx:119-135`, which
  uses a boolean:
  ```ts
  interface AuthStatusResponse {
    /** Whether the agent has credentials configured. true means present (validity not guaranteed). false means none. */
    authenticated: boolean;
    /** Human-readable description of the overall auth state */
    message?: string;
  ```
  The RFD removed richer per-method state on purpose (`docs/rfds/get-auth-state.mdx:235-239`): "keeps `auth/status`
  minimal and reliable: top-level `authenticated` plus optional `message`".

Boolean or presence-marker plus separate detail (v2 stable and unstable):
- `ProviderInfo` (`S/v2/agent.rs:3727-3745`): `required: bool` ("whether this provider is mandatory"), and
  `current: Option<ProviderCurrentConfig>` where "Null or omitted means provider is disabled". So a presence marker
  encodes enabled/disabled, and the detail struct (`apiType`, `baseUrl`) is only present when enabled.
- `SessionConfigBoolean.currentValue: bool` (`S/v2/agent.rs:2227-2239`): plain toggle, no reason.
- `ElicitationAcceptAction.content: Option<...>` (`S/v2/elicitation.rs:1984-1994`): optional payload tied to one
  action value.
- `ErrorStopReason.error: Option<...>` (`S/v2/agent.rs:3384-`): optional failure detail tied to `stopReason: "error"`.
- `MessageMcpResponse` (`S/v2/mcp.rs:58-100`): untagged union, exactly one of `result` or `error` present. This is a
  presence-keyed alternative with no discriminator.
- Only four `bool` fields exist in the v2 schema source (`elicitation.rs:583`, `agent.rs:2238`, `agent.rs:3738`,
  `nes.rs:2766`). None pairs a boolean with a separate open-enum reason.

### 2. Open-enum rules and unknown-value handling

Convention text:
- `docs/protocol/v2/extensibility.mdx:110-122` (verbatim rules): `_` values are implementation extensions; "Unknown
  values that do not begin with `_` are reserved for future ACP variants"; extensions "MUST NOT define custom
  non-underscore values"; implementations "MUST NOT treat unknown non-underscore values as custom extensions". "This rule
  applies only where the schema defines a fallback path. Closed discriminators can still reject unknown values when the
  receiver cannot safely continue without understanding the variant." Then: "SHOULD preserve unknown values ...
  When displaying an unknown value, implementations SHOULD fall back to generic UI behavior that matches the field's purpose."
- `docs/protocol/v2/migration.mdx:16` and `:758`: "Every enum-like string accepts unknown values ... Receivers SHOULD
  preserve unknown values ... and either render them generically or fall back safely. Do not mint non-underscore custom values."
- `docs/protocol/v2/overview.mdx:191`.
- RFD: `docs/rfds/v2/enum-variant-extension.mdx`: opens enums "when the receiving side has an acceptable fallback"
  (lines 14-34, 62-66 of the file). Closed is appropriate "if understanding the variant is required for correctness"
  (FAQ "Does this make every enum extensible?"). "Implementations that cannot act on an unknown value SHOULD degrade
  gracefully according to the field's semantics." Fallback behavior is "not identical for every unknown value".
  It also says "Capabilities are still available when a sender needs to know whether the receiver can act on a value."

Per-field "what to do with unknown" text (the closest things to classification guidance):
- Permission outcome: `S/v2/client.rs:2843-2852`: "Agents that do not understand this outcome MUST NOT treat it as
  approval ... fail or decline the permission request according to policy." Same in `docs/protocol/v2/migration.mdx:564`.
  This is a safety-biased default ("treat unknown as not-approved"), the only precedent that classifies an unknown
  value, and it does so by prohibition.
- Elicitation action: `S/v2/elicitation.rs:1891-1893`: "preserve the raw payload ... They MUST NOT treat it as a known
  elicitation action."
- Elicitation mode: `S/v2/elicitation.rs:1636-1638`: "MUST NOT render it as a known elicitation mode." Property schema:
  `:987-989`.
- String `format`: `S/v2/elicitation.rs:51-52`: "should treat it as an annotation rather than rejecting the schema".
  This is the only "treat unknown as X" in the repo.
- Session update: `S/v2/client.rs:204-206`: "preserve the raw payload ... and otherwise ignore it or display it generically".
- Permission subject: `S/v2/client.rs:2504-2508`: "display a generic permission prompt or decline it according to policy".
- Permission option kind, status, stop reason docs: `docs/protocol/v2/tool-calls.mdx:346` ("preserve the value and use a
  generic permission option treatment"), `:373` ("display a generic progress state"), `docs/protocol/v2/prompt-lifecycle.mdx:519`
  ("display a generic stopped state"), `docs/protocol/v2/agent-plan.mdx:100` ("preserve or display a generic plan-entry state").
- v1 draft compaction status (`docs/protocol/v1/draft/prompt-turn.mdx:395-400`): "Clients preserve unknown strings and
  present a generic state without inferring lifecycle or control behavior." Notice severity
  (`docs/protocol/v1/draft/prompt-turn.mdx:302-306`): "use a generic notice presentation without inferring additional
  behavior from them."
- Config option category: `docs/protocol/v2/session-config-options.mdx:166-170`: "MUST handle missing or unknown
  categories gracefully"; categories are UX-only and "MUST NOT be required for correctness". Unknown option type: `:203`.
- Malformed values: optional fields use `DefaultOnError` and omit-equals-null, e.g. `IdleStateUpdate.stopReason`
  (`schema/v2/schema.json:5003-5020`: "a malformed value is treated the same way [as omitted]"). A bad value of
  an optional enum degrades to absent instead of failing the message.
- Schema form of an open enum: `schema/v2/schema.json:819-853` (`ToolCallStatus`): `anyOf` of `const` strings plus a final
  `{"title": "other", "type": "string"}` branch. Tagged unions get a branch whose discriminator is `string` and
  `not: anyOf [known consts]` (`schema/v2/schema.json:5070-`).

There is NO text anywhere saying "treat an unknown status as failed/unavailable/inactive". The unstated default is
"preserve and render generically; infer nothing".

### 3. Conditional presence precedents

- Stop reason `error` payload: `S/v2/agent.rs:3384-` (`ErrorStopReason`, flattened into the tagged union via
  `#[serde(tag = "stopReason")]` and `Error(ErrorStopReason)`). Schema: `schema/v2/schema.json:5070-5090`, the
  `error` branch has `"allOf": [{"$ref": "#/$defs/ErrorStopReason"}]` so `error` is a property only on that branch. Docs:
  `docs/protocol/v2/prompt-lifecycle.mdx:523-550`: "This stop reason carries the failure in an `error` field beside
  `stopReason`, which the Agent SHOULD include" and "`error` belongs to the `error` stop reason; beside any other stop
  reason it is an unknown field." Unknown fields are tolerated (ignored), not rejected.
- `IdleStateUpdate.stop_reason` is `Option<StopReason>` with `#[serde(flatten)]` (`S/v2/client.rs:1520-1560`):
  an optional tagged value that carries its own conditional fields. A custom schema transform
  (`idle_state_update_schema`, `S/v2/client.rs:1596-`) was needed because flattening an optional tagged union adds an
  empty `{}` `anyOf` branch. Precedent for the cost: conditional fields via flatten need extra schema work.
- Elicitation `accept` carries `content` (`S/v2/elicitation.rs:1984-1994`); permission `selected` carries `optionId`
  (`S/v2/client.rs:2937-2939`).
- `ProviderInfo.current` present only when the provider is enabled (`S/v2/agent.rs:3743-3745`); see section 1.
- Doc idiom for omit-vs-null: "Optional. Omitted or `null` both mean ..." (`S/v2/agent.rs` `ErrorStopReason.error` doc,
  `S/v2/client.rs:1536-1538` for `usage`). This repo's `CLAUDE.md` rule: for any nullable field, state whether it is
  required or optional and whether `null` equals omitted.
- `docs/rfds/get-auth-state.mdx:209-215`: `authenticated: true` "means credentials are present, NOT that they are
  guaranteed to be valid": a boolean with semantics defined by prose.
- Nothing in the repo expresses "field X required if and only if boolean Y is false" in the schema. JSON Schema `if/then`
  is not used (`grep` shows no `"if"` or `"then"` use in the v2 schema; UNVERIFIED beyond a quick grep, not re-run exhaustively).
  Conditional presence is always via discriminated `anyOf`/tagging or plain optional fields documented in prose.

## Implications for the RFD

Candidate A, single open enum `state`:
- Matches the dominant ACP idiom (`ToolCallStatus`, `PlanEntryStatus`, `CompactionStatus`, `StopReason`, `NesRejectReason`).
  Success and failure values in one open string enum is normal. The v2 open form is a `Other(String)` fallback with `_`
  reserved for extensions and non-underscore reserved for ACP. `_custom` in the candidate list is just that convention,
  not a literal value.
- Forward-compat rule needed: the repo gives no default classification for unknown values, and the enum RFD (`docs/rfds/v2/enum-variant-extension.mdx`)
  says to "degrade gracefully according to the field's semantics". The quota RFD must therefore state the semantics:
  "Only `available` means the pool can be used. A client MUST NOT infer usability from any other value, including an
  unknown or `_`-prefixed one, and SHOULD show it generically (e.g. as 'unavailable', plus any provided message)."
  This mirrors the permission outcome rule ("MUST NOT treat it as approval") and the compaction rule ("without inferring
  behavior"). Reverse risk: if a future ACP value meant "usable, with caveats" (e.g. `degraded`, `available_overage`), old
  clients would treat it as unavailable. That is a safe failure, but the RFD should say new "usable" values need a
  capability or a new field. Otherwise keep exactly one usable value.
- Do not have clients default unknown to "exhausted": the RFD should say unknown means "not known to be usable" rather than
  that it is a quota failure.
- Reason granularity is not extensible without a new value, but values are open, so adding `not_entitled` etc. is additive.

Candidate B, `available: boolean` plus optional `unavailableReason`:
- Not contradicted by ACP (booleans exist, `auth/status` RFD uses `authenticated: boolean` plus `message`), but there is no
  precedent in the repo for bool plus open-enum reason. Reasons elsewhere are values of the same enum, with detail as a
  sibling field attached to a value.
- Benefit: the usability bit is a closed, forward-compatible boolean, so unknown reasons cannot flip usability. No need for
  a "unknown means not usable" rule for the bit itself; only the reason is open (and `Other` handling would be "show
  generically").
- Costs and rules needed: (1) state that `unavailableReason` MUST be omitted (or ignored) when `available` is true, and
  whether `null` equals omitted (per `CLAUDE.md` schema rule); (2) consumers must tolerate `available: false` with no
  reason, and an unknown reason; (3) the schema can't enforce the dependency without `if/then` or a tagged-union form (the
  repo's tools are `anyOf` with discriminators; `ErrorStopReason` needed a custom transform for a similar shape), so it
  would be a prose-only constraint; (4) adds two fields to keep consistent, the split-brain risk of `available: true`
  with `unavailableReason` set; (5) the boolean cannot express a future tri-state (e.g. "limited") without a breaking
  change or a second field. If `available` could later gain a third condition, B would force a new field.
- A hybrid that stays within ACP idiom: keep A, and attach optional detail (`message`, reset time) as sibling fields,
  as `error` is a sibling of `stopReason: "error"`.

Bottom line: A is closer to ACP conventions. B is defensible if the RFD prioritizes a hard closed "usable" signal. In both
cases the RFD must define the usability rule for unknown values explicitly, because ACP's general rule only says "preserve and
render generically".

## Gaps

- The auth `AuthStatusResponse` is an RFD, not in the schema (`grep` of `S/v2` finds no `auth/status`; `docs/rfds/get-auth-state.mdx`
  is the only source). Whether it ships as specified is UNVERIFIED.
- I did not exhaustively check schema JSON for `if`/`then` constructs; claim in section 3 is from a limited grep.
- I did not read `schema/v1` JSON line by line; v1 claims come from `S/v1` source and `docs/protocol/v1`.
- `docs/rfds/session-notices.mdx` and `session-compaction.mdx` (the v1 RFD sources of the "without inferring" wording)
  were not read in full; wording is cited from `docs/protocol/v1/draft/prompt-turn.mdx`.
- No Rust deserialization behavior was tested (e.g. whether a bad `state` string in an optional field degrades to absent
  for every type).

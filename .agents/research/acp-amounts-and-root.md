# ACP `Cost` type and root-shape conventions

Scope: this repository only (`agent-client-protocol-schema/src` v1 and v2, `schema/v1`, `schema/v2`, `docs/protocol`,
`docs/rfds`). Paths below are relative to the repo root. `S/` = `agent-client-protocol-schema/src/`.

## Summary

- `Cost` is `{amount: f64, currency: String, _meta?}`. It has one use site: the optional `cost` field of
  `UsageUpdate` (`sessionUpdate: "usage_update"`). It is stable in v1 and v2 (present in `schema.json`, not only in
  `schema.unstable.json`). The feature gate `unstable_session_usage` is gone.
- `amount` is a JSON `number` with `format: double`. No decimal-as-string type, minor-unit field, or amount/unit type
  exists anywhere in the schema. No RFD discusses precision or minor units. Currency is explicit only because agents may
  bill in different currencies; non-currency amounts (credits, tokens, percentages) are not discussed.
- v1 and v2 `Cost` differ in one way. v2 adds `pattern: ^[A-Z]{3}$` on `currency`. v1 has no pattern.
- A new generic `{amount, unit}` type added alongside `Cost` (new fields only) is non-breaking. Removing or retyping
  `Cost.amount`/`currency` is a breaking change for v1 stable. In v2, which is still unstable/draft, it is possible but
  would diverge from the "unchanged" status in the v2 migration guide.
- Root shape: every request params, request result, notification params, and `SessionUpdate` variant in the schema is a
  JSON object. No `$defs` entry has `type: array` at the top level. Lists always live in a named field of an object
  (`availableCommands`, `sessions`, `entries`, `configOptions`). A `usage_update`-style quota update must therefore be an
  object (with `sessionUpdate` discriminator), and a list of quota windows must sit in a named field.

## Findings

### 1. `Cost` definition and use sites

Definition (v1, stable):
- `S/v1/client.rs:1889-1911`: `pub struct Cost { pub amount: f64 /* "Total cumulative cost for session." */, pub currency: String /* "ISO 4217 currency code (e.g., "USD", "EUR")." */, pub meta: Option<Meta> /* _meta */ }`.
  `#[non_exhaustive]`, `rename_all = "camelCase"`. Constructor `Cost::new(amount: f64, currency)` at `S/v1/client.rs:1913-1922`.
- JSON Schema: `schema/v1/schema.json:4194-4215` (stable) and `schema/v1/schema.unstable.json:5749` (identical).
  `amount`: `"type": "number", "format": "double"`. `currency`: `"type": "string"`. `required: ["amount","currency"]`.

Definition (v2):
- `S/v2/client.rs:1819-1842`: same fields, plus `#[schemars(pattern(r"^[A-Z]{3}$"))]` on `currency` (`S/v2/client.rs:1829-1830`).
  Schema: `schema/v2/schema.json:5737-` (the `pattern` appears at about line 5749) and `schema/v2/schema.unstable.json:7459`.
  `amount` is still `number`/`double` (`schema/v2/schema.unstable.json:7466`).

Use sites (exhaustive; `grep -w Cost` over src, schema, docs):
- `UsageUpdate.cost: Option<Cost>`: `S/v1/client.rs:1845`, `S/v2/client.rs:1418`. Declared with
  `DefaultOnError` + `#[serde(default)]`, so a malformed `cost` deserializes to `None` rather than failing the update
  (`x-deserialize-default-on-error`). Schema: `schema/v1/schema.json:4236`, `schema/v2/schema.json:5780`.
  Field doc is "Cumulative session cost (optional)."
- `UsageUpdate` is the `SessionUpdate::UsageUpdate` variant, `sessionUpdate: "usage_update"`: `S/v1/client.rs:145`,
  `S/v2/client.rs:150`.
- Tests: `S/v1/client.rs:5322-5331` and `S/v2/client.rs:4395-4404` use `Cost::new(0.045, "USD")`.
- Nothing else references `Cost`. The unstable end-turn `Usage` struct (`S/v1/agent.rs:3212`) has only `u64` token
  counts and no cost.

Stability:
- Stable. `docs/updates.mdx:98-106` and `docs/announcements/session-usage-stabilized.mdx` (published June 5, 2026) say
  `usage_update` notifications are stabilized. `CHANGELOG.md:205` says "Stabilize session usage updates (#1371)".
  Earlier unstable gate: `CHANGELOG.md:416` (#454). `S/v1/client.rs` has no `cfg(feature = "unstable_session_usage")`
  now. `unstable_session_usage` appears in no `Cargo.toml`.
- v2 migration guide lists `usage_update` as "Unchanged": `docs/protocol/v2/migration.mdx:76`.
- Protocol docs: `docs/protocol/v1/prompt-turn.mdx:214-235`, `docs/protocol/v2/prompt-lifecycle.mdx:357-378`.
  Text: "`cost` is optional and, if present, `amount` and `currency` are required. `currency` is an ISO 4217 currency code
  like "USD"." Generated reference: `docs/protocol/v1/schema.mdx:2585-2604`.

RFDs that mention cost:
- `docs/rfds/session-usage.mdx` (Completed; author @ahmedhesham6, champion @benbrandt).
  - Field spec at lines 69-76: `cost` (object, optional, nullable); `amount` (number, required, "Total cumulative cost
    for the session"); `currency` (string, required, ISO 4217).
  - Design principle 4, line 114: "Cost is optional because not all agents track it, and the currency is explicit because
    agents may bill in different currencies."
  - FAQ "Why not assume USD for cost?" (line 187): European agents might bill in EUR, Asian in JPY or CNY, rates change.
    "Agents should report the actual billing currency and let clients convert."
  - Line 147: "Add a `Cost` type with `amount` and `currency` fields." Line 148 gated it behind `unstable_session_usage`
    (no longer applicable).
  - Rate limits and quotas are explicitly out of scope: "Rate limits and quotas are separate concerns that could be
    addressed in a future RFD." (FAQ "What about rate limits and quotas?")
  - Cost is cumulative session state, not per-turn (FAQ "Why is cost in `session/update`...").
- `docs/rfds/end-turn-token-usage.mdx` (Draft). It does not use `Cost`. Line 101, open question 5: "Should any per-turn
  cost estimate exist here, or should cost remain exclusively cumulative session state in `usage_update`?" Lines 14 and
  145-147 keep token accounting separate from `usage_update`.
- `docs/rfds/subagents.mdx:843-869`: "Cost reporting without inferred aggregation". It keeps `usage_update.cost`
  optional and non-additive. "Missing child cost is unknown, not zero." Line 869: it adds no cost-scope field.
  Line 977: "Per-thread token usage is not a monetary cost report."
- `docs/rfds/v2/*`: no mention of cost or `usage_update`. `docs/rfds/mcp-over-acp.mdx` matched only the
  `currency`/`ISO 4217` grep as a false positive (no cost semantics), so ignore it.

Precision, minor units, non-currency amounts:
- No document discusses floating-point precision, decimal strings, minor units, or non-currency (credits, requests,
  percent) amounts for `Cost`. The only precision remark in the repo is about JSON-RPC ids:
  `S/rpc.rs:20` "Fractional parts may be problematic, since many decimal fractions cannot be represented exactly as
  binary fractions" (quoting JSON-RPC about `id`). It is not about `Cost`.
- `Cost.amount` doc says "Total cumulative cost for session", so the type name and field docs are session-cost
  specific. Reusing it for a quota amount would also carry that doc text.

Existing decimal/amount/unit types:
- None. A search for `decimal`, `unit`, `minor unit`, `bignum`, `rust_decimal` over `S/` and `schema/*/schema.unstable.json`
  finds nothing. Numeric JSON formats used in v2 unstable are only `int32`, `int64`, `uint16`, `uint32`, `uint64`,
  `double`, plus string formats `uri` and `date-time`. `double` appears for `Cost.amount`, content `priority`, and
  elicitation `minimum`/`maximum`/`default` (`schema/v1/schema.unstable.json:719,1864-1874,5756,8928`).
- Elicitation number values are `double` (`schema/v1/schema.unstable.json:8928`), so a decimal-as-string would be a new
  convention.

### 2. Root shape: objects always

- JSON-RPC envelope: `S/rpc.rs:52-60` `Request<Params>` has `params: Option<Params>`. `S/rpc.rs:95-108`
  `Response::Result { result: Result }`. `S/rpc.rs:119-126` `Notification<Params>`. The schema is generic, but every
  concrete Params/Result type is an object (below). The Rust code does not enforce "object" on its own.
- Schema scan (script over `schema/{v1,v2}/schema{,.unstable}.json`):
  - All `$defs` entries carrying `x-method` (requests and responses) have `type: object`. The only exception is
    `MessageMcpResponse` (unstable, `anyOf` Result/Error carrier objects, which wraps an opaque MCP result).
  - No `$defs` entry has top-level `type: array`.
  - Notification defs (`SessionNotification` in v1 / `UpdateSessionNotification` in v2 with `x-method: "session/update"`,
    `CancelNotification`, `DidOpenDocumentNotification`, ...) are objects. v2 `UpdateSessionNotification` has
    properties `sessionId`, `update`, `_meta`.
- `SessionUpdate` is an internally tagged union on `sessionUpdate`:
  - v1: `S/v1/client.rs:100-182`, `#[serde(tag = "sessionUpdate", rename_all = "snake_case")]` plus
    `discriminator: {propertyName: "sessionUpdate"}`. Every variant payload is an object type (`ContentChunk`,
    `ToolCall`, `Plan`, `AvailableCommandsUpdate`, `UsageUpdate`, ...).
  - v2: `schema/v2/schema.unstable.json` `SessionUpdate.anyOf` has 24 entries. Each is `type: object` with
    `sessionUpdate: {const: ...}` required. The last has `title: "other"`: "Custom or future session update", per
    `docs/rfds/v2/enum-variant-extension.mdx` (unknown non-`_` values are reserved for ACP; `_` is the extension
    namespace; line 43: opened tagged unions must keep the discriminator and the unknown object payload).
  - Every payload also carries `_meta: object | null` with `x-deserialize-default-on-error`.
- JSON-RPC constraints mentioned in repo docs:
  - The protocol "follows the JSON-RPC 2.0 specification" (`docs/protocol/v1/overview.mdx:10`,
    `docs/protocol/v2/overview.mdx:10`).
  - v2 batch rules, where a batch is an array, are only at the transport level: `docs/protocol/v2/transports.mdx:44-70`
    and `docs/protocol/v2/overview.mdx:15`. This is not about message payloads.
  - `docs/rfds/mcp-over-acp.mdx:198`: inner `params` "accepts an object or `null`... positional arrays are invalid."
    This is the only place the repo states an "object-only params" rule, and it is for the MCP bridge.
  - The repo does not cite the JSON-RPC rule that `params` must be a structured value (object or array). It does not
    cite any restriction on `result`. UNVERIFIED: no repo doc says "result may not be a bare array". The rule in
    practice is a convention of the schema.
- How list-carrying updates and results are shaped (all named array field inside an object, all with
  `DefaultOnError<VecSkipError<_>>`, schema extension `x-deserialize-default-on-error` and
  `x-deserialize-skip-invalid-items`, so a malformed item is skipped instead of failing the whole message):
  - `AvailableCommandsUpdate.availableCommands: Vec<AvailableCommand>`: `S/v1/client.rs:2028-2032`,
    `S/v2/client.rs:2156-2160`.
  - `ListSessionsResponse.sessions: Vec<SessionInfo>` (+ `nextCursor`): `S/v1/agent.rs:1608-1612`,
    `S/v2/agent.rs:1774-1778`. Docs: `docs/protocol/v1/session-list.mdx:130`, `docs/protocol/v2/session-list.mdx:109`.
  - `Plan.entries: Vec<PlanEntry>`: `S/v1/plan.rs:33-40`. v2 `PlanUpdate.entries` (plus `planId`): `S/v2/plan.rs:237`.
    Docs: `docs/protocol/v1/agent-plan.mdx:45`, `docs/protocol/v2/agent-plan.mdx:63`. Plan updates replace the full list.
  - `ConfigOptionUpdate.configOptions: Vec<SessionConfigOption>`: `S/v1/client.rs:1724-1728`, `S/v2/client.rs:1291-1295`.
  - Config select `options` are "a flat array of values or an array of groups; values and groups are not mixed":
    `docs/protocol/v1/session-config-options.mdx:108-109`.
- Collections precedent in v2: terminal auth `env` changed from an object map to an array of `{name, value}` entries
  with unique names: `docs/protocol/v2/migration.mdx:230`. v2 also made support markers objects, not booleans, so they
  "grow fields ... without another breaking change": `docs/protocol/v2/migration.mdx:179-183`. `BTreeMap` fields exist
  only for open-ended unknown-field capture (`S/v2/content.rs:142`, `S/v2/elicitation.rs:756,1128,1666,1918`), and
  `ElicitationSchema.properties` is a map (`S/v2/elicitation.rs:1128`).

## Implications for the RFD

Cost and amount type:
- `UsageUpdate` already carries session-cost semantics, "cumulative" per `Cost.amount` doc and
  `docs/rfds/session-usage.mdx`. It does not model windows, reset times, or non-money units. Quota reporting should not
  overload `Cost`, because its field docs and the RFD say "cumulative session cost".
- New generic `{amount, unit}` type next to `Cost` (new optional field(s) on an update or new update variant):
  non-breaking. `UsageUpdate` and `SessionUpdate` are `#[non_exhaustive]`, v1 `SessionUpdate` is internally tagged and
  adding variants is additive, and `cost` already tolerates malformed values (`DefaultOnError`). Old clients ignore
  unknown fields. UNVERIFIED for v1 clients that validate `SessionUpdate` strictly with the generated schema: adding a
  new `sessionUpdate` value is additive in the schema, but v1 closed unions can reject unknown variants (see
  `docs/rfds/v2/enum-variant-extension.mdx:19-23`). A new field on an existing `usage_update` object is safer than a
  new v1 variant for old validators. In v2, the `other` fallback variant exists, so a new variant degrades gracefully.
- Replacing `Cost` with `{amount, unit}`: breaking for v1 stable (field names/types are in the stable schema and
  docs: `docs/protocol/v1/prompt-turn.mdx:235`, `docs/protocol/v1/schema.mdx:2585`). For v2, `usage_update` is documented
  as "Unchanged" in the migration guide (`docs/protocol/v2/migration.mdx:76`), so replacing it would need a migration
  entry. Complementing `Cost` (keeping `cost` as is, adding a new type for quota amounts) avoids both.
- Representation: all numbers in the schema are JSON numbers (`double`, `uint64`, ...). There is no string-decimal
  precedent, so a decimal string for `amount` would be a first. `f64` precision is not discussed in any RFD.
  Quota amounts are mostly integral or percentage values, so `number` is consistent with `Cost` and `UsageUpdate.used/size`
  (`u64`). Whether to use string-decimal is a design call (UNVERIFIED which clients would accept it).
- If the new type should allow non-currency units, v2's `^[A-Z]{3}$` pattern on `Cost.currency` shows how currency
  codes are constrained, so a free-form `unit` string would need its own rule. ACP's precedent is the `_` prefix for
  custom values (`docs/rfds/v2/enum-variant-extension.mdx:31-38`).
- Reuse note: `currency` is required on `Cost`. A union of `Cost`-like and `{amount, unit}` would need a discriminator or
  `anyOf`. Weak precedent only: `CHANGELOG.md:614` says Rust enums are "either newtype or struct variants within a single
  enum, not mixed". No repo text states a preference between tagged and untagged unions for JSON payloads (UNVERIFIED).

Root shape:
- A quota notification must be an object. The natural slot is a new `SessionUpdate` variant (object with
  `sessionUpdate: "..."`) or new optional fields on `UsageUpdate`. Do not make `params` or `result` a bare array.
- A list of quota windows/buckets goes in a named array field, following `availableCommands` / `sessions` / `entries` /
  `configOptions`, deserialized with `DefaultOnError<VecSkipError<_>>` so one bad entry does not drop the update.
- For a full-replace snapshot, the precedent is `Plan.entries` ("agent must send a complete list ... client replaces",
  `S/v2/plan.rs:231-236`). For incremental update semantics there is no precedent in lists of this kind.
- Prefer an array of objects with a key field (like `env: [{name, value}]` in v2, `docs/protocol/v2/migration.mdx:230`) over a
  map keyed by window id. This keeps the shape open to new fields and avoids map-key conventions.
- Every object should carry `_meta` (`Meta`, `x-deserialize-default-on-error`), and item types should be
  `#[non_exhaustive]` like `Cost`.

## Gaps

- No discussion of float precision or minor units for `Cost` exists in the repo; any rationale beyond the RFD FAQ
  (currency explicit) is unavailable here (the RFD discussion thread on GitHub was not checked).
- The repo does not state a rule that JSON-RPC `result`/`params` must be an object. The conclusion is derived from the
  schema scan, not from a cited normative statement. JSON-RPC 2.0 itself (external) restricts `params` to object or
  array. Not re-verified here.
- Behavior of third-party v1 clients when they meet an unknown `sessionUpdate` value or unknown fields in `usage_update`
  is UNVERIFIED (see `docs/rfds/v2/enum-variant-extension.mdx` for the stated risk). The Rust types use `DefaultOnError`
  for known fields only.
- `schema/v1|v2/src/lib.rs` are thin re-exports; the type definitions live in `agent-client-protocol-schema/src`.

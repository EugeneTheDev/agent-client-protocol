# ACP capability-gating conventions (v1 and v2)

Paths are relative to the repo root. `S/` = `agent-client-protocol-schema/src/`. Line numbers were read at the current
HEAD of branch `eugenethedev/rfd-rate-limits-quotas`.

## Summary

- Capabilities are **presence-marked objects**, not booleans. The convention is `Option<XCapabilities>` where omitted or
  `null` means "not advertised" and `{}` means "supported". The marker struct holds only `_meta`. Older fields
  (`loadSession`, `terminal`, `fs.readTextFile`, `auth.terminal`) are still plain booleans (legacy).
- Agent capabilities gate **client-to-agent request methods**. They live in `InitializeResponse`: v1 at
  `agentCapabilities.sessionCapabilities.*`, v2 at `capabilities.session.*`.
- Client capabilities gate **agent-to-client session updates and notifications**. In v1 they live at
  `clientCapabilities.session.*` (`compaction`, `notices`, `configOptions`) or top-level (`plan`, `subagents`,
  `elicitation`). In v2 they are mostly dropped, because v2 `SessionUpdate` has an `Other` fallback.
- No existing capability gates "agent-to-client notification declared by the agent". `usage_update`,
  `session_info_update` and `config_option_update` are **ungated** in both versions. A `quota_update` gated by an agent
  capability would be a new pattern. The closest precedents are the NES agent capabilities. They gate client-to-agent
  `document/*` notifications and `nes/*` requests, and are declared by the agent (`S/v1/nes.rs:181-261`, `295-415`).
- Not every method takes `sessionId`. Session-less methods: `initialize`, `authenticate`/`auth/login`, `logout`/
  `auth/logout`, `providers/*`, `session/new`, `session/list`, `nes/*`, `document/*` (partly), and `$/cancel_request`.
  `session/list` and `providers/*` are the best templates for a global `quota/read`.
- Method naming is `noun/verb` or `noun/noun_phrase`, lower-case with `snake_case` after the slash
  (`session/set_config_option`, `terminal/wait_for_exit`, `fs/read_text_file`). NES uses `document/didOpen` (camelCase),
  which is an outlier.
- Open enums are `snake_case` known variants plus `#[serde(untagged)] Other(String)`. `_` is the extension namespace and
  non-underscore unknowns are reserved for future ACP. Timestamps are RFC 3339 / ISO 8601 strings with `format:
  date-time`. There are **no** duration fields in the protocol. Only `*_ms` epoch timestamps exist (NES).

## Findings with file:line

### 1. Where capabilities live, shape, absent-capability wording

Initialize envelope:

| Version | Client caps | Agent caps | Source |
|---|---|---|---|
| v1 | `InitializeRequest.clientCapabilities: ClientCapabilities` | `InitializeResponse.agentCapabilities: AgentCapabilities` | `S/v1/agent.rs:56-60`, `136-140` |
| v2 | `InitializeRequest.capabilities: ClientCapabilities` | `InitializeResponse.capabilities: AgentCapabilities` | `S/v2/agent.rs:61-65`, `129-133` |

Notes:
- Field rename between versions: `clientCapabilities`/`agentCapabilities` (v1) became `capabilities` (v2).
- Both are tolerant on the wire. `DefaultOnError` plus `#[serde(default)]` makes a malformed capability deserialize as
  default, and a bare `false` is accepted (test fixtures `"agentCapabilities": false` at `S/v1/agent.rs:5254-5264`).

**v1 agent capabilities** (`S/v1/agent.rs:3810-3882`):
- `loadSession: bool` (legacy boolean).
- `promptCapabilities`, `mcpCapabilities`, `auth` are non-optional structs with defaults.
- `sessionCapabilities: SessionCapabilities` (non-optional, defaults to empty).
- `providers: Option<ProvidersCapabilities>` (`unstable_llm_providers`, `S/v1/agent.rs:3836-3848`) and
  `nes: Option<NesCapabilities>` (`unstable_nes`, `S/v1/agent.rs:3850-3861`).

**v1 SessionCapabilities** (`S/v1/agent.rs:4036-4105`):
- `list`, `delete`, `additionalDirectories`, `fork` (unstable), `resume`, `close`. Each is
  `Option<SessionXCapabilities>`.
- Example doc comment (`list`, `S/v1/agent.rs:4038-4044`): "Optional. Omitted or `null` both mean the agent does not
  advertise support. Supplying `{}` means the agent supports listing sessions."
- Marker struct, e.g. `SessionListCapabilities` (`S/v1/agent.rs:4195-4233`), `SessionForkCapabilities` (4322-4362),
  `SessionResumeCapabilities` (4364-4402), `SessionCloseCapabilities` (4404-4444). Each has only `_meta`. The doc line is
  "Supplying `{}` means the agent supports ...".
- The doc comment states the baseline (`S/v1/agent.rs:4020-4027`): "all Agents **MUST** support `session/new`,
  `session/prompt`, `session/cancel`, and `session/update`. Optionally, they **MAY** support other session methods and
  notifications by specifying additional capabilities." Note the phrase "other session methods **and notifications**".

**v2 agent capabilities** (`S/v2/agent.rs:4123-4190`):
- `session: Option<SessionCapabilities>`, `auth: Option<AgentAuthCapabilities>`, `providers`, `nes`, `positionEncoding`.
  All are `Option`.
- `session` doc (`S/v2/agent.rs:4125-4133`): "Omitted or `null` both mean the agent does not support the `session/*`
  method surface. Supplying `{}` means the agent supports the baseline session methods: `session/new`, `session/prompt`,
  `session/cancel`, and `session/update`."
- `SessionCapabilities` (`S/v2/agent.rs:4332-4393`) keeps `prompt`, `mcp`, `delete`, `additionalDirectories`, `fork`.
  `list`/`resume`/`close` markers were removed. Those methods became baseline (`S/v2/agent.rs:4318-4323`;
  `docs/rfds/v2/required-session-methods.mdx:26-56`).
- v2 baseline is "if you advertise `session`, you MUST support list/resume/close".
- v2 `auth` caveat (`S/v2/agent.rs:4135-4143`): `capabilities.auth` does not advertise `auth/login`/`auth/logout`. Those
  are advertised by a non-empty `authMethods`.
  `docs/protocol/v2/draft/initialization.mdx:82` says "If `authMethods` is omitted or empty, Clients **MUST NOT** call
  either method."

**Client capabilities**:
- v1 `ClientCapabilities` (`S/v1/client.rs:3303-3420`):
  - `fs: FileSystemCapabilities` (booleans `readTextFile`, `writeTextFile`; `S/v1/client.rs:3869`).
  - `terminal: bool`.
  - `session: Option<ClientSessionCapabilities>` (`S/v1/client.rs:3318-3325`).
  - `subagents`, `plan` (both unstable).
  - `auth: AuthCapabilities`, `elicitation: Option<ElicitationCapabilities>`, `nes`, `positionEncodings`.
- v1 `ClientSessionCapabilities` (`S/v1/client.rs:3560-3597`): `compaction: Option<CompactionCapabilities>` (unstable),
  `configOptions: Option<SessionConfigOptionsCapabilities>`, `notices: Option<NoticeCapabilities>` (unstable).
- v2 `ClientCapabilities` (`S/v2/client.rs:2988-3042`) is slimmer: only `auth`, `elicitation`, `nes`,
  `positionEncodings`. It has no `fs`, `terminal`, `session`, `plan`, `subagents`, `notices` or `compaction` fields.
  See `docs/rfds/v2/client-filesystem-terminal-capabilities.mdx` for the removal rationale (UNVERIFIED, not read).

**Absent-capability wording** (normative, copy these patterns):
- Global rule, v1 `docs/protocol/v1/draft/initialization.mdx:115-117`: "All capabilities included in the `initialize`
  request are **OPTIONAL**... Clients and Agents **MUST** treat all capabilities omitted in the `initialize` request as
  **UNSUPPORTED**." v2 is identical at `docs/protocol/v2/draft/initialization.mdx:106-108`.
- Same text, line 121 (v1) and 113 (v2): "Capabilities may specify the availability of protocol methods, notifications, or
  a subset of their parameters. They may also signal behaviors..."
- Agent method gated, client obligation (`docs/protocol/v1/draft/session-list.mdx:37,54`): "Before attempting to list
  sessions, Clients **MUST** verify that the Agent supports this capability by checking the `sessionCapabilities.list`
  field... If `sessionCapabilities.list` is not present, the Agent does not support listing sessions and Clients **MUST
  NOT** attempt to call `session/list`."
- `docs/protocol/v1/draft/session-delete.mdx:35,52`: "If `sessionCapabilities.delete` is omitted or `null` ... Clients
  **MUST NOT** attempt to call `session/delete`. Supplying `{}` means the Agent supports the method."
- Resume/close: `docs/protocol/v1/draft/session-setup.mdx:235-236` and `310-311` ("... Clients **MUST NOT** attempt to
  call `session/resume`").
- `loadSession` boolean variant: "`loadSession` is `false` or not present" (`session-setup.mdx:106`).
- Request docstrings carry the gate too: `S/v1/agent.rs:1287` ("Only available if the Agent supports the
  `sessionCapabilities.resume` capability"), `1447`, `1539`, `1665`, and the `AgentRequest` variants at `4875-4905`
  ("This method is only available if the agent advertises ...").
- Providers: `docs/rfds/custom-llm-endpoint.mdx:332` ("agents that support provider methods MUST advertise
  `agentCapabilities.providers: {}` ... Clients SHOULD only call `providers/*` when this capability is present").

### 2. One capability gating several things; capability gating a notification

**Client capability gates agent notification (session update)**. Template for "Agent MUST NOT send ... unless":
- `notices`:
  - Field: `ClientSessionCapabilities.notices: Option<NoticeCapabilities>` (`S/v1/client.rs:3578-3592`). Doc: "Optional.
    Omitted or `null` both mean the client does not advertise support. Supplying `{}` means the client can present
    notices to the user."
  - Marker: `pub struct NoticeCapabilities {}` (`S/v1/client.rs:3678-3694`). It has no `_meta`, unlike most agent
    markers.
  - Update variant: `SessionUpdate::Notice(Notice)` (`S/v1/client.rs:147-155`), documented "Agents MUST only send this
    update when the Client advertised [`ClientSessionCapabilities::notices`]."
  - Payload: `Notice` (`S/v1/client.rs:683-742`), repeating the MUST.
  - Prose: `docs/protocol/v1/draft/prompt-turn.mdx:272-279`: "The Agent **MUST NOT** send `notice` updates unless the
    Client advertised support by supplying `clientCapabilities.session.notices: {}`... Omitting `session` or `notices`,
    or setting either field to `null`, means the Client does not advertise support. When the capability is absent, the
    Agent may use an ordinary agent message instead."
  - Table row: `docs/protocol/v1/draft/prompt-turn.mdx:32`.
  - Config entry: `docs/protocol/v1/draft/initialization.mdx:219-226`.
  - RFD: `docs/rfds/session-notices.mdx` ("Version-specific capability" section; FAQ "Why is the capability required in
    v1?").
- `compaction`: `CompactionCapabilities {}` (`S/v1/client.rs:3663`). One capability gates two updates:
  `CompactionUpdate` and `CompactionSummaryChunk` (`S/v1/client.rs:157-175`, both "Agents MUST only send this update
  when the Client advertised `ClientSessionCapabilities::compaction`"). Docs: `initialization.mdx:201-207` ("Agents must
  not send these updates to a v1 Client that does not advertise it"). RFD `session-compaction.mdx:161-183`.
- `plan` (`PlanCapabilities`): one capability gates both `plan_update` and `plan_removed` (doc at `S/v1/client.rs`
  ClientCapabilities `plan` field, "Supplying `{}` means the client can receive both update types").
- `subagents` (`SubagentCapabilities`, `S/v1/client.rs:3510-3552`): "The client must advertise this capability before the
  agent sends subagent updates or session-directed messages". One marker gates several updates.
- `configOptions.boolean` and `elicitation.{form,url}` are nested sub-capabilities that gate a *value variant* or
  *mode* of a request, not a whole method.

**v2 drops the client capability for notices and compaction**. `SessionUpdate::Notice` carries the doc "No Client
capability is required. Clients that do not understand or present notices may ignore them." (`S/v2/client.rs:153-160`,
`Notice` at `S/v2/client.rs:649-651`). The mechanism is the v2 open-union fallback `SessionUpdate::Other(OtherSessionUpdate)`
(`S/v2/client.rs:196-208`, struct at `1146`), with doc "Receivers that do not understand this update type should preserve
the raw payload ... and otherwise ignore it or display it generically." Rationale in `docs/rfds/session-notices.mdx`
("V2 does not add or require a capability... V1 keeps its closed `SessionUpdate` union").

**Agent capability gates client-to-agent notifications and requests** (closest precedent for an agent-declared quota
capability):
- `AgentCapabilities.nes: Option<NesCapabilities>` (`S/v1/agent.rs:3850-3861`): "Omitted or `null` both mean the agent
  does not advertise support for NES methods."
- `NesCapabilities.events.document.{didOpen,didChange,didClose,didSave,didFocus}` are `Option<...Capabilities>` markers
  (`S/v1/nes.rs:295-415`) that gate the client-to-agent notifications `document/didOpen` etc.
- `nes/start|suggest|accept|reject|close` are gated by `nes` itself.
- Net: one agent capability object gates both requests and notifications. Each leaf is `Option<{ _meta }>`.

**Observed pattern for agent-to-client session updates with no capability**: `usage_update`, `session_info_update`,
`config_option_update`, `available_commands_update`, `current_mode_update` have no gate in v1 or v2
(`S/v1/client.rs:120-143`; `S/v2/client.rs:140-158`). They were added as a baseline. The `usage_update` shape is
`used: u64`, `size: u64`, `cost: Option<Cost { amount: f64, currency: String (ISO 4217) }>` (`S/v1/client.rs:1836-1911`).
RFD: `docs/rfds/session-usage.mdx`. Any `quota_update` would sit next to it.

### 3. Session-scoped vs non-session-scoped methods, naming

- Not every method takes `sessionId`. From the `x-method` tags in `S/v1/agent.rs`:
  - **No `sessionId`**:
    - `initialize` (`:4750`)
    - `authenticate` (v1) / `auth/login` (v2 `:5236`) and `logout` (v1) / `auth/logout` (v2 `:5266`)
    - `providers/list|set|disable` (`S/v1/agent.rs:4755-4761`). `ListProvidersRequest` has only `_meta`
      (`S/v1/agent.rs:3485-3510`), so it has no params.
    - `session/new` (creates the id)
    - `session/list` (`ListSessionsRequest { cwd?, cursor?, _meta }`, `S/v1/agent.rs:1544-1565`; "Must be an absolute
      path")
    - `nes/*` and `document/*` (NES has its own `NesSessionId`; UNVERIFIED in detail)
    - `$/cancel_request` (`S/v1/protocol_level.rs:73`)
  - **With `sessionId`**: `session/load`, `session/resume`, `session/close`, `session/delete`, `session/prompt`,
    `session/cancel`, `session/set_mode`, `session/set_config_option`, `session/fork`, `session/request_permission`
    (agent-to-client), and `session/update` (notification, `SessionNotification.sessionId`, `S/v1/client.rs:54-57`).
- Client-side methods (agent to client): `session/request_permission`, `fs/*`, `terminal/*`, `elicitation/create`
  (`S/v1/client.rs:3978-3998`).
- v2 renames the notification wrapper to `UpdateSessionNotification` (`S/v2/client.rs:51-81`), same method
  `session/update`.
- Naming (CLAUDE.md, "Adding new methods"): protocol method `noun/verb` maps to Rust trait method `verb_noun`, request
  and response structs `VerbNounRequest` / `VerbNounResponse`, method-names struct field `noun_verb`. Example in the
  repo: `session/list` has `ListSessionsRequest`, `ListSessionsResponse`, and constant `SESSION_LIST_METHOD_NAME`.
  `providers/list` has `ListProvidersRequest`. For `noun/noun`, e.g. `terminal/output`, it maps to `terminal_output`
  and `TerminalOutputRequest`. A `quota/read` request would map to `read_quota`, `ReadQuotaRequest` and
  `ReadQuotaResponse`, with the constant `QUOTA_READ_METHOD_NAME`. A response `x-method` tag is added via
  `schemars(extend("x-side" = "agent", "x-method" = ...))`. Docs: add to `src/bin/generate.rs` SideDocs, run
  `npm run generate` and `npm run check`.
- Notification constant naming: `SESSION_UPDATE_NOTIFICATION`, `ELICITATION_COMPLETE_NOTIFICATION` (suffix
  `_NOTIFICATION`, `S/v1/client.rs:3978,3998`) versus `_METHOD_NAME` for requests.
- Non-NES method segments are lower snake_case. Method names starting with `_` are reserved for custom extensions
  (`docs/protocol/v1/draft/extensibility.mdx:47`).
- Existing sample update names are `snake_case` discriminators on `sessionUpdate` (`#[serde(tag = "sessionUpdate",
  rename_all = "snake_case")]`, `S/v1/client.rs:101-102`). `quota_update` fits the pattern.
- Unstable gating mechanism in the Rust crate: Cargo feature `unstable_<name>` plus `#[cfg(feature = ...)]`, listed in
  the `unstable` umbrella (`agent-client-protocol-schema/Cargo.toml` `[features]`). v2 is separate
  (`unstable_protocol_v2`, not in the umbrella). Doc comments carry the "**UNSTABLE** This capability is not part of the
  spec yet" banner.

### 4. Open-enum, timestamp, duration conventions

**Open enums**:
- `NoticeSeverity` (`S/v1/client.rs:649-668` and `S/v2/client.rs:622-641`): `#[serde(rename_all = "snake_case")]` with
  `Info`, `Warning`, `Error`, and `#[serde(untagged)] Other(String)`. Doc: "Values beginning with `_` are reserved for
  implementation-specific extensions. Other unknown values are reserved for future ACP severities." Same in both
  versions "from its introduction" (`docs/rfds/session-notices.mdx`).
- Other `Other(String)` enums:
  - v1: `CompactionStatus` (`S/v1/client.rs:786`), `SessionConfigOptionCategory` (`S/v1/agent.rs:2321`), another at
    `S/v1/agent.rs:3327`.
  - v2: `CompactionStatus` (`S/v2/client.rs:757`), `PermissionOptionKind` (`:2777`), `SessionConfigKind` (`S/v2/agent.rs:2277`),
    `StopReason` (`:3638`), `PlanEntryStatus`/`PlanEntryPriority` (`S/v2/plan.rs:511,537`), `ToolKind`/`ToolCallStatus`
    (`S/v2/tool_call.rs:368,648,671`), `PositionEncodingKind`, NES enums, `NesTriggerKind`, and content annotation
    enums (`S/v2/content.rs:793,890`).
  - Elicitation string format (`S/v2/elicitation.rs:54`).
- Rule (normative RFD): `docs/rfds/v2/enum-variant-extension.mdx:23-31`. `_` prefix means extension-owned. Non-underscore
  means ACP-owned. "Extensions MUST NOT define custom non-underscore values." "Implementations MUST NOT treat an unknown
  non-underscore value as a custom extension." "SHOULD preserve unknown values when storing, replaying, proxying, or
  forwarding."
- Opening is applied "when the receiving side has an acceptable fallback"; closed enums remain where understanding is
  required for correctness (RFD lines 29, 55). The FAQ states that capabilities can still gate behavior when behavior
  matters. Tagged-union fallback in v2: `SessionUpdate::Other(OtherSessionUpdate)` preserves discriminator and payload
  (`S/v2/client.rs:196-208`), and the RFD says "Raw fallbacks are appropriate for notification-style and display-data
  payloads".
- v1 `SessionUpdate` is **closed** (no `Other`), so a new v1 variant needs a capability or old clients fail to parse
  (`docs/rfds/session-notices.mdx`, "Why is the capability required in v1?").

**Timestamps**:
- v1 uses the phrase "ISO 8601 timestamp" for `SessionInfo.updatedAt` and `SessionInfoUpdate.updatedAt`
  (`S/v1/agent.rs:1779,1824`; `S/v1/client.rs:1779,1810`), typed `String`/`MaybeUndefined<String>`.
- v2 uses "RFC 3339 timestamp" with the schemars extension `"format" = "date-time"` (`S/v2/client.rs:1349,1383`;
  `S/v2/agent.rs:1945,1994`). v2 `Annotations.lastModified` is "Must be an RFC 3339 formatted string (e.g.,
  "2025-01-12T15:00:58Z")" (`S/v2/content.rs:811`).
- v2 elicitation: "Date-time format (RFC 3339)" (`S/v2/elicitation.rs:47`); v1 says "Date-time format (ISO 8601)"
  (`S/v1/elicitation.rs:48`).
- Docs examples: `"updatedAt": "2025-10-29T14:22:15Z"` (`docs/protocol/v2/draft/session-list.mdx:85`).
- Clear semantics for patch fields: `MaybeUndefined<T>` (omitted = unchanged, `null` = clear).
  `SessionInfoUpdate` is documented as "Omitted fields leave the existing session info unchanged. `null` clears the
  corresponding value." (`S/v1/agent.rs`/`client.rs` `SessionInfoUpdate` doc).
- Epoch milliseconds exist only in NES: `timestamp_ms: u64` ("Timestamp in milliseconds since epoch",
  `S/v1/nes.rs:2128`; `S/v2/nes.rs:2155`) and `last_focused_ms: Option<u64>` (`S/v1/nes.rs:2193`).
  Wire names are `timestampMs` / `lastFocusedMs`.

**Durations**:
- None. A full-text search of `S/v1` and `S/v2` plus docs for duration, TTL, `retry_after`, `expires`, `seconds`,
  `timeout` found no protocol duration field. Hits were only the NES `*_ms` epoch timestamps, a terminal-timeout
  client-side recipe (`docs/protocol/v1/draft/terminals.mdx:256`), and a terminal test using `reason: timeout` in
  `_meta`.
- Unit-in-name convention is established by NES: `_ms` suffix means milliseconds, camelCase on the wire.

**Other numeric conventions** (relevant to usage and quota):
- `UsageUpdate.used`/`size`: `u64` tokens (`S/v1/client.rs:1836-1840`). `Cost.amount`: `f64`, `Cost.currency`: ISO 4217
  string (`S/v1/client.rs:1896-1899`).

**Optional/nullable field idiom** (CLAUDE.md "Schema rules" requires it be explicit):
- `#[serde_as(deserialize_as = "DefaultOnError")]`, `x-deserialize-default-on-error`, `Option<T>` plus
  `#[serde(default)]`, and a doc line like "Omitted and `null` are equivalent" (see `Notice.description`, `Notice._meta`
  at `S/v1/client.rs:696-709`).
- Lists use `DefaultOnError<VecSkipError<_>>` so malformed entries are dropped (e.g. `ListSessionsResponse.sessions`,
  `S/v1/agent.rs:1612-1616`).
- Cursor pagination idiom: `cursor`/`nextCursor` (`S/v1/agent.rs:1555-1562`, `1617-1621`).

## Implications for the RFD

1. **Agent capability shape (a)**. Follow the presence-marker pattern, not a boolean. In v1 add
   `AgentCapabilities.quota: Option<QuotaCapabilities>` or `SessionCapabilities.quota` (see decision below). `{}`
   means supported, omitted/`null` unsupported. The marker struct holds `_meta` only (like `SessionListCapabilities`).
   Booleans are legacy (`loadSession`) and should not be used. Required doc wording:
   - "Optional. Omitted or `null` both mean the agent does not advertise support. Supplying `{}` means the agent
     supports ..."
   - "If `quota` is omitted or `null`, Clients **MUST NOT** call `quota/read`" (mirrors session-list.mdx:54).
   - Unstable gating via a new Cargo feature (`unstable_quota`, add to the `unstable` umbrella).
2. **Placement decision (not made here)**. `quota/read` is global, not session-scoped (like `session/list`,
   `providers/*`). There are two precedents:
   - top-level agent capability (as `providers`, `nes`): `agentCapabilities.quota` in v1 and `capabilities.quota` in
     v2;
   - under `sessionCapabilities` (as `list`, `delete`): this would make quota unavailable to agents without
     `session` in v2, because v2 treats `capabilities.session` as the "session surface" marker
     (`S/v2/agent.rs:4125-4133`).
   Top-level matches `providers`/`nes` better for session-independent state. A `quota_update` session update would be
   session-scoped, but quota is usually account-wide. The RFD should say explicitly that `quota_update` is delivered
   on a session and may repeat account-wide data.
3. **One capability can gate both request and notification**. There is a precedent: NES (one `nes` capability gates
   `nes/*` requests and `document/*` notifications, `S/v1/nes.rs`), `plan` (two updates), `compaction` (two updates).
   There is **no** precedent for an agent capability that gates an agent-to-client `session/update` variant. Today an
   agent-to-client update is gated only by a *client* capability. For `quota_update` the options are:
   - ungated, like `usage_update` and `session_info_update`. In v1 an old client that cannot parse an unknown
     `sessionUpdate` would fail, since v1 `SessionUpdate` is closed.
   - gated by a client capability (e.g. `clientCapabilities.session.quota: {}`), as `notices` and `compaction` are in v1.
   - the agent capability declares the request; `quota_update` is gated by the client capability in v1 and ungated in v2
     via `SessionUpdate::Other`.
   The conservative reading of the notice/compaction RFDs is the third option. An agent-only capability would let an
   agent push `quota_update` at a v1 client that may not parse it.
4. **v1 vs v2 asymmetry is deliberate**. v1 requires a client capability to add a `SessionUpdate` variant because the
   union is closed. v2 does not, because of `Other`. The RFD must define both a v1 client capability and "v2: no client
   capability" if it adds a session update, mirroring `docs/rfds/session-notices.mdx` and
   `docs/rfds/session-compaction.mdx:161-183`.
5. **`quota/read` request design**. `ReadQuotaRequest { _meta }` plus an optional `sessionId`, or an optional
   `providerId`/scope filter, would follow `ListProvidersRequest`/`ListSessionsRequest`. It would have no required
   params. Include the `x-side = "agent"` and `x-method` tags and follow the CLAUDE.md steps (method-name constant, enum
   variants in `AgentRequest`/`AgentResponse`, SideDocs, `npm run generate`, `npm run check`, example agents/clients).
6. **Open enums for status or severity**. Use `snake_case` known values plus `Other(String)`, with the exact docstring
   from `NoticeSeverity`. Define any unit or window `kind` enums the same way. Those are display or informational,
   so open enums are safe (RFD `enum-variant-extension.mdx:29`). Do not open enums that gate request correctness.
7. **Timestamps**. Use RFC 3339 `date-time` strings for `resetsAt` (v2 wording, with `"format" = "date-time"`). Note
   that v1 text says "ISO 8601" for `updatedAt`. Pick one phrase per version, or "RFC 3339 (a profile of ISO 8601)"
   to cover both.
8. **Durations**. The protocol has no duration field and no convention beyond `_ms` in NES. If the RFD needs window
   length, prefer a typed enum or `windowSeconds`/`windowMs: u64`, and say so explicitly. Epoch-ms for `resetsAt` would be
   inconsistent with the session `updatedAt` convention. Choose RFC 3339 for absolute times.
9. **Numeric fields**. Token counts are `u64` (`UsageUpdate`). `Cost.amount` is `f64` with an ISO 4217 currency
   string. Percentages and fractions have no precedent. Specify range explicitly (e.g. `0.0..=1.0` or `0..=100`).
10. **Nullable semantics must be spelled out**. CLAUDE.md "Schema rules" requires a statement on every nullable field.
    Reuse "Omitted and `null` are equivalent and mean ..." (see `Notice`), or `MaybeUndefined` patch semantics when the
    update is a delta (see `SessionInfoUpdate`: omitted = unchanged, `null` = clear).
11. **Replay**. Notices are live-only and "SHOULD NOT be included in session replay" (`session-notices.mdx`). Quota is
    snapshot-like. The RFD should state whether `quota_update` is replayed by `session/load`/`session/resume`. Compaction
    RFD handles replay explicitly (`session-compaction.mdx:236-237`) and is the template.
12. **Relation to existing `usage_update`**. It already covers `used`/`size` (context occupancy) and cumulative `cost`,
    ungated. `quota_update` should be a separate update, not an overload, to keep context window and quota distinct.

## Gaps

- **UNVERIFIED**: `docs/rfds/v2/client-filesystem-terminal-capabilities.mdx` was not read. It presumably explains why v2
  `ClientCapabilities` dropped `fs`, `terminal`, `session`.
- **UNVERIFIED**: NES `NesSessionId` and the exact set of `nes/*` and `document/*` methods carrying a session id.
- **UNVERIFIED**: whether `PlanCapabilities` (`S/v1/plan.rs:411`) is `{}` or has fields. Only its doc comment on
  `ClientCapabilities.plan` was read. The `subagents` doc range cited above (`S/v1/client.rs:3514-3552`) and the
  compaction variant range (`S/v1/client.rs:157-175`) are approximate (+/- a few lines).
- No repository doc discusses an *agent* capability that gates an agent-to-client update. This must be argued in the RFD
  rather than cited.
- No convention was found for percent/fraction fields, a currency-less money amount, or duration units outside NES
  `*_ms`.
- The 7 "(unstable)" Cargo feature flags are listed in `agent-client-protocol-schema/Cargo.toml`. No `unstable_session_usage`
  feature exists, since `usage_update` is stable in v1. `unstable_end_turn_token_usage` exists for the end-of-turn token
  usage RFD. This RFD was not read.
- `docs/protocol/v1/draft/overview.mdx` and `extensibility.mdx` were only grepped, not read in full (extension rule
  at line 47 and custom-capabilities section at line 115).

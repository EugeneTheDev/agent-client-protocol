# ACP error codes and their relationship to capabilities

Scope: ACP schema (`agent-client-protocol-schema/src/{v1,v2}/error.rs`), docs (`docs/protocol/`), RFDs (`docs/rfds/`).
All `file:line` are relative to the repo root. v1 and v2 `error.rs` are byte-identical (verified with `diff`), so v1
line numbers apply to both.

## Summary

- ACP defines exactly two custom codes in its `ErrorCode` enum: `-32000` `AuthRequired` and `-32002` `ResourceNotFound`.
  It also adopts `-32800` `RequestCancelled` (LSP-derived, not JSON-RPC 2.0) and the five standard JSON-RPC codes.
  Everything else decodes as `Other(i32)`.
- Neither custom code is capability-gated. `-32000` is gated by authentication state, not by a capability; `-32002` has
  no capability or doc association at all (only the Rust helper and enum doc).
- The only explicit error-to-capability rule in stable-track docs is for elicitation: a request in a mode the Client did
  not advertise yields `-32602` (Invalid params), and the Agent MUST NOT make such a request in the first place.
- The repo's dominant rule is "Agent MUST NOT send feature X unless the peer advertised capability Y", applied to
  notifications, update variants, and payload shapes. It is never applied to error responses. No doc says "an error code
  MUST NOT be returned unless a capability was negotiated". No doc says the opposite explicitly either, except the
  current quota RFD draft (`docs/rfds/rate-limits-and-quotas.mdx:428`), which is the first.
- Precedent for structured `error.data`: `-32002` carries `{"uri": ...}` (Rust helper only), NES `-32000` example carries
  `{"reason": "auth_required"}`, and a since-removed `-32042` carried `{"elicitations": [...]}`. None is capability-gated.
  `error.data` is an untyped `Option<serde_json::Value>`.
- Precedent for a feature-specific error that was added and then removed before stabilization: `UrlElicitationRequired`
  `-32042` (commit `d39c879a`).

## Fields

### (1) Every ACP-defined error code

| Code | Rust variant | Meaning | Capability-gated? | Source |
| --- | --- | --- | --- | --- |
| `-32700` | `ParseError` | Invalid JSON received | No (standard JSON-RPC) | `agent-client-protocol-schema/src/v1/error.rs:155`, `:197` |
| `-32600` | `InvalidRequest` | Not a valid Request object | No (standard) | `error.rs:159`, `:198` |
| `-32601` | `MethodNotFound` | Method does not exist or is not available | No (standard); see "Method not found" below | `error.rs:163`, `:199` |
| `-32602` | `InvalidParams` | Invalid method parameter(s) | No (standard); used for un-advertised elicitation mode | `error.rs:167`, `:200` |
| `-32603` | `InternalError` | Internal JSON-RPC error, "reserved for implementation-defined server errors" | No (standard) | `error.rs:172`, `:201` |
| `-32800` | `RequestCancelled` | Method aborted by caller cancellation or "because of resource constraints or shutdown" | No; tied to the `$/cancel_request` feature, but the Agent may emit it for internal cancellation without a capability | `error.rs:173-177`, `:202`; `docs/rfds/request-cancellation.mdx:96-100`, `:80-91` |
| `-32000` | `AuthRequired` | "Authentication is required before this operation can be performed" | No capability. Gated by auth state. See below | `error.rs:183`, `:203` |
| `-32002` | `ResourceNotFound` | "A given resource, such as a file, was not found" | No capability | `error.rs:187`, `:204` |
| any other i32 | `Other(i32)` | "Other undefined error code" | n/a | `error.rs:188-191`, `:205` |

Enum doc: "These codes follow the JSON-RPC 2.0 specification for standard errors and use the reserved range (-32000 to
-32099) for protocol-specific errors" (`error.rs:139-142`). The module header says "additional protocol-specific error
codes for authentication and other ACP-specific scenarios" (`error.rs:4`). The generated docs list the same set
(`docs/protocol/v1/schema.mdx:3471-3516`; v1 draft `docs/protocol/v1/draft/schema.mdx:4553-4598`; v2
`docs/protocol/v2/schema.mdx:3425-3470`).

Codes proposed or used by RFDs but not in the `ErrorCode` enum (they decode as `Other`):

| Code | Where | Meaning | Source |
| --- | --- | --- | --- |
| `-33000` | MCP-over-ACP RFD | Binding resource limit exceeded | `docs/rfds/mcp-over-acp.mdx:180-186` |
| `-33001` | MCP-over-ACP RFD | Server registration unavailable | same |
| `-33002` | MCP-over-ACP RFD | Backend or transport failed without a valid MCP outcome | same |
| `-32003` | Quota RFD draft | Placeholder for `quotaExceeded`, "exact number to be assigned" | `docs/rfds/rate-limits-and-quotas.mdx:399`, `:414`, `:425` |
| `-32042` | Elicitation RFD (removed) | `UrlElicitationRequired` | removed in `d39c879a` ("fix(unstable): remove URL elicitation error (#1574)") |

Other uses of `-32000` outside the enum: MCP inner errors, which are explicitly not ACP codes
(`agent-client-protocol-schema/src/v1/mcp.rs:24` "Inner MCP error code; never an ACP error code."; tests at
`mcp.rs:342-347`; RFD `docs/rfds/mcp-over-acp.mdx:172` "an inner `-32000` must not trigger ACP authentication").

### Rust helper constructors (`Error`)

| Helper | Behaviour | Source |
| --- | --- | --- |
| `Error::auth_required()` | code `-32000`, no data | `error.rs:114-117` |
| `Error::resource_not_found(uri: Option<String>)` | code `-32002`; if `uri` is Some, `data = {"uri": uri}` | `error.rs:120-128` |
| `Error::into_internal_error(err)` | `-32603`, `data = err.to_string()` (a bare string) | `error.rs:130-136` |
| `Error::data(impl IntoOption<Value>)` | attach arbitrary `data` | `error.rs:59-63` |

### `Error` object shape

| Field | Type | Optional? | Meaning | Source |
| --- | --- | --- | --- | --- |
| `code` | `ErrorCode` (i32 on the wire; `serde(from = "i32", into = "i32")`) | required | error type | `error.rs:39`, `:146-148` |
| `message` | string | required | "concise single sentence" | `error.rs:41-43` |
| `data` | `Option<serde_json::Value>` (any JSON value) | optional; `DefaultOnError` so an invalid value deserializes to `None`; `skip_serializing_none` | "primitive or structured value ... debugging information or context-specific details" | `error.rs:44-50` |

`ErrorCode` is `#[non_exhaustive]` and unknown ints round-trip via `Other` (`error.rs:148`, tests `error.rs:322-327`).
`docs/rfds/v2/enum-variant-extension.mdx:41` says numeric code spaces such as JSON-RPC error codes are outside its
open-enum policy.

## (2) Capability relationship per code

- `-32000` AuthRequired. Not tied to a capability. It is returned when the Agent requires authentication that has not
  completed. Documented as possible on `session/new` ("May return an `auth_required` error if the agent requires
  authentication", `agent-client-protocol-schema/src/v1/agent.rs:4858`; `docs/protocol/v1/draft/session-setup.mdx:10`),
  on `authenticate` flows (`agent.rs:4819`; `docs/protocol/v1/authentication.mdx:167`), after `logout`
  (`docs/protocol/v1/authentication.mdx:215-225`, `docs/rfds/logout-method.mdx:185-194`), and on `nes/start`
  (`docs/rfds/next-edit-suggestions.mdx:561-579`: "Clients **must** be prepared to handle `auth_required` errors on
  `nes/start`"). It can therefore appear on a capability-gated method, but the capability is what makes the method
  callable; the error is not conditional on it. The v2 prompt RFD and the draft subagents RFD reuse it inside `error`
  stop reasons (`docs/rfds/v2/prompt.mdx:272`, `:288`; `docs/rfds/subagents.mdx:555`;
  `docs/protocol/v2/prompt-lifecycle.mdx:536-550`).
- `-32002` ResourceNotFound. No capability tie. No RFD or protocol doc mentions it (grep over `docs/` found only the
  generated schema pages). The only usage is the enum plus `Error::resource_not_found` helper (`error.rs:120-128`).
- `-32602` for un-advertised elicitation mode. This is the one error explicitly defined in terms of a capability: "Requests
  using a mode the Client has not advertised produce JSON-RPC `-32602` (Invalid params)"
  (`docs/protocol/v1/elicitation.mdx:166-167`; RFD `docs/rfds/elicitation.mdx:679-684`). It is a Client-returned error for an
  Agent protocol violation, and the preceding normative sentence forbids the violation: "Agents **MUST NOT** request a mode
  the Client has not advertised" (`docs/protocol/v1/elicitation.mdx:54-55`).
- `-32601` MethodNotFound. The extensibility docs use it for unknown custom (underscore) methods and then recommend
  advertising custom capabilities "so that callers can check their availability first" (`docs/protocol/v1/extensibility.mdx:84-97`).
  So Method-not-found is the fallback when a capability was not checked. For stable methods, the rule is the inverse:
  callers must check the capability (e.g. `docs/protocol/v1/authentication.mdx:191-194` "Clients should only call
  [logout] after verifying the Agent advertised `agentCapabilities.auth.logout`"; `docs/protocol/v2/authentication.mdx:49-55`
  "Clients **MUST NOT** call either method" when `authMethods` is omitted). No doc specifies which error code an Agent
  returns when a Client calls an un-advertised method anyway. UNVERIFIED whether any doc says `-32601` for that case
  (only the custom-method case is documented).
- `-32800` RequestCancelled. Gated by the cancellation feature in the sense that it answers `$/cancel_request`, but
  internal cancellation also uses it (`docs/rfds/request-cancellation.mdx:86-92`). v2 narrows when it may be used on
  `session/prompt` (only before the user message is inserted: `docs/protocol/v2/cancellation.mdx:40`,
  `docs/rfds/v2/prompt.mdx:286`).
- `-33000..-33002` (MCP-over-ACP draft). Outer ACP errors "reserved for failures to admit, route, or execute the binding
  itself" (`docs/rfds/mcp-over-acp.mdx:178`). The RFD states the capability is a separate optional object, `{}` = supported
  (`:95`), but does not say these codes are only returned when it is advertised. They are inherently scoped to requests
  that use the binding, which in practice requires the capability. UNVERIFIED that any sentence ties them explicitly.

## (3) Rules: MUST NOT without capability vs. errors without capability

General capability rules (apply to all features, not to errors):

- "All capabilities included in the `initialize` request are **OPTIONAL**... Clients and Agents **MUST** treat all
  capabilities omitted in the `initialize` request as **UNSUPPORTED**" (`docs/protocol/v1/initialization.mdx:104-106`).
  New capabilities are non-breaking by this rule.
- Omitted and `null` are equivalent; `{}` advertises (`docs/protocol/v1/initialization.mdx:176`, `:251`, `:260`;
  `docs/rfds/session-notices.mdx:124`; `docs/rfds/boolean-config-option.mdx:55`).

"Agent MUST NOT send X unless Client advertised capability" (all about success-path data, never errors):

| Rule | Source |
| --- | --- |
| `notice` updates | `docs/protocol/v1/draft/prompt-turn.mdx:272-275`; `docs/rfds/session-notices.mdx:127`. Fallback allowed: ordinary agent message (`docs/rfds/session-notices.mdx:130`) |
| `compaction_update`, `compaction_summary_chunk` (live and replay) | `docs/protocol/v1/draft/prompt-turn.mdx:341-346` |
| `plan_update`, `plan_removed` | `docs/protocol/v1/draft/agent-plan.mdx:53` ("If the Client omits `plan`, the Agent MUST fall back to the existing `plan` update") |
| `type: "boolean"` config options in `session/new`, `session/load`, `session/update`, `session/set_config_option` responses | `docs/rfds/boolean-config-option.mdx:57` |
| Subagent updates unless Client advertised `subagents` | `docs/rfds/subagents.mdx:72-75` |
| Elicitation modes not advertised | `docs/protocol/v1/elicitation.mdx:54-55` |
| NES events only if key present in Agent's `events` capability (Client-side rule) | `docs/rfds/next-edit-suggestions.mdx:140` |

Errors-related statements:

- No statement found that forbids an Agent from returning a particular ACP error code absent a capability.
- The closest precedent was in the removed URL-elicitation text: "Agents MUST NOT return the `URLElicitationRequiredError`
  (code `-32042`) except when URL mode elicitation is required" (removed in `d39c879a`; see `git show d39c879a --
  docs/rfds/elicitation.mdx`). That gated the error on a situation, not on a capability. The removal rationale: "it isn't
  well speced out in the schema anyway, and MCP has moved to a different pattern" (commit message). The current text is
  "ACP has no separate URL-required error. Agents request URL mode directly through `elicitation/create`"
  (`docs/rfds/elicitation.mdx:685-686`).
- Explicitly "errors may appear regardless of capability": only the quota RFD draft says it ("An Agent MAY return
  `quotaExceeded` even without the quota capability", `docs/rfds/rate-limits-and-quotas.mdx:428-430`). The v2 prompt RFD
  is consistent in spirit: the `error` stop reason is present in every v2 agent, and more specific categories "such as rate
  limits, an exhausted context window, or an overloaded provider, can be added later as codes or as structured `data`
  without changing this shape" (`docs/rfds/v2/prompt.mdx:288`). That is the only existing doc that names rate limits as a
  future error category.
- Failure-reporting taxonomy (v2): rejected before insertion is a JSON-RPC error on `session/prompt`; failure after
  insertion is an idle `state_update` with `stopReason: "error"` plus a JSON-RPC error object; recoverable failures are
  `notice` or a failed tool call (`docs/rfds/v2/prompt.mdx:292-302`; `docs/protocol/v2/prompt-lifecycle.mdx:552+`). A quota
  rejection could occur in either of the first two surfaces, so error `data` on both paths matters. A bridge maps v2
  `error` absence to `-32603` (`docs/rfds/v2/prompt.mdx:306`).

## (4) Precedents for structured `error.data`

| Error | `data` shape | Capability-gated? | Source |
| --- | --- | --- | --- |
| `-32002` via `Error::resource_not_found(Some(uri))` | `{"uri": "<string>"}` | No | `error.rs:120-128` |
| `-32603` via `Error::into_internal_error` | bare string (`err.to_string()`) | No | `error.rs:130-136` |
| NES `-32000` example | `{"reason": "auth_required"}` (a redundant string discriminator inside data) | No | `docs/rfds/next-edit-suggestions.mdx:566-576` |
| `-32042` (removed) | `{"elicitations": [{"mode":"url","elicitationId","url","message"}]}` | Conceptually (URL mode), error removed | `git show d39c879a -- docs/rfds/elicitation.mdx` |
| MCP-over-ACP inner error carrier | `{"code","message","data"}` inside `result.error`, `data` opaque and preserved, distinguishes omitted from explicit `null` | n/a (different error domain) | `docs/rfds/mcp-over-acp.mdx:154-172`, `:174-176` |
| v2 `error` stop reason | full JSON-RPC error object, optional and nullable; `data` optional | v2 core, not gated | `docs/rfds/v2/prompt.mdx:280-284`; `docs/protocol/v2/prompt-lifecycle.mdx:545-550` |

Observations:

- No ACP error has a schema-typed `data`. `Error.data` is untyped JSON (`error.rs:44-50`), and `DefaultOnError` means a
  malformed `data` degrades to `None` instead of failing the whole error (`error.rs:46-48`). So extra fields in `data` are
  always safe for old Clients to receive; no capability is required to carry them.
- No precedent exists for `data` content that differs by negotiated capability. UNVERIFIED claim would be that none
  exists; a grep of `docs/` and `src/` for `"data"` near `error` found only the entries above.
- `_meta` is available on most typed objects for extensions (`docs/protocol/v1/extensibility.mdx:8-13`) but `Error` itself
  does not define `_meta` (`error.rs:34-51`), so extension data in an error goes in `data`.
- Error `message` is meant to be one concise sentence (`error.rs:41-43`), so human-readable detail belongs there and
  machine-readable detail in `data`.

## (5) How RFDs that add or define error codes phrase the capability relationship

- Request cancellation (`docs/rfds/request-cancellation.mdx:96-100`): "Add standard JSON-RPC error code `-32800` for
  cancelled requests" with Code, Message, and Meaning. No capability phrasing; the code is unconditional and is also used
  for internal cancellation (`:86-92`).
- Elicitation (`docs/rfds/elicitation.mdx:679-686`): lists error codes under "Error Handling" and ties the error to the
  capability as a violation consequence ("a mode not declared in client capabilities: `-32602`"). The earlier version
  (removed) phrased a gate on situation: "MUST NOT return ... except when URL mode elicitation is required".
- MCP-over-ACP (`docs/rfds/mcp-over-acp.mdx:178-188`): a table of codes, "reserved for failures to admit, route, or execute
  the binding itself", and a namespace rule "These binding-specific codes do not allocate new meanings in MCP's reserved
  `-32000` through `-32019` range. No implementation may report overload as ACP's authentication-required error." It
  chooses `-33xxx` to avoid colliding with another spec's reserved range.
- Logout (`docs/rfds/logout-method.mdx:185-194`): `auth_required` after logout, phrased with "should", and the method itself
  gated by `agentCapabilities.auth.logout`.
- NES (`docs/rfds/next-edit-suggestions.mdx:561-579`): "the agent may reject `nes/start` with an error", the error is
  `auth_required` with a `reason` discriminator, "Clients **must** be prepared to handle" it.
- v2 prompt (`docs/rfds/v2/prompt.mdx:288`): the code space is open: "More specific categories ... can be added later as
  codes or as structured `data`".
- Quota RFD draft (this branch): `quotaExceeded` code placeholder `-32003`; `data` all-optional; "An Agent MAY return
  `quotaExceeded` even without the quota capability. In that case `poolId` and `meterId` have no snapshot to refer to and
  SHOULD be omitted" (`docs/rfds/rate-limits-and-quotas.mdx:428-430`).

## Delivery mechanism

Errors travel as the JSON-RPC `error` member of a response (`docs/protocol/v1/overview.mdx:217-224`: "Errors include an
`error` object with `code` and `message`; Notifications never receive responses (success or error)"). Under v2 they may
also be embedded as an `error` object on the idle `state_update` stop reason, and in v1 draft on `subagent_update.state`.

## Windows and semantics

Not applicable to this task.

## Implications for the RFD

1. Allocation: `-32000` and `-32002` are the only ACP-owned codes in `-32000..-32099`. `-32001` is unallocated in ACP
   (JSON-RPC server-error range; MCP reserves `-32000..-32019`, see `docs/rfds/mcp-over-acp.mdx:188` and
   `agent-client-protocol-schema/src/v1/mcp.rs:24`). The RFD's `-32003` placeholder sits in that MCP-reserved band and so
   risks collision with MCP codes, which the MCP-over-ACP RFD explicitly avoided by using `-33000..-33002`. The
   RFD author should decide the allocation range deliberately (design question; not decided here).
2. "Error may appear without capability" is consistent with how `-32000`, `-32002`, `-32800`, and the v2 `error` stop reason
   work, and with the generic rule that Clients must treat unknown codes as `Other(i32)` (`error.rs:205`). It is the
   inverse of every `MUST NOT send` rule for updates, so the RFD should state it explicitly, as the draft does.
3. Because `Error.data` is untyped and `DefaultOnError`, the typed `data` shape for `quotaExceeded` is a documentation
   contract only; the generated schema will not type it unless a new struct is added and referenced from prose.
   `Error::resource_not_found` is the existing precedent for a convenience constructor that fixes a `data` shape.
4. If the RFD wants data fields gated by capability (e.g. `poolId`/`meterId` only when quota is advertised), no precedent
   exists; the draft's SHOULD-omit rule is the first of its kind. A simpler precedent-aligned alternative is "fields are
   always optional; Clients ignore unknown or dangling IDs", like elicitation's "Clients **MUST** ignore unknown or
   already-completed IDs" (`docs/protocol/v1/elicitation.mdx:163-165` region).
5. For v2, the `error` stop reason (`docs/rfds/v2/prompt.mdx:280-288`) already names rate limits as a future code or `data`
   category, so the `quotaExceeded` code and `data` shape should work identically on a `session/prompt` error response
   and on the stop-reason `error` object.

## Gaps and unverified items

- Which code an Agent should return when a Client calls a method the Agent did not advertise is not specified in any doc
  read (only custom methods => `-32601`, `docs/protocol/v1/extensibility.mdx:84-95`). UNVERIFIED for stable methods.
- `-32002` has no protocol-level documentation beyond the enum and helper. No example of `data.uri` exists in `docs/`.
- Whether MCP-over-ACP `-33xxx` codes are explicitly tied to its capability is not stated (inferred only).
- I did not search the `schema/` JSON files or external SDKs (e.g. TypeScript SDK) for additional error code definitions
  or for helper constructors with other `data` shapes.
- Line numbers verified against `error.rs` (RequestCancelled variant `:177`, AuthRequired `:183`, ResourceNotFound `:187`).

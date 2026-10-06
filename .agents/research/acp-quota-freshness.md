# ACP convention: deciding which quota snapshot is newer (push vs poll)

Scope: an ACP protocol question, not a single agent. The draft (`docs/rfds/rate-limits-and-quotas.mdx`) delivers a
`QuotaSnapshot { pools, updatedAt?, _meta? }` both as a pushed `quota_update` and as a polled `quota/read` result. Today
the draft says message order decides freshness and `updatedAt` is display only (`rate-limits-and-quotas.mdx:251-252`,
`:283-284`, `:300`, `:509-512`). This note checks whether message order can do that job, lists precedents, records what
timestamps each agent actually has, and sketches design options.

Paths: ACP repo paths are relative to the repo root. Schema paths are under `agent-client-protocol-schema/src/`.
`RUST-SDK` = local checkout of the ACP Rust SDK at `b55cdc7` (2026-09-28), paths under `src/agent-client-protocol/src/`.
`TS-SDK` = local checkout of the ACP TypeScript SDK at `c77d9bd` (2026-09-28), paths under `src/`.

## Summary

- **Message order does not tell the client which data is newer.**
  - stdio is one FIFO byte stream, and both SDKs serialize outgoing writes. The client therefore sees the order in
    which the Agent *sent* messages.
  - Send order is not the order in which the data was *observed*. A `quota/read` handler that awaits a cache or a
    provider call runs outside the dispatch loop, so a push from a concurrent turn can be written between the moment
    the read's data was taken and the moment its response is written.
  - The proposed Streamable HTTP transport sends session updates and connection-level responses on **different SSE
    streams**. There is no order between them at all. ACP v2 docs already tell clients not to infer anything from the
    arrival order of a response relative to updates.
- **ACP has no precedent for a staleness token.**
  - The only timestamps are display-only "last activity" strings: `SessionInfoUpdate.updatedAt` and
    `SessionInfo.updatedAt`, plus `NesUserAction.timestampMs` (epoch ms).
  - There are no `version`, `revision`, or ETag fields apart from `protocolVersion`.
- **Other protocols:**
  - HTTP picks "the most recent" stored response by `Date` (1 s resolution). ETags support equality only.
  - Kubernetes `resourceVersion` was opaque but is now orderable as a decimal integer (conformance requirement since
    1.35).
  - LSP resolves exactly this push/pull conflict for diagnostics: the higher document version wins, and on a tie the
    pull wins.
  - MCP pushes only an invalidation (`uri`), so there is nothing to order.
- **Agents:**
  - All six agents can stamp a *local wall-clock* observation time. Claude Code already orders its own quota state
    by observation time, using a "newer-or-equal wins" rule (`isStaleObservation`).
  - Only Copilot's raw payload has a server timestamp (`timestamp_utc`), and only on a passthrough type.
  - None has a monotonic revision.
- **Options:**
  - (A) required `updatedAt` as an Agent-monotonic observation time, with "newer-or-equal wins";
  - (B) an orderable per-connection `revision`, with `updatedAt` kept for display;
  - (C) both.
  - Recommendation (not a decision): A, with an "equal means identical" rule and clamping against clock steps.

## Findings

### 1. Message ordering in ACP and JSON-RPC

**What ACP specifies**

- stdio framing: "Messages are delimited by newlines" (`docs/protocol/v2/transports.mdx:25`;
  `docs/protocol/v1/transports.mdx:24`). One stream means FIFO delivery in send order. Nothing in the docs states an
  ordering *guarantee* between a notification and a response beyond that.
- Batches: "The receiver MAY process batch entries as concurrent tasks, in any order" and "Response objects MAY appear
  in any order" (`docs/protocol/v2/transports.mdx:60-69`).
- The only ordering rule that ties a response to notifications is for cancellation. The Agent must send its trailing
  updates before responding to `session/prompt` (`docs/protocol/v1/prompt-turn.mdx:365`). In v2 the rule is "before
  sending the idle `state_update`" (`docs/protocol/v2/prompt-lifecycle.mdx:610`). It is a per-method rule, not a
  general one.

**ACP v2 explicitly rejects inferring from arrival order**

- "If the message update arrived first, the Client reconciles with that existing message ... Clients must not infer
  the association from content or arrival order" (`docs/protocol/v2/prompt-lifecycle.mdx:185`).
- "clients MUST tolerate those updates arriving before or after the response" (`docs/rfds/v2/prompt.mdx:74`). The same
  idea appears in `docs/rfds/v2/message-updates.mdx:63`.
- "The terminal reference and terminal session updates may arrive in either order"
  (`docs/rfds/v2/terminal-output.mdx:48`).

**Remote transport: no order between session updates and sessionless responses**

- The Streamable HTTP RFD defines a connection-scoped SSE stream that carries "responses to `session/new` ... and any
  server-initiated messages not tied to a specific session". A separate session-scoped stream carries "session update
  notifications ... and responses to session-scoped POSTs" (`docs/rfds/streamable-http-websocket-transport.mdx:35`).
- A `quota/read` without `sessionId` would therefore be answered on the connection stream, while `quota_update`
  arrives on each session's stream. Two independent HTTP/2 streams have no relative order.
- Pushes on two different sessions (the same account data on session A and session B) are not ordered relative to
  each other either.
- The RFD also says "There is no message sequencing or stream resumption" (`:78`) and lists "lost or out-of-order
  updates" as something v2 should reduce (`:85`).

**Agent side (Rust SDK): wire order is send order, not data order**

- All outgoing messages go through one unbounded mpsc channel consumed by a single actor (RUST-SDK
  `jsonrpc/outgoing_actor.rs:9-18`, `:27-35`). The wire order is therefore the order of the `send` calls.
- Incoming messages are dispatched one at a time: "the dispatch loop waits for each handler to complete before
  processing the next message" (RUST-SDK `concepts/ordering.rs:13-15`).
- Long work must leave the loop. A handler that awaits another response blocks the loop and deadlocks
  (`concepts/ordering.rs:41-58`), and the guidance is to `spawn` it (`:127-143`).
- A `quota/read` that refreshes from a provider or awaits shared state therefore runs concurrently with the prompt
  turn's pushes. In that case its data can be older than a push that is written before its response.
  - This is my inference from the dispatch model. I did not observe it in an agent.

**Client side**

- RUST-SDK: notification callbacks run in order inside the dispatch loop. A response delivered to `block_task()` is
  handed off and "the dispatch loop continues immediately" (`concepts/ordering.rs:60-92`). Ordered response handling
  needs `on_receiving_result` registered before the response is routed (`:94-121`).
- The SDK's cookbook says application code must put updates and responses on the same FIFO to keep wire order
  (`agent-client-protocol-cookbook/src/ordered_application_dispatch.rs:1-14`).
- TS-SDK: the receive loop does not await message processing (`jsonrpc.ts:1221-1230`). Notifications go to async
  handlers, and responses resolve a pending promise directly (`jsonrpc.ts:1363-1371`, `:1489-1503`).
  - At the application level, wire order holds only if the handler applies state before its first `await`. This is
    my inference from the code.
  - Outgoing writes are serialized through `writeQueue` (`jsonrpc.ts:1530-1554`).

**Answer to Q1:**

- On stdio, a client can reliably tell which message was *sent* later, if its SDK and application preserve order.
- It cannot tell which *data* is newer. On the HTTP transport it cannot even tell send order across the session
  stream and the connection stream.
- The current rule in the draft ("ordered by message order", `rate-limits-and-quotas.mdx:251-252`) is therefore
  insufficient for the push-vs-read race.

### 2. ACP precedents for timestamps and versions

| Field | Location | Format | Semantics |
| --- | --- | --- | --- |
| `SessionInfoUpdate.updatedAt` (v2) | `v2/client.rs:1349-1353` | string, "RFC 3339 timestamp of last activity. Set to null to clear." | Display or "last activity". No ordering rule. |
| `SessionInfoUpdate.updatedAt` (v1) | `v1/client.rs:1779-1783` | string, "ISO 8601" | Same |
| `SessionInfo.updatedAt` (`session/list`) | `v1/agent.rs:1779-1783` | optional string, ISO 8601 | Sorting and display of sessions (`docs/rfds/session-list.mdx:42`, `:137`) |
| RFD guidance | `docs/rfds/session-info-update.mdx:322-326` | — | "Clients should process updates in order". The Agent "typically" sets `updatedAt` on any activity. Message order is the ordering rule here too. |
| `NesUserAction.timestampMs` | `v1/nes.rs:2128-2129`; `v2/nes.rs:2155-2156` | `u64`, "milliseconds since epoch" | Client-to-Agent event time. Shows ACP already accepts a numeric epoch-ms field. |
| `protocolVersion` | `version.rs:4-50` | `u16` | Protocol negotiation, not data versioning |

- I found no `revision`, `etag`, `sequence`, or data-`version` field in the v1 or v2 schema. A grep over
  `agent-client-protocol-schema/src` returns only test `_meta` samples.
- Ecosystem (non-normative): a vendor-namespaced failure-record extension used by two ACP adapters carries an integer
  `revision` per record id, incremented on each republish (`claude-code-acp-adapter.md:101`;
  `codex-acp-adapter.md:90`). It is a per-entity counter, not a cross-channel freshness token.
- RFC 3339 strings allow any offset and any fractional precision. Lexicographic comparison is only valid if the format
  is pinned (UTC `Z`, fixed fraction digits). ACP does not pin it today.

### 3. Precedents in other protocols

| Protocol | Mechanism | Ordering? | Notes and trade-offs | Source |
| --- | --- | --- | --- | --- |
| HTTP `Date` | Origination time of each message. Origin servers with a clock MUST send it on 2xx/3xx/4xx | yes, coarse | Caches choose "the most recent one (as determined by the Date header field)". 1 s resolution, so ties are common. Origin-server clock, so skew across servers is possible | RFC 9110 §6.6.1; RFC 9111 §4 ("When more than one suitable response is stored, a cache MUST use the most recent one") |
| HTTP `Last-Modified` | Server's belief of the last change time | yes, coarse | Implicitly a *weak* validator because of 1 s resolution ("did not change twice during the second"). MUST NOT be later than `Date`, and a future value is clamped to `Date`. This is a precedent for clamping | RFC 9110 §8.8.2.1, §8.8.2.2 |
| HTTP `ETag` | "opaque validator", strong or weak | **no**, equality only | Exists because "the one-second resolution of HTTP-date values is not sufficient". Tells "same or different", never "newer" | RFC 9110 §8.8.3, §8.8.3.2 |
| Kubernetes `resourceVersion` | Per-object or per-list version used for list-then-watch | historically opaque, now orderable | Watch resumes "from the last returned `resourceVersion`". Expired versions give `410 Gone`, and the client must re-list. Since 1.35, versions "must be orderable as a monotonically increasing integer", compared as arbitrary-size decimal strings. Extension servers may still be non-numeric, and then only equality is allowed. Lesson: pure opacity blocked legitimate "which is newer" checks, and the project later mandated orderability | https://kubernetes.io/docs/reference/using-api/api-concepts/ ("Efficient detection of changes", "Resource versions") |
| LSP document `version` | Integer per document that "will increase after each change ... doesn't need to be consecutive" | yes | Client-owned counter. Servers echo it on `publishDiagnostics.version` (optional, `versionSupport` capability) | LSP 3.17 `VersionedTextDocumentIdentifier`; `textDocument/publishDiagnostics` |
| LSP pull vs push diagnostics | Same data from two paths | yes | "diagnostics for a higher document version should win over those from a lower document version", and "diagnostics from a document pull should win over diagnostics from a workspace pull". This is a direct precedent: a version decides first, then a source-precedence tie-break. Pull also has an opaque `resultId` / `previousResultId` for "unchanged" | LSP 3.17 "Workspace Diagnostics"; `resultId` in `DocumentDiagnosticReport` |
| LSP message ordering | Responses "roughly the same order as the requests". Reordering is allowed "as long as this reordering doesn't affect the correctness" | n/a | JSON-RPC-based protocols do not promise cross-message order | LSP 3.17 "Request, Notification and Response Ordering" |
| MCP `notifications/resources/updated` | Carries only `uri`. The resource "may need to be read again" | n/a | Invalidate-then-read sidesteps the race: the push has no data to compare. Costs an extra round trip. `Annotations.lastModified` (ISO 8601) is display or priority metadata only. Progress uses "should increase every time" | MCP schema 2025-11-25 (`ResourceUpdatedNotification`, `Annotations.lastModified`, `ProgressNotification.progress`) |
| GraphQL subscriptions | Query results and subscription event streams are separate operations | no | The spec defines no version or ordering link between a query response and a subscription event. Apps add their own versioning | GraphQL spec §6 "Subscription" / "Event Streams" |
| Codex TUI (in-agent precedent) | Source precedence, not time | n/a | Only poll results (`AccountUsage`) update the status display. "Streamed updates still drive warnings/recovery above, but must not overwrite status data" | `codex-rs/tui/src/chatwidget/rate_limits.rs:374-389` @ `823ea830` |

**Trade-offs (synthesis)**

- **Wall-clock timestamp** (agent observation time):
  - Survives Agent restarts and client persistence.
  - Comparable across sessions, streams, and connections from the same host.
  - Doubles as the "as of" display value.
  - Risks: clock steps backwards (NTP), skew between remote instances behind a load balancer, coarse resolution
    causing ties, and agents stamping the *send* time instead of the observation time.
- **Monotonic revision counter**:
  - Exact. Ties are meaningful (same revision means same state). No clock involved.
  - Resets on restart, so it is only comparable within one Agent process or connection. It needs an epoch or instance
    id to survive restarts.
  - Needs a shared counter for multi-instance remote agents.
  - Not displayable.
  - If opaque, clients cannot order it (the Kubernetes experience). JSON numbers above 2^53 lose precision.
- **Both**: exact within a connection, coarse across connections. More rules for clients to implement.

### 4. Timestamps each agent actually has

| Agent | Server-provided timestamp | Local observation time available | Monotonic revision | Notes | Sources |
| --- | --- | --- | --- | --- | --- |
| Claude Code | None in the SDK types. `rate_limit_event` and `SDKRateLimitInfo` have no time field, and neither do the status-line or `get_usage` payloads (`resetsAt` is a reset time, not an observation time). Response `Date` headers exist on HTTP (RFC 9110) but are not used | **Yes, internally.** The CLI records a clock read when response headers arrive (`n9=he.now()` right after "query_response_headers_received", bin@189353159). It passes this as `observedAt` to the header parser (`$ke(...,n9,...)`, bin@189378382; `extractQuotaStatusFromHeaders(e,n,r=!1,s=Date.now(),...)`, bin@187253310). `isStaleObservation(e){if(e<this.lastAppliedObservationAtMs)return!0;...}` drops older observations and accepts equal ones (bin@187249842). Per-window `observedAtMs` uses `s>=w.observedAtMs` (same region). An `accountEpoch` drops observations from a previous account (`isFromPreviousAccount`, bin@187249491) | No. Wall-clock ms plus an account epoch | Not exposed to SDK consumers. An ACP adapter can only stamp the time it *receives* `rate_limit_event`, and that event fires only on change (deep-compare; `claude-code-native.md` delivery table). So adapter-side `updatedAt` means "last changed", not "last confirmed". `/usage` snapshots are cached, written at most once per 60 s with a max age of 1 h (`claude-code-native.md:178`) | CLI 2.1.289 binary, offsets above; `claude-code-native.md:54-70`, `:170-182`; `claude-code-active.md:40`, `:83` |
| Codex | None. `RateLimitSnapshot` has no time field (`protocol/src/protocol.rs:2350-2362`, per `codex-native.md:40-64`). The backend's `reset_after_seconds` is dropped | Yes, at receipt. The TUI stamps `captured_at = Local::now()` when a poll result arrives (`tui/src/chatwidget/rate_limits.rs:384`) and calls data stale after 15 min (`tui/src/status/rate_limits.rs:65-66`, doc at `:7`, `:106-107`, `:210-213`) | No | Pushes can **reuse the cached snapshot** when a response carried no new data (`codex-active.md:55`). Stamping at push time would claim false freshness. The ACP adapter caches pushes and exposes them only as `/status` text (`codex-acp-adapter.md:106`, `:161`) | `823ea830` sources fetched via GitHub API; notes cited |
| Gemini CLI | None in `BucketInfo` (`resetTime` only; `gemini-native.md:51-54`) | Yes. `lastQuotaFetchTime = Date.now()` is set **after** `retrieveUserQuota` resolves (`packages/core/src/config/config.ts:2325`; field at `:864`, reset at `:1856`). `refreshUserQuotaIfStale(staleMs=30_000)` compares it to `Date.now()` (`:2392-2397`). Credit balance comes with each `generateContent` response (`remainingCredits`) and has no own timestamp | No | Refreshes are fire-and-forget after every response, plus forced from `/stats`. I saw no in-flight dedupe, so the last to *complete* wins regardless of request start (inference from `config.ts:2313-2330`, `:2392-2400`). The ACP mode exposes no quota (`gemini-acp.md:124`) | `fb972b2f` sources |
| Copilot CLI | **Only on the raw passthrough.** `CopilotUserResponse.quota_snapshots[*].timestamp_utc`, "when snapshot was captured" (`rpc.d.ts:6008`, per `copilot-cli.md:60`). Whether the server or the CLI sets it is UNVERIFIED. The typed `account.getQuota` `AccountQuotaSnapshot` has no time field (`copilot-cli.md:36-46`) | Yes, at poll completion. Quota is pull-only (`copilot-cli.md:120-121`) | No | No quota over ACP (`copilot-cli.md:112-117`). A server timestamp has unknown resolution and comes from a different clock than the Agent's | `copilot-cli.md`, `copilot-cli-active.md:93-95` |
| Cursor | None found. The undocumented `usage-summary` has billing-cycle bounds only (`cursor.md:48-60`) | Only if an agent polled that endpoint, which the CLI does not (`cursor.md:7`, `:64-66`) | No | No quota surface in CLI or ACP | `cursor.md` |
| Kimi CLI | None among the parsed fields (`kimi-cli.md:20-31`) | Yes, at response time. `/usage` is a single on-demand GET with no cache (`kimi-cli.md:50`) | No | No quota over ACP (`kimi-cli.md:8`) | `kimi-cli.md` |

**Answer to Q4:**

- Every agent that has quota data can produce a local wall-clock "observed at". None can produce a server-side or
  monotonic one without new code.
- Wall clocks are not monotonic. An agent that wants a non-decreasing value must clamp it: `max(now, lastEmitted)`.
- Claude Code's internal rule ("drop if older, accept if equal") is the closest existing precedent for option A.

## Design options (Q5)

Common definitions for all options:

- **Observation time**: when the Agent obtained the data from its provider, that is, when response headers or a poll
  result were received. It is not when the ACP message was sent.
- **Scope**: a snapshot pushed on session S, or read with `sessionId: S`, belongs to scope S. A read without
  `sessionId` belongs to the sessionless (default account) scope. Clients compare snapshots only within one scope (see
  the NEEDS_DECISION item on scope).

### Option A: required `updatedAt` as an Agent-monotonic observation time, "newer-or-equal wins"

Normative sketch:

- `updatedAt` (string, RFC 3339, **required**) is the time at which the Agent's quota knowledge reflected in this
  snapshot was last updated from its provider.
  - Agents SHOULD use UTC (`Z`) with millisecond precision.
  - Agents MUST NOT set it to the send time when the data was not re-observed.
- Within one Agent process, `updatedAt` values MUST be non-decreasing across all snapshots the Agent sends, pushed or
  read. If the system clock moves backwards, the Agent MUST use the last value it emitted.
- Two snapshots for the same scope with equal `updatedAt` MUST have identical `pools`.
  - An Agent that changes its knowledge within the same millisecond MUST advance `updatedAt` by at least 1 ms.
- Clients MUST keep, per scope, the snapshot with the greatest `updatedAt`. A snapshot whose `updatedAt` is older
  than the kept one MUST be ignored.
  - If they are equal, either may be kept, because they are identical.
- Clients MUST NOT compare `updatedAt` with their own clock for ordering. They MAY use it for "as of" display.
- `quota/read` MAY return cached data. If the cached data is older than what the Agent already pushed, the Agent
  SHOULD return its latest snapshot instead. A client that receives an older read result keeps the pushed snapshot.

Edge cases:

- **Equal timestamps.** The rule makes equality harmless (same state). Without the "identical" rule, a tie needs a
  tie-break, and "later received wins" brings the ordering problem back.
- **Agent restart.** The wall clock continues, so a client's persisted snapshot from a previous process stays
  comparable. A backward clock step across a restart can make fresh data look older. Clients SHOULD treat a new
  connection as resetting the "kept" snapshot, or at least accept the first snapshot of a new connection.
- **Clock going backwards.** Handled on the Agent side by clamping, like the `Last-Modified` ≤ `Date` clamp in
  RFC 9110 §8.8.2.1. Without clamping, Claude Code-style logic would drop fresh data until the clock catches up
  (inference from `isStaleObservation`).
- **Cached data older than the last push.** The client ignores it. This is correct, but a read that is answered with
  stale cache becomes a no-op. Agents should prefer returning their newest state.
- **Change-only sources** (Claude SDK events, Gemini `quota-changed`). The adapter only learns about changes, so
  `updatedAt` advances on change, not on confirmation. "As of" display may then look older than reality. This is safe
  for ordering, imperfect for display.
- **Remote multi-instance agents.** Skew between instances can misorder snapshots by up to the skew. This is
  acceptable for quota, which moves slowly relative to clock skew, but the RFD should say so.
- **Removal or empty snapshots.** `pools: []` also carries an `updatedAt`, and it must be newer to replace data. An
  account switch produces fresh observations, so the clock advances naturally.

Pros: one field, already in the draft, displayable, survives restarts, matches Claude Code's internal practice and the
HTTP cache rule. Cons: depends on Agent clock discipline. The "identical on tie" rule is a new kind of obligation in
ACP.

### Option B: orderable `revision` per connection, `updatedAt` stays display-only

Normative sketch:

- `revision` (integer ≥ 0, required, < 2^53) increases by at least 1 whenever the Agent's quota knowledge changes.
  Snapshots with equal `revision` are identical.
- Revisions are comparable only within one connection. Clients MUST reset their kept revision on a new connection.
- Clients keep the snapshot with the greatest `revision`, per scope or connection-wide.
- `updatedAt` stays optional and display-only.

Edge cases:

- **Restart.** The counter resets. Persisted snapshots across restarts cannot be ordered, so the client falls back to
  "first snapshot of the new connection wins".
- **Sessionless read vs per-session push.** A single connection-wide counter makes them comparable even across HTTP
  streams. This is a plus over A only if A's clock is untrustworthy.
- **Multi-instance remote agents.** Needs a shared counter, or sticky routing. The HTTP RFD relies on cookies for
  affinity (`streamable-http-websocket-transport.mdx:49`).
- **Opaque vs integer.** An opaque string would allow equality only (the ETag and early Kubernetes problem), so it must
  be an integer or a decimal string.

Pros: exact, no clocks. Cons: a second field, not displayable, no agent has one today (each must add a counter), and
nothing in ACP is shaped like it.

### Option C: both (`revision` within a connection, `updatedAt` across connections)

- Within a connection, compare `revision`. Across connections or after a reload, compare `updatedAt`.
- Most robust, most rules. It also needs a statement of what to do if the two disagree (they cannot on one connection
  if `updatedAt` is clamped).

### Rejected alternatives worth recording in the RFD

- **Strengthened message order** ("Agents MUST NOT send a snapshot older than one already sent on the connection").
  This works on stdio if the Agent serializes snapshot assembly and sending. It fails across HTTP session and
  connection streams (`streamable-http-websocket-transport.mdx:35`), and it relies on client SDKs that do not all
  preserve order (TS-SDK `jsonrpc.ts:1221-1230`).
- **MCP-style invalidate-then-read.** `quota_update` would carry no data, and the client would call `quota/read`.
  This removes the race for pushes, but two concurrent reads can still race, and it doubles traffic.
- **LSP-style source precedence** ("read wins over push on tie" or "read always wins"). This works as a tie-break, as
  in Codex's TUI and LSP diagnostics, but not as the primary rule, because a read can be served from older cache.

## Implications for the RFD

- Replace `rate-limits-and-quotas.mdx:251-252` and `:509-512`. "Message order decides" is not implementable across
  transports, and it conflicts with v2's "don't infer from arrival order" stance (`prompt-lifecycle.mdx:185`).
- Whatever the option, define `updatedAt` as the **observation** time, not the send time. Codex's reuse of cached
  snapshots on pushes (`codex-active.md:55`) is the concrete trap.
- Pin the timestamp format (UTC `Z`, fixed millisecond digits) if string comparison is to be allowed. Otherwise
  require parsing. This follows CLAUDE.md:29 on nullable or required fields: if `updatedAt` becomes required, `null`
  is not allowed.
- State the comparison scope explicitly. Account data appears on several sessions and in sessionless reads.

## NEEDS_DECISION

1. **Freshness mechanism.**
   - **A.** Required, Agent-monotonic `updatedAt` with newer-or-equal wins.
     - Pro: one field, displayable, survives restarts, matches Claude Code.
     - Con: relies on clock discipline; ms ties need the "identical" rule.
   - **B.** Required integer `revision` per connection, `updatedAt` display-only.
     - Pro: exact.
     - Con: new field, resets on restart, not displayable, needs a shared counter for remote multi-instance agents.
   - **C.** Both.
     - Pro: exact within a connection and robust across them.
     - Con: two rules for clients.
   - Recommendation: **A**.
2. **Tie rule** (only for A).
   - (a) Equal `updatedAt` implies identical pools, so the Agent must advance the value. Recommended.
   - (b) Equal means later-received wins. Simpler for agents, but reintroduces order dependence.
   - (c) Equal means the `quota/read` result wins (LSP precedent).
3. **Comparison scope for a sessionless `quota/read`.**
   - (a) It is its own scope and never replaces per-session snapshots. Simple and safe, but the client may hold two
     views of one account.
   - (b) The client may compare it with a session's snapshot when it knows they describe the same account. This needs
     an account or scope identifier, which the draft does not have.
   - (c) Compare across all scopes on the same connection. Wrong if sessions use different accounts.
   - Recommendation: (a), unless an account identifier is added.
4. **Semantics of `updatedAt` when the Agent only learns about changes** (adapters on change-only events).
   - (a) "Last observed or confirmed". Agents that cannot confirm stamp the change time.
   - (b) Define it as "last changed".
   - Recommendation: (a), with a note that change-only agents give a conservative (older) value. This is safe for
     ordering.

## Gaps and unverified items

- UNVERIFIED: in Claude Code, `he.now()` (bin@189353159) is wall-clock ms. The inference rests on the same parameter
  defaulting to `Date.now()` (bin@187253310) and on being compared with `Date.now()`-based values.
- UNVERIFIED: whether Copilot's `timestamp_utc` is set by the server or by the CLI, and its resolution.
- No live capture of a push/read race in any agent. The race is inferred from the SDK dispatch models and agent code.
- I did not check the Kotlin or Python ACP SDKs for incoming-dispatch ordering.
- The Streamable HTTP transport is an RFD, not shipped protocol. Its stream split is as proposed in
  `docs/rfds/streamable-http-websocket-transport.mdx:35`.
- GraphQL: the "no cross-operation ordering" statement is based on its absence from spec §6, not on an explicit
  sentence.

# ACP update semantics: partial updates, upserts, and removal

Scope: an ACP protocol convention, not an agent. The question is how ACP represents partial updates, upserts, and
removal, so that a `quota_update` session update carrying several limits keyed by id can follow existing practice.
All paths are relative to the repository root. Schema paths are under `agent-client-protocol-schema/src/`.

## Summary

- ACP uses three update styles:
  1. **Full snapshot or replacement**: `usage_update`, `available_commands_update`, `config_option_update`,
     `current_mode_update` (v1 only), v1 `plan`, and the `entries` of `plan_update` items.
  2. **Per-id upsert or patch, one entity per notification**: v2 `tool_call_update`, `terminal_update`,
     `user_message`, `agent_message`, `agent_thought`, `plan_update` (keyed by `planId`, but the content is replaced
     wholesale), and the unstable `compaction_update`, `subagent_update`, and `session_message`.
  3. **Singleton patch with no id**: `session_info_update`.
- The patch convention is consistent across these updates. An omitted key means unchanged, `null` clears the field, a
  concrete value replaces it, and arrays and nested objects are replaced wholesale. For collections, `[]` and `null`
  both clear. Chunks append. In Rust this is modeled with `MaybeUndefined<T>`.
- v2 clearly moves toward upserts. The migration guide lists "Updates are upserts" as one of its five headline points.
- Removal of an entity has exactly one precedent: the separate `plan_removed` update, which is unstable. No update uses
  `null` to delete an entity, and no update today carries several keyed patches in one payload. A keyed-list upsert
  with removal would be new to ACP.

## Findings

### 1. `MaybeUndefined<T>`: definition

- Three-state enum `Undefined | Null | Value(T)`, with `Undefined` as the default (`serde_util.rs:578-586`, default at
  `:580`). It is documented as "undefined (key absent), null, or value" (`serde_util.rs:9`) and adapted from
  async-graphql (`serde_util.rs:16`). It is re-exported at `lib.rs:55`.
- Wire mapping:
  - Deserialize: `null` becomes `Null` and a value becomes `Value` (`serde_util.rs:766-779`). An absent key becomes
    `Undefined` through `#[serde(default)]`.
  - Serialize: `Undefined` must be skipped with `skip_serializing_if = "MaybeUndefined::is_undefined"`, which every
    usage does.
- Merge helper `update_to`: `Value` sets, `Null` clears, `Undefined` leaves the target unchanged
  (`serde_util.rs:704-728`).
- JSON Schema: the generated schema only says `type: [T, "null"]`, via `schemars(with = "Option<Option<T>>")`
  (`serde_util.rs:576`). The omitted vs `null` distinction exists **only in prose descriptions**. Example:
  `SessionInfoUpdate` in `schema/v2/schema.json`.
- Malformed values: almost every usage also applies `DefaultOnError`. A malformed value therefore deserializes to the
  default, `Undefined`, which is treated as "unchanged" rather than as an error. This is my inference from
  `serde_util.rs:580` plus the `x-deserialize-default-on-error` annotations; I found no test that pins it down.
- Repository rule: CLAUDE.md:29 says "For any nullable field, explicitly define whether it is required or optional and
  whether `null` is equivalent to an omitted key".

### 1a. Fields that use `MaybeUndefined`

Unless the Meaning column says otherwise, every field below follows the standard rule: omitted = unchanged,
`null` = clear, value = replace.

**v1**

| Field(s) | Location | Meaning |
| --- | --- | --- |
| `SessionMessage.content`, `_meta` (unstable subagents) | `v1/client.rs:306`, `:317` | Standard rule. For `content`, `null` and `[]` both clear, and a non-empty array replaces all content (`:301-302`). |
| `CompactionUpdate.summary`, `error`, `_meta` (unstable) | `v1/client.rs:818`, `:823`, `:832` | Standard rule. `summary: []` also clears (`:798-801`). |
| `SubagentUpdate.title`, `description`, `capabilities`, `state`, `_meta` (unstable) | `v1/client.rs:958`, `:967`, `:977`, `:987`, `:1001` | Standard rule. A concrete `capabilities` or `state` object replaces the previous one wholesale. Clearing `state` does not imply idle (`:934-940`, `:979-983`). |
| `SessionInfoUpdate.title`, `updatedAt` | `v1/client.rs:1778`, `:1783` | "Set to null to clear." |
| `SessionInfoUpdate._meta` | `v1/client.rs:1793` | Exception: this is a plain `Option<Meta>`, not `MaybeUndefined`, so v1 code cannot tell omitted from `null`. The RFD nevertheless says `null` is an explicit clear (`docs/rfds/session-info-update.mdx:280-282`). |
| `McpError.data` | `v1/mcp.rs:30` | Not an update. This is a value-preservation case: "explicit null is retained" (`v1/mcp.rs:18-19`). |

v1 `ToolCallUpdateFields` does **not** use `MaybeUndefined`. All its fields are `Option`
(`v1/tool_call.rs:261-300`), so `null` behaves like omission. The docs confirm that v1 cannot clear `name`
(`docs/protocol/v1/tool-calls.mdx:128-131`).

**v2**

| Field(s) | Location | Meaning |
| --- | --- | --- |
| `ToolCallUpdate.name`, `title`, `kind`, `status`, `content`, `locations`, `rawInput`, `rawOutput`, `_meta` | `v2/tool_call.rs:49-99` | Standard rule (struct doc at `:23-28`). Arrays are replaced wholesale, and `[]` and `null` both clear. For an unseen id, omitted fields use client defaults. Merge code: `apply_update` at `v2/tool_call.rs:193-222` keeps `Null` as `Null` "so callers can decide how to render". |
| `TerminalUpdate.command`, `cwd`, `output`, `exitStatus`, `_meta` | `v2/terminal.rs:205-233` | Standard rule (doc at `:187-192`). For a new id, omitted fields start unknown. `output` is an authoritative replacement snapshot. `apply_update` is at `:289-307`. |
| `SessionInfoUpdate.title`, `updatedAt`, `_meta` | `v2/client.rs:1348`, `:1353`, `:1366` | Standard rule. Unlike v1, `_meta` is now `MaybeUndefined` (doc at `:1335-1336`). |
| `UserMessage`, `AgentMessage`, `AgentThought`: `content`, `_meta` | `v2/client.rs:1943/1956`, `:2015/2028`, `:2087/2100` | Standard rule. `content` is replaced as a whole array, and `[]` or `null` clears it. Later chunks append (`:1919-1930`). |
| `SessionMessage`, `CompactionUpdate`, `SubagentUpdate` (unstable) | `v2/client.rs:316/327`, `:787/792/801`, `:936-974` | Same as in v1. |
| `McpError.data` | `v2/mcp.rs:30` | Same as in v1. |

Fields that are explicitly **not** patch fields use `Option`, where omitted and `null` are equivalent. These are
chunk-scoped `_meta` (for example `TerminalOutputChunk`, `v2/terminal.rs:320-324`), `Notice.description` and `_meta`
(`v2/client.rs:683-693`), and nested snapshot `_meta` (`docs/protocol/v2/migration.mdx:759`).

### 2. Update patterns and how they are classified

| Update | v1 | v2 | Class | Removal |
| --- | --- | --- | --- | --- |
| `tool_call` + `tool_call_update` | `v1/tool_call.rs:208-213`. A patch by `toolCallId`. `null` is the same as omitted. Collections are "overwritten, not extended" (`:249-252`). | `tool_call` is removed. `tool_call_update` is the single upsert (`docs/rfds/v2/tool-call-updates.mdx:11`, `:45-57`). Chunks append (`:68-76`). | Per-id upsert or patch | None. Tool calls are never deleted, only given a terminal `status`. |
| `session_info_update` | `v1/client.rs:1767-1794` | `v2/client.rs:1330-1367` | Singleton patch with no id | Fields cleared with `null` |
| `usage_update` | `v1/client.rs:1830-1852`. `used` and `size` are required, `cost` is `Option`. | `v2/client.rs:1403-1425`, identical to v1 | Full snapshot of required values | Not defined. If size is unknown, "should not send" (`docs/rfds/session-usage.mdx:67`, `:205`). |
| `plan` (v1) | `v1/plan.rs:33-47`: "The client replaces the entire plan with each update" (`:36-37`) | Removed in v2 (`docs/rfds/v2/plan-variants.mdx:53`) | Full replacement | Send an empty list |
| `plan_update` | Unstable (`v1/plan.rs:109`, `:155`, `:197`) | Stable for `items` (`v2/plan.rs:42`, `:81`, `:228`) | Keyed by `planId`. The plan's content is replaced wholesale, and items-based plans send the complete `entries` list (`v2/plan.rs:233-234`; `docs/protocol/v2/agent-plan.mdx:106-108`). | `plan_removed { planId }` |
| `plan_removed` | Unstable (`v1/plan.rs:361`) | Unstable (`v2/plan.rs:392`) | Separate removal event | This is the removal mechanism. The RFD chose it over a `type: "removed"` variant inside `plan_update` because it "keeps the schema cleaner" (`docs/rfds/plan-operations.mdx:37`, `:59-61`, `:142-153`). |
| `available_commands_update` | `v1/client.rs:2028-2041`. The v1 docs are vague: "update the list" (`docs/protocol/v1/slash-commands.mdx:73`). | `v2/client.rs:2156`. "complete replacement list, not a delta. An empty array clears" (`docs/protocol/v2/slash-commands.mdx:83`, `:111`). The initial list can come in the `session/new` or `session/resume` response (`:12`). | Full replacement | Leave the item out of the list. `[]` clears everything. |
| `config_option_update` | `v1/client.rs:1724-1737`. "The full set of configuration options" (`:1730`). Docs: "complete configuration state" (`docs/protocol/v1/session-config-options.mdx:351`). | `v2/client.rs:1291`, identical. Docs: `docs/protocol/v2/session-config-options.mdx:365`. | Full replacement | Leave the item out of the set |
| `current_mode_update` | `v1/client.rs:1680-1691`, a scalar | Removed and folded into config options (`docs/protocol/v2/migration.mdx:780`) | Scalar replacement | Not applicable |
| `notice` (unstable) | `v1/client.rs:690` | `v2/client.rs:661`. "Fire-and-forget". No id. | Event, not state | Not applicable |

The ACP schema has no **map-keyed or keyed-list patch inside a single update**. The only `BTreeMap` fields in v1 and
v2 payloads are flattened "Other" or extension catch-alls (for example `v2/client.rs:1760`), not keyed entity
collections.

### 3. Guidance in docs and RFDs

- **v2 headline**: "**Updates are upserts.** Messages, tool calls, and plans are patched by ID with uniform semantics:
  omitted field = unchanged, `null` = cleared, value = replaced, chunks append" (`docs/protocol/v2/migration.mdx:13`).
- **Identity**: "New entities introduced in v2 follow the same rule (`messageId` on messages, `planId` on plans)"
  (`migration.mdx:750`). Id fields are named `<entity>Id`.
- **SDK checklist**: "Model omitted vs `null` vs concrete values distinctly wherever v2 defines patch semantics. A
  plain nullable/optional type erases a distinction the protocol depends on" (`migration.mdx:807`).
- **Arrays**: "Array fields such as `content` and `locations` are replaced wholesale when present"
  (`migration.mdx:396`). v1 treats `null` like omission, so a strict v2-to-v1 adapter must reject clears it cannot
  represent (`migration.mdx:396-402`; `docs/rfds/v2/tool-call-updates.mdx:97`).
- **Why `null` differs from omission**: "Without the distinction, Agents cannot unset a previously supplied field
  without resending a full replacement object" (`tool-call-updates.mdx:109-111`). The RFD also credits
  `session_info_update` as the place where unsetting was established (`tool-call-updates.mdx:39`).
- **Why upserts and not snapshots**: upserts let agents update `_meta` or future fields without resending content
  (`docs/rfds/v2/message-updates.mdx:150-152`, `:44-53`).
- **Terminal upserts**: the same rules apply, and on a first-seen id omitted fields start unknown or empty
  (`docs/rfds/v2/terminal-output.mdx:70-83`).
- **Compaction**: on a first-seen id, "omission and `null` both start with no value"
  (`docs/rfds/session-compaction.mdx:119-136`).
- **Subagents**: the RFD describes itself as "a single upsert-style `subagent_update` following the v2 entity pattern"
  (`docs/rfds/subagents.mdx:1197`), and `session_message` is an upsert (`:422-425`).
- **`_meta` in upserts**: top-level `_meta` follows the patch semantics. Nested or chunk `_meta` does not: there,
  omitted and `null` are equivalent (`migration.mdx:759`).
- **Usage RFD**: it explicitly defers rate limits and quotas to a future RFD (`docs/rfds/session-usage.mdx:207-209`).

## Implications for the RFD

These are options, not decisions. All three are consistent with current ACP conventions to different degrees.

1. **Full snapshot list.** This follows the `available_commands_update` and `config_option_update` precedent:
   `{ "sessionUpdate": "quota_update", "quotas": [ {...}, ... ] }` is the complete replacement list, `[]` clears
   everything, and a limit is removed by leaving it out. Optionally, an initial list is returned in session setup
   responses (`docs/protocol/v2/slash-commands.mdx:12`). This is the simplest option. It needs no `MaybeUndefined`,
   but every update must resend all limits.
2. **One entity per notification, as a per-id upsert.** This follows `tool_call_update`, `terminal_update`, and
   `plan_update`, the dominant v2 pattern: `{ "sessionUpdate": "quota_update", "quotaId": "...", <patch fields> }`.
   - Fields use `MaybeUndefined` with omitted, `null`, and value semantics. Nested objects (for example a `window`)
     and arrays are replaced wholesale. On a first-seen id, omitted fields start unknown.
   - Removal follows the `plan_removed` precedent: a separate `quota_removed { quotaId }` update. This precedent is
     unstable and debated (`docs/rfds/plan-operations.mdx:142-153`).
3. **Keyed list of per-id patches in one payload** (what the task describes). There is **no precedent** for this in
   ACP. If chosen, the conventions above imply:
   - Each entry has a required `<entity>Id` and v2-style three-state patch fields, and is merged into stored state by
     id (as `apply_update` does).
   - Ids not mentioned in the payload stay unchanged. This makes the outer list a delta, the opposite of the
     `available_commands_update` and `config_option_update` lists, so the RFD must say so explicitly to avoid
     confusion.
   - Removal has to be explicit. In ACP, `null` on a field never deletes the entity; it clears that field only. See
     `SubagentUpdate` ("Clearing state ... does not imply idle", `v1/client.rs:934-940`). A per-entry `null` meaning
     "delete" would therefore be a new semantic.
   - Options consistent with current practice: a parallel `removedQuotaIds: [...]` field, or a separate
     `quota_removed` update (`plan_removed` style). A `removed` discriminator inside the entry is the alternative the
     plan RFD rejected.
   - A hybrid could add a `replace: true` marker or similar to send a full snapshot. Nothing in ACP does this today.
4. **For any option**:
   - Name ids `<entity>Id` (`migration.mdx:750`).
   - State omitted vs `null` behavior per nullable field (CLAUDE.md:29).
   - Remember the JSON Schema cannot express the three-state distinction, so prose must carry it
     (`serde_util.rs:576`).
   - Decide whether a malformed patch field (handled by `DefaultOnError`, so it becomes `Undefined`) silently means
     "unchanged".
   - For v1 back-compat, use plain `Option` only if `null` is not needed as a clear, since v1 `ToolCallUpdateFields`
     cannot clear.

## Gaps

- `usage_update` never says whether omitting `cost` in a later update clears a previously reported cost or leaves it
  unchanged. It is a snapshot with required `used` and `size`, but the `cost` behavior is unspecified
  (`docs/rfds/session-usage.mdx:69-73`; `v1/client.rs:1842-1845`). UNVERIFIED either way.
- `docs/rfds/plan-operations.mdx:35`, `:91`, and `:130` use `id` in their JSON examples, but the schema and protocol
  docs use `planId` (`v1/plan.rs:197`ff; `docs/protocol/v2/agent-plan.mdx:58`; `docs/protocol/v1/draft/agent-plan.mdx:135`).
  The RFD text is stale; the schema is authoritative.
- v1 `SessionInfoUpdate._meta` is `Option<Meta>` (`v1/client.rs:1793`), which contradicts the RFD's statement that
  `null` is an explicit clear (`docs/rfds/session-info-update.mdx:280-282`). Only the v2 code implements the
  distinction.
- I found no test that asserts a malformed `MaybeUndefined` patch value becomes `Undefined`. The behavior is inferred
  from `DefaultOnError` plus `#[default] Undefined`.
- I did not check whether the TypeScript or Zod outputs preserve the three-state distinction (out of scope).

# Orchestrator state: Rate limits and quotas RFD

## Phase

6. Review (draft written: `docs/rfds/rate-limits-and-quotas.mdx`)

## Agent list

| Agent       | Kind      | Status      |
|-------------|-----------|-------------|
| Claude Code | mandatory | done |
| Codex       | mandatory | done |
| Gemini CLI  | mandatory | done |
| Copilot CLI | candidate | done: SDK `account.getQuota` (chat/completions/premium_interactions; entitlement, used, remaining %, overage, monthly resetDate); push `session.error` rate_limit/quota codes; nothing over ACP |
| Cursor | candidate | done: no ACP/CLI quota; monthly pools via undocumented poll endpoint only |
| Amp | candidate | skipped: no structured quota; only `amp usage` balance text |
| Kimi CLI | candidate | done: `/usage` polls 5h/weekly windows (used/limit/remaining/reset); nothing over ACP |
| Qwen Code | candidate | skipped: no quota fields; free tier discontinued; string-matched quota errors only |
| Cline | candidate | done: nothing over ACP; structured spend-limit 429 (period, limit, spent, resets_at) and balance poll |
| OpenCode | candidate | done (minor): ACP has only usage_update; gateway 429 metadata `limitName` 5h/weekly/monthly dropped |
| Goose | candidate | skipped: no quota reporting; only `credits_exhausted` JSON-RPC error data |

## Research tasks

| Task | Model | Status | Note |
|------|-------|--------|------|
| Claude Code (a): native data model and delivery | opus | done | `.agents/research/claude-code-native.md` |
| Claude Code (b): ACP adapter output and relation to usage/context | opus | done | `.agents/research/claude-code-acp-adapter.md` |
| Codex (a): native data model and delivery | opus | done | `.agents/research/codex-native.md` |
| Codex (b): ACP adapter output and relation to usage/context | opus | done | `.agents/research/codex-acp-adapter.md` |
| Gemini CLI (a): native data model and delivery | opus | done | `.agents/research/gemini-native.md` |
| Gemini CLI (b): ACP integration output and relation to usage/context | opus | done | `.agents/research/gemini-acp.md` |
| Copilot CLI triage | sonnet | done | `.agents/research/copilot-cli.md` |
| Cursor triage | sonnet | done | `.agents/research/cursor.md` |
| Amp triage | sonnet | done | `.agents/research/amp.md` |
| Kimi CLI triage | sonnet | done | `.agents/research/kimi-cli.md` |
| Qwen Code triage | sonnet | done | `.agents/research/qwen-code.md` |
| Cline triage | sonnet | done | `.agents/research/cline.md` |
| OpenCode triage | sonnet | done | `.agents/research/opencode.md` |
| Goose triage | sonnet | done | `.agents/research/goose.md` |
| Synthesis (orchestrator) | — | done | `.agents/research/synthesis.md` |

## Decisions

| Date | Question | Answer |
|------|----------|--------|
| 2026-10-05 | Which candidate agents to research? | All of them: Copilot CLI, Cursor, Amp, Kimi CLI, Qwen Code, Cline, OpenCode, Goose. Each one gets a Sonnet triage first and is skipped if it has no quota reporting. |
| 2026-10-05 | Task granularity for mandatory agents | 2 tasks per agent: (a) native data model and delivery; (b) ACP adapter output plus relation to token usage/context. |
| 2026-10-05 | D1 Delivery | Push `quota_update` session update **plus** a client-to-agent `quota/read` request. All quota functionality is behind an agent capability: the agent must explicitly advertise that it has quota and supports reporting it. |
| 2026-10-05 | D3 Update semantics | Keyed list in one update: `quota_update` carries `limits[]`, each entry a keyed patch (omitted = unchanged, `null` = clear, value = replace). This is new in ACP; the RFD must justify it. `quota/read` returns the full list. Removal mechanism still to define. |
| 2026-10-05 | D4 Status/severity | Optional agent-graded `status` (`ok` / `warning` / `exceeded`), open enum with `_`-prefixed custom values. Clients may derive their own when it's absent. |

| 2026-10-05 | D5 Value fields | All optional: `usedPercent` (used share, 0-100, may exceed 100), `used` / `limit` / `remaining` with `unit`. Agents SHOULD send `usedPercent` when known. A limit must carry at least one of `usedPercent`, `used`, `remaining` or `status`. Agents never synthesize values. |
| 2026-10-05 | D7 Limit-hit errors | New ACP error code (e.g. `quotaExceeded`) with optional `data` `limitId`, `resetsAt`, `retryAfter`. The agent also pushes `quota_update` with `status: exceeded` when it can. |
| 2026-10-05 | D8 Client capability | Both sides: the agent advertises quota support; the client advertises it can display quota (`session.quota`, like `session.notices`). Push only when both are set; `quota/read` needs only the agent capability. |
| 2026-10-05 | D9 Protocol version | v1 unstable + v2. |
| 2026-10-05 | D10 Window duration | ISO 8601 duration string (`PT5H`, `P7D`, `P1M`). Timestamps: RFC 3339 in v2, ISO 8601 in v1. |
| 2026-10-05 | D6 Overage | Direction: a separate `overage` entity next to `limits[]` (own state: allowed / active / disabled reason, optional meter, list of covered limits). Same keyed-list upsert semantics as limits. `quota/read` returns the full information, including overages. Pivot if research shows incompatible formats. Common fields: pending research (milestone 3). |
| 2026-10-05 | D3 revised | Pivot from patch semantics to full snapshots for simpler updates (scope to confirm: whole `quota_update` or overage only). Removal and `unlimited` questions become moot. |
| 2026-10-05 | D6a Overage caps | `cappedBy` references rejected as confusing (two kinds of meters). User leans toward a nested list of meters, but meters must not be split between overage root fields and the nested list. Brainstorming. |
| 2026-10-05 | D6b Overage state reason | A field named just `reason` is confusing; the name must say it explains unavailability. |
| 2026-10-05 | D6c Amounts | Reuse the existing `Cost` type. |
| 2026-10-05 | D6d Consent | Out of scope. |
| 2026-10-05 | D6e Overage fields | Drop `unlimited` (no patch semantics). Drop `covers`; marking an overage `active` is enough. |
| 2026-10-06 | D11 Structure | Option B: everything is a pool holding meters; amounts only inside meters. User questions a `kind` (included vs overage) field: to a client all pools are just limits; prefer no `kind`. An `active` marker is wanted; reliability across agents under research. |
| 2026-10-06 | D3 final | Full snapshots for the whole `quota_update`; same shape as the `quota/read` result. |
| 2026-10-06 | D6c revised | Brainstorm a new generic amount type (`{amount: decimal, unit: string}`) vs a union with `Cost`; research where `Cost` is used. |
| 2026-10-06 | D12 Root shape | User asks whether the root can be a bare list of pools instead of an object with a `pools` field; check ACP conventions. |
| 2026-10-06 | D13 Active marker | Drop `active` from this RFD (no agent can fill it reliably). Encoding question is moot. |
| 2026-10-06 | D14 State vs status | Both (pool encoding reopened, see D19): meter `status` (`ok` / `warning` / `exceeded`) and pool `state` (`available` / `unavailable`) with `unavailableReason`. |
| 2026-10-06 | D15 Amount type | `Quantity { value: decimal string, unit }`; `unit` is ISO 4217 for money or a lowercase open enum (`credits`, `requests`, `tokens`). `Cost` stays unchanged in `usage_update`. |
| 2026-10-06 | D12 Root shape | Objects, no extra nesting: `{sessionUpdate: "quota_update", pools: [...]}` and `quota/read` result `{pools: [...]}`. |
| 2026-10-06 | D11 kind | No `kind` field; all pools are just limits. Pool order is meaningful (agent's display order). |
| 2026-10-06 | D16 As-of time | Optional `updatedAt` on the snapshot root (when the agent last got the data); display only, never an ordering key. |
| 2026-10-06 | D7 refined | `quotaExceeded` error `data` (all optional): `poolId`, `meterId`, `resetsAt`, `retryAfter` (ISO 8601 duration). |
| 2026-10-06 | D17 Ordering | Pools and meters are in display order; clients SHOULD keep it. |
| 2026-10-06 | D18 `quota/read` scope | Option 1: optional `sessionId`. Without it the agent answers for the account it would use for a new session; with it, for that session. The push carries pools for its own session. |
| 2026-10-06 | D19 Pool state | One optional open enum `state`: `available` / `exhausted` / `not_enabled` / `not_permitted` / `not_eligible` / `_custom`. Only `available` means usable; any other value (including unknown and `_` values) means not usable. Absent = the agent doesn't know (clients make no availability claim). Replaces `state` + `unavailableReason`. |
## Open questions

Choices made in the draft without an explicit user decision (to confirm in review):
- Agent capability placement: top-level `agentCapabilities.quota` (v1) / `capabilities.quota` (v2), like `providers`.
- Client capability `session.quota` in both v1 and v2 (v2 could drop it thanks to `SessionUpdate::Other`).
- `quota_update` is not replayed on `session/load` / `session/resume`.
- Error code number: `-32003` used as a placeholder.
- `quotaExceeded` MAY be returned without the quota capability (ids then omitted).
- Agents SHOULD NOT send identical consecutive snapshots; MAY push at session start.
- Adapters keep vendor `_meta` keys for one compatibility cycle; stabilize after two Agents and one Client ship.
- FAQ: API-key rate-limit headers MAY map to `PT1M` meters (optional).
- RFD navigation registration (docs config) not done.

## Next step

Get the user's review of the draft and iterate.

# Orchestrator state: Rate limits and quotas RFD

## Phase

6. Review. Review round 1 applied (freshness, v2 capability, optional unit, per-pool mappings). Waiting for the
   user's next round of feedback.

## Resume notes

- Workflow: `.agents/prompt.md`. The orchestrator **never researches itself**, not even quick greps of this repo
  (schema, Rust types, docs). Spawn `quota-researcher` for everything (Sonnet for simple lookups); ACP convention
  notes go to `.agents/research/acp-<topic>.md`.
- `.agents/` is gitignored. Stage explicit paths with `git add -f` (`.agents/research/*.md`, `.agents/state.md`, the RFD,
  and `.agents/prompt.md` / `.agents/agents/quota-researcher.md` when the user asks to change them). Push to
  `eugenethedev/rfd-rate-limits-quotas` only, never force.
- Before every commit, scan notes and the RFD for internal names:
  `grep -rniE "jetbrains|IJAI|air-ext|AirExt|plugins/air|JbCentral|AcpSessionEventMapper|ThreadGui|SessionEventStore|_meta\.jetbrains"`.
- Do not run `npm run generate` / `npm run check` unless asked.

## Agent list

| Agent       | Kind      | Status |
|-------------|-----------|--------|
| Claude Code | mandatory | done |
| Codex       | mandatory | done |
| Gemini CLI  | mandatory | done |
| Copilot CLI | candidate | done: SDK `account.getQuota` per category (entitlement, used, remaining %, overage, monthly reset); `session.error` rate_limit/quota codes; nothing over ACP |
| Cursor      | candidate | done: monthly pools + on-demand via undocumented poll endpoint only; nothing over ACP |
| Kimi CLI    | candidate | done: `/usage` polls 5h/weekly rows (used/limit/remaining/reset); nothing over ACP |
| Cline       | candidate | done: nothing over ACP; spend-limit 429 error data; balance poll; no overage fallback |
| OpenCode    | candidate | done (minor): ACP has only `usage_update`; gateway 429 `limitName` dropped |
| Amp         | candidate | skipped: no structured quota; only `amp usage` balance text |
| Qwen Code   | candidate | skipped: no quota fields; free tier discontinued; string-matched quota errors |
| Goose       | candidate | skipped: no quota reporting; only `credits_exhausted` error data |

## Research tasks

All tasks are done. No researcher is running.

| Milestone | Task | Model | Note |
|-----------|------|-------|------|
| 1 | Claude Code native model | opus | `claude-code-native.md` |
| 1 | Claude Code ACP adapter | opus | `claude-code-acp-adapter.md` |
| 1 | Codex native model | opus | `codex-native.md` |
| 1 | Codex ACP adapter | opus | `codex-acp-adapter.md` |
| 1 | Gemini CLI native model | opus | `gemini-native.md` |
| 1 | Gemini CLI ACP output | opus | `gemini-acp.md` |
| 1 | Triage: Copilot CLI, Cursor, Amp, Kimi CLI, Qwen Code, Cline, OpenCode, Goose | sonnet | `<agent>.md` |
| 1 | Cross-agent synthesis (orchestrator) | — | `synthesis.md` (sections 1-4) |
| 2 | ACP update/upsert semantics | opus | `acp-update-semantics.md` |
| 2 | ACP capabilities, scoping, open enums, time conventions | sonnet | `acp-capabilities.md` |
| 3 | Overage: Claude Code, Codex, Gemini CLI | opus | `<agent>-overage.md` |
| 3 | Overage: Copilot CLI, Cursor, Cline | sonnet | `<agent>-overage.md` |
| 3 | Overage synthesis (orchestrator) | — | `synthesis.md` (section 5) |
| 4 | Active-pool deducibility: Claude Code, Codex, Gemini CLI | opus | `<agent>-active.md` |
| 4 | Active-pool deducibility: Copilot CLI, Cursor | sonnet | `<agent>-active.md` |
| 4 | ACP `Cost` usage and root-shape conventions | sonnet | `acp-amounts-and-root.md` |
| 5 | ACP precedents for collapsed status enums vs boolean + reason | sonnet | `acp-state-enums.md` |
| 6 | ACP error codes and capability gating | sonnet | `acp-capability-errors.md` |
| 6 | Quota freshness: push vs poll ordering | opus | `acp-quota-freshness.md` |

All notes are in `.agents/research/`.

## Current design (as written in the draft)

- **Delivery:** a `quota_update` session update (full snapshot) plus a client-to-agent `quota/read` request; both return
  `QuotaSnapshot { pools, updatedAt?, _meta? }`. No extra nesting.
- **Capabilities:** the agent advertises `quota: {}` (top-level). In v1 the client advertises `session.quota: {}`; v2
  has no client capability. Push needs the agent capability (plus the client one in v1); `quota/read` needs only the
  agent capability.
- **Freshness:** required `updatedAt` = observation time, non-decreasing per Agent process; greatest wins per scope;
  equal means identical pools. A sessionless `quota/read` stands alone. First snapshot per scope on a new connection
  is accepted.
- **Full snapshots:** each update replaces the previous one. No patch semantics. Omitted and `null` mean "not reported".
- **`QuotaPool`** `{ id, label?, state?, meters[] }`. No `kind` field and no `active` marker. Display order is meaningful.
- **`QuotaPoolState`**, an open enum: `available` / `exhausted` / `not_enabled` / `not_permitted` / `not_eligible` /
  `_custom`. Only `available` means usable. Absent means unknown.
- **`QuotaMeter`** `{ id, label?, usedPercent?, used?, limit?, remaining?, window? (ISO 8601 duration),
  resetsAt? (RFC 3339), status? }`. It needs at least one of `usedPercent` / `used` / `remaining` / `status`. Agents never
  synthesize values.
- **`QuotaMeterStatus`**, an open enum: `ok` / `warning` / `exceeded`.
- **`Quantity`** `{ value: decimal string, unit? }` (absent unit = not reported). `unit` is ISO 4217 for money (major units) or a lowercase unit
  (`requests`, `tokens`, `credits`). `Cost` is unchanged.
- **`quotaExceeded` error:** a new code (placeholder `-32003`). Its `data` is `{ poolId?, meterId?, resetsAt?, retryAfter? }`.
- **`quota/read`:** `sessionId` is optional. Without it, the agent answers for the account it would use for a new session.
- **Versions:** v1 unstable (`unstable_quota`) plus v2. Consent and purchase flows are out of scope.

## Decisions

Superseded rows are kept for history and marked "superseded".

| Date | ID | Question | Answer |
|------|----|----------|--------|
| 2026-10-05 | — | Candidate agents | All eight candidates triaged on Sonnet; skipped if no quota reporting. |
| 2026-10-05 | — | Task granularity | Two tasks per mandatory agent: native model, ACP output. |
| 2026-10-05 | D1 | Delivery | Push `quota_update` plus `quota/read`; everything behind an explicit agent capability. |
| 2026-10-05 | D3 | Update semantics | Superseded. First: a keyed list of patches. Final (2026-10-06): full snapshots for the whole `quota_update`. |
| 2026-10-05 | D4 | Meter status | Optional agent-graded `status` (`ok` / `warning` / `exceeded`), open enum. |
| 2026-10-05 | D5 | Value fields | All optional. `usedPercent` SHOULD be sent when known. At least one of `usedPercent` / `used` / `remaining` / `status`. Never synthesize. |
| 2026-10-05 | D6 | Overage | Superseded: a separate overage entity with `cappedBy`, `covers` and `unlimited`. Replaced by D11 (everything is a pool). Consent is out of scope. |
| 2026-10-05 | D7 | Limit-hit errors | New error code plus `status: exceeded` push. Data: `poolId`, `meterId`, `resetsAt`, `retryAfter` (refined 2026-10-06). |
| 2026-10-05 | D8 | Client capability | Both sides opt in: agent `quota`, client `session.quota`. |
| 2026-10-05 | D9 | Versions | v1 unstable plus v2. |
| 2026-10-05 | D10 | Window duration | ISO 8601 duration string. Timestamps RFC 3339 (v2) / ISO 8601 (v1). |
| 2026-10-06 | D11 | Structure | Option B: a flat list of pools, each holding meters; amounts only in meters; no `kind`. |
| 2026-10-06 | D12 | Root shape | Objects: `{sessionUpdate: "quota_update", pools}` and `{pools}`. A bare array is not ACP convention. |
| 2026-10-06 | D13 | Active marker | Dropped: no agent can fill it reliably. |
| 2026-10-06 | D14 | State vs status | Both pool `state` and meter `status`. Encoding refined by D19. |
| 2026-10-06 | D15 | Amount type | `Quantity { value: decimal string, unit }`. `Cost` is unchanged. |
| 2026-10-06 | D16 | As-of time | Superseded by D20. Optional `updatedAt` on the snapshot; display only. |
| 2026-10-06 | D17 | Ordering | Pools and meters are in display order. |
| 2026-10-06 | D18 | `quota/read` scope | Optional `sessionId`. Without it, the account for a new session. |
| 2026-10-06 | D19 | Pool state encoding | One optional open enum. Only `available` is usable. Absent means unknown. |
| 2026-10-06 | D20 | Freshness | Required `updatedAt` (observation time, non-decreasing); newer-or-equal wins; not message order. |
| 2026-10-06 | D21 | `updatedAt` tie | Equal `updatedAt` means identical pools; Agent bumps by 1 ms on change. |
| 2026-10-06 | D22 | v2 client capability | None in v2 (follow notices). v1 keeps `clientCapabilities.session.quota`. |
| 2026-10-06 | D23 | Unknown units | `Quantity.unit` optional; absent = not reported. |
| 2026-10-06 | D24 | Mapping section | Nested headings per agent and pool, with JSON examples; no table. |

## Open questions

Choices made in the draft without an explicit user decision; confirm them in review:

1. Agent capability placement: top-level `agentCapabilities.quota` (v1) / `capabilities.quota` (v2), like `providers`.
2. `quota_update` is not replayed on `session/load` / `session/resume`; clients call `quota/read`.
3. Error code `-32003` is a placeholder, and it falls in MCP's reserved `-32000..-32019` band. The MCP-over-ACP RFD
   uses `-33xxx`. Proposal: pick a code outside that band.
4. `quotaExceeded` MAY be returned without the quota capability (ids omitted). Research: no ACP error is
   capability-gated, so this is consistent. Proposal: keep.
5. Reference `docs/rfds/v2/prompt.mdx:288` (rate limits as a future `error` stop-reason category) from the RFD.
6. Freshness details chosen from research recommendations: a sessionless `quota/read` stands alone; change-only
   providers stamp the time they learned of the change; first snapshot per scope on a new connection is accepted.
7. Agents push when their data changes, MAY push at session start, and SHOULD NOT send identical consecutive snapshots.
8. Adapters keep vendor `_meta` keys for one compatibility cycle. Stabilize after two agents and one client ship.
9. FAQ: API-key per-minute rate-limit headers MAY map to `PT1M` meters.

## Next step

Get the user's answers on open questions 3-6 and their next review round of `docs/rfds/rate-limits-and-quotas.mdx`.

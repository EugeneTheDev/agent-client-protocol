# Orchestrator state: Rate limits and quotas RFD

## Phase

4. Decisions (synthesis written to `.agents/research/synthesis.md`)

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
## Open questions

- D6 Overage: common field set pending milestone 3 research. Consent (who decides to spend) not decided yet.
- D3 follow-up: how a limit is removed under the keyed-list update.

## Research tasks (milestone 2: ACP conventions)

| Task | Model | Status | Note |
|------|-------|--------|------|
| ACP update/upsert semantics (`MaybeUndefined`, existing update kinds) | opus | done | `.agents/research/acp-update-semantics.md` |
| ACP capability gating, method scoping, open enums, time/duration conventions | sonnet | done | `.agents/research/acp-capabilities.md` |

## Research tasks (milestone 3: overage)

| Task | Model | Status | Note |
|------|-------|--------|------|
| Claude Code overage | opus | done | `.agents/research/claude-code-overage.md` |
| Codex overage (credits, spend control) | opus | done | `.agents/research/codex-overage.md` |
| Gemini CLI overage (AI credits) | opus | done | `.agents/research/gemini-overage.md` |
| Copilot CLI overage (premium requests) | sonnet | done | `.agents/research/copilot-cli-overage.md` |
| Cursor overage (on-demand usage) | sonnet | done | `.agents/research/cursor-overage.md` |
| Cline overage (pay-as-you-go, spend limits) | sonnet | done | `.agents/research/cline-overage.md` |

## Research tasks (milestone 4: active marker, amounts, root shape)

| Task | Model | Status | Note |
|------|-------|--------|------|
| Claude Code: deducing the active pool | opus | done (overage: reliable w/ caveats; included: heuristic, multiple windows charged at once) | `.agents/research/claude-code-active.md` |
| Codex: deducing the active pool | opus | done (included: heuristic, exact only on 429; credits: poll-only) | `.agents/research/codex-active.md` |
| Gemini CLI: deducing the active pool | opus | done (included: reliable w/ caveats; credits: heuristic, after-the-fact consumedCredits) | `.agents/research/gemini-active.md` |
| Copilot CLI: deducing the active pool | sonnet | done (heuristic only) | `.agents/research/copilot-cli-active.md` |
| Cursor: deducing the active pool | sonnet | done (included: reliable w/ caveats for explicit model; Auto and on-demand: heuristic) | `.agents/research/cursor-active.md` |
| ACP: `Cost` usage and root-shape conventions | sonnet | done | `.agents/research/acp-amounts-and-root.md` |

## Next step

Ask the user: `active` semantics and encoding, amount type name/representation, pool `state` vs meter `status`. Then write the RFD draft.

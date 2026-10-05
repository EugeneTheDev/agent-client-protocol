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
| 2026-10-05 | D3 Update semantics | User leans toward per-limit upserts, consistent with existing patch semantics (`MaybeUndefined`). Pending research: `acp-update-semantics.md`. |
| 2026-10-05 | D4 Status/severity | Optional agent-graded `status` (`ok` / `warning` / `exceeded`), open enum with `_`-prefixed custom values. Clients may derive their own when it's absent. |

## Open questions

- D3 update semantics: confirm the upsert shape. Per `acp-update-semantics.md`: v2 says "updates are upserts"; per-id upserts carry one entity per notification; the patch rule is omitted = unchanged, `null` = clear, value = replace; the only removal precedent is a separate `plan_removed` update.
- D5 value fields: user asked for an explanation of what making `usedPercent` required would cost, and how "percent" is
  disambiguated across windows (hourly, daily, weekly). Explained in chat; awaiting a decision.
- D6 overage/credits/balances, D7 limit-hit errors, D8 capability details (now: agent capability, decided; client side
  still open), D9 protocol version: not asked yet.
- Decision 10 (timestamps): follow the existing convention, RFC 3339 `date-time` in v2 and "ISO 8601" in v1 (see `acp-capabilities.md`). ACP has no duration fields yet, so a window-duration unit is new (to be asked).

## Research tasks (milestone 2: ACP conventions)

| Task | Model | Status | Note |
|------|-------|--------|------|
| ACP update/upsert semantics (`MaybeUndefined`, existing update kinds) | opus | done | `.agents/research/acp-update-semantics.md` |
| ACP capability gating, method scoping, open enums, time/duration conventions | sonnet | done | `.agents/research/acp-capabilities.md` |

## Next step

Ask the remaining decisions (D3 confirm, D5, D6, D7, D8 client side, D9, duration unit) in one batch.

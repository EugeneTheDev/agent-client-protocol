# Orchestrator state: Rate limits and quotas RFD

## Phase

3. Synthesis (research milestone 1 complete: all 14 notes written, no HANDOFF or NEEDS_DECISION)

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

## Decisions

| Date | Question | Answer |
|------|----------|--------|
| 2026-10-05 | Which candidate agents to research? | All of them: Copilot CLI, Cursor, Amp, Kimi CLI, Qwen Code, Cline, OpenCode, Goose. Each one gets a Sonnet triage first and is skipped if it has no quota reporting. |
| 2026-10-05 | Task granularity for mandatory agents | 2 tasks per agent: (a) native data model and delivery; (b) ACP adapter output plus relation to token usage/context. |

## Open questions

(none)

## Next step

Synthesis: read the notes selectively, build the agent x field x semantics comparison table, and list the open design decisions to batch to the user.

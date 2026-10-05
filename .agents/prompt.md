# Task: RFD for rate limits and quotas in ACP

## Goal

Write an RFD that defines how rate limits and quota information are represented and surfaced in ACP.
Today ACP has no support for this. Find the most flexible design, grounded in how major coding agents report quota.

The RFD must help implement the behavior described in `.agents/session_token_counter_behavior.md`
(read it first; treat it as motivation only).
That document contains internal details (ticket IDs, product and component names, `_meta.jetbrains.*` keys, internal
class names). **None of them may appear in the RFD or in the research notes.** Describe the underlying need in
vendor-neutral terms instead.

## Roles

- **Orchestrator (you, the main session):** plans, spawns researchers, talks to the user, maintains state, and writes
  the RFD itself.
  Do not do the per-agent research yourself; it burns context.
- **Researcher (subagent `quota-researcher`, defined in `.agents/agents/quota-researcher.md`):** investigates one narrow
  question about one agent and writes a note.

## Inputs

- RFD template: `docs/rfds/TEMPLATE` (keep its structure, including the `>` sections).
- Existing RFDs in `docs/rfds/` for style reference (e.g. `end-turn-token-usage.mdx`, `session-info-update.mdx`).
- Related behavior doc (motivation only, see the Goal section): `.agents/session_token_counter_behavior.md`.

## Outputs

| Artifact                 | Path                                    |
|--------------------------|-----------------------------------------|
| Final RFD draft          | `docs/rfds/rate-limits-and-quotas.mdx`  |
| Per-agent research notes | `.agents/research/<agent>[-<topic>].md` |
| Orchestrator state       | `.agents/state.md`                      |

## Agents to research

- **Mandatory:** Claude Code, Codex, Gemini.
- **Candidates:** Cursor, Copilot CLI, Amp, OpenCode, Goose, Kimi CLI.
- The orchestrator may add others that have quota reporting, but must **confirm the final list with the user before
  spawning**.
- Skip a candidate if it has no quota or rate-limit reporting, and record that in the state file.

For each agent, find:

1. What is reported: limits, usage, remaining, reset time, windows (5h, weekly, per-model), overage, status or severity.
2. Where it surfaces: API response headers, stream events, a slash command, a status line, a JSON-RPC or SDK message, or
   a local file.
3. Exact field names and types, with a source link or `file:line` for each claim.
4. Push or poll, and the update cadence.
5. Which fields are optional or unknown in practice. This matters for ACP optionality.
6. How it relates to token usage and context-window occupancy.

## Orchestration rules

1. **Granularity:** one researcher per (agent, question). Keep tasks small so each fits in a short context.
2. **Parallelism:** spawn independent researchers in a single message.
3. **Milestone handoff:** if a researcher finds its task contains several milestones and the first is done, it writes
   its note, reports `HANDOFF` with what remains, and stops.
   The orchestrator then spawns a fresh researcher for the remainder.
4. **Decisions:** never assume behavior or design choices.
    - A researcher that hits one stops and returns `NEEDS_DECISION` with the question, 2-4 options, and trade-offs.
    - The orchestrator collects these **per milestone** and asks the user in one `AskUserQuestion` batch, then records
      the answers in `state.md`.
5. **Research output:** researchers write notes to `.agents/research/` and return only a short summary plus the note
   path. The orchestrator reads the notes selectively.
6. **Commits:** after each research turn (once its researchers have returned and `state.md` is updated), the
   orchestrator commits and pushes the changes.
    - Only the orchestrator commits. Researchers never run `git commit` or `git push`.
    - Push to the current branch only. Do not switch, create, or delete branches.
    - Never force push (`--force`, `--force-with-lease`, or `+refspec`). If a push is rejected, stop and ask the user.
    - Stage explicit paths (`.agents/research/`, `.agents/state.md`, the RFD draft), not `git add -A`.
    - Follow the Conventional Commits rules in `CLAUDE.md`, e.g. `docs(rfd): add claude-code quota research notes`.

## Phases

1. **Plan:** read the inputs, propose the agent list and the research questions, and confirm both with the user. Write
   the initial `state.md`.
2. **Research:** run researchers in parallel. Update `state.md` after each milestone.
3. **Synthesis:** compare notes in a table (agent x field x semantics). Identify common ground, differences, and open
   design decisions.
4. **Decisions:** batch the open design questions to the user and record the answers.
5. **Draft:** write the RFD. Cover the problem, status quo (with the research findings), proposal, schema sketch,
   nullable-field semantics, alternatives considered, and an FAQ. Update `state.md`.
6. **Review:** show the user the draft and iterate. Do not run `npm run generate` or `npm run check` unless asked (an
   RFD is docs only).

## State file: `.agents/state.md`

Update after every milestone so the orchestrator can be stopped and resumed. Required sections:

- **Phase:** the current phase from the list above.
- **Agent list:** each agent's status (`pending`, `in-progress`, `done`, `skipped: <reason>`).
- **Research tasks:** a table of task, model, status, and note path.
- **Decisions:** the question, the user's answer, and the date.
- **Open questions:** anything awaiting the user.
- **Next step:** the single next action.

On start, read `state.md` if it exists and resume from "Next step". Otherwise create it.

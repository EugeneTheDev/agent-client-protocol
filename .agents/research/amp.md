# Amp (ampcode.com): quota, credit and rate-limit reporting

## Summary

Amp has no machine-readable quota, credit or rate-limit reporting in its streaming interface.
It has exactly one programmatic-ish surface: the `amp usage` CLI command, which prints the signed-in account and a credit balance.
That command is documented only in one sentence and its output format is undocumented.
`UNVERIFIED`: whether `amp usage` has a JSON mode.

- The `--stream-json` output has per-message token usage and no cost, credit, balance, reset or rate-limit fields.
- The community ACP adapter `amp-acp` forwards no usage or quota data to ACP clients.
- A free-tier daily grant existed ($10/day, hourly replenishment) but Amp Free is now paused. It was never exposed as a structured field.
- Recommendation for the RFD: treat Amp as "no structured quota reporting".
  Use it only as evidence that some agents have a plain balance with no limit, window or reset.

## Fields

Only token usage exists in the stream. Nothing quota-related is structured.

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `usage.input_tokens` | number | `usage` itself is optional on `assistant` and `result` messages | Input tokens | https://ampcode.com/docs/cli/streaming-json |
| `usage.output_tokens` | number | same | Output tokens | same |
| `usage.cache_creation_input_tokens` | number | yes | Cache write tokens | same |
| `usage.cache_read_input_tokens` | number | yes | Cache read tokens | same |
| `usage.cache_creation.ephemeral_5m_input_tokens`, `.ephemeral_1h_input_tokens` | number | yes | Cache write breakdown | same |
| `usage.max_tokens` | number | yes | Max tokens (exact meaning not documented) | same |
| `usage.service_tier` | `'standard' \| 'enterprise'` | yes | Account tier hint. This is not a quota. | same |
| `result.is_error`, `result.error` (string), `subtype` (`error_during_execution` or `error_max_turns`) | bool / string | `error` only when failed | Failure info. Free-form string only; no code, no "out of credits" category documented. | same |
| `amp usage` command output | text (format undocumented) | n/a | Signed-in account and credit balance | https://ampcode.com/docs/pricing ("by running `amp usage`"); third-party description at https://github.com/yong076/agentcat-connectors/pull/74 (balance only, no included amount, no reset) |

The streaming-json docs page lists no cost, credits, balance, reset or rate-limit fields.
The page was read through a summarizing fetch tool, so the absence is fairly strong evidence but not a verbatim schema read.

## Delivery mechanism

- **Streaming JSON (`amp -x --stream-json`)**: pushes `assistant`, `user`, `result` and `system` messages. Token `usage` only. Source: https://ampcode.com/docs/cli/streaming-json
- **`amp usage`**: poll-only CLI command, run on demand. Balance only.
- **Web UI**: balance in user or workspace settings. Per-thread cost is the `$` price in the right sidebar of a thread page, and the user can ask Amp to "explain your usage". Source: https://ampcode.com/docs/pricing
- **Execute mode docs** (https://ampcode.com/docs/cli/execute-mode) mention no usage command, no credit errors and no exit codes.
- **Amp SDK** (`@ampcode/sdk` 0.1.0-20260918210405-g81edbf0): its README has no usage, credit or quota API. It only mentions "cost" in mode descriptions (`README.md:207-210` in the npm tarball).
- **ACP adapter** `amp-acp` 0.9.0 (npm, `dist/index.js`):
  - It maps Amp stream messages to `agent_message_chunk`, `agent_thought_chunk`, `tool_call` and `tool_call_update` (`to-acp.ts` section, ~line 19952 onward).
  - It never emits ACP `usage_update`. The `usage_update` literal at line 12702 is the bundled ACP SDK schema, not adapter logic.
  - On a failed `result` it sends the error string as an `agent_message_chunk` with text `Error: <message>` and returns `stopReason: "end_turn"` (~lines 232-243 of the `server.ts` section).
  - Only auth errors get special handling, via `isAuthError`, which requests authentication.
  - So an out-of-credits failure reaches the ACP client as plain assistant text.
- Other adapters were seen only in search results and their source was not read: `acp-amp` (SuperagenticAI), `edlsh/amp-acp`. The SuperagenticAI docs claim ACP use needs a paid credit balance and does not work on free credits. This is `UNVERIFIED` (secondary source).

## Windows and semantics

- Paid tiers: included usage resets at the end of each monthly billing period and does not roll over.
- Paid credits: added when included usage runs out, charged at actual model and tool usage with no markup for individuals and non-enterprise workspaces, and expiring 12 months after purchase. Workspace credits are pooled. Source: https://ampcode.com/docs/pricing
- Orbs (cloud machines) have a separate minute-based usage allowance. Same source.
- Pricing page rate limits: Hobby, Individual and Teams list "No Amp token fees or limits". Enterprise lists "Higher resource quotas and rate limits", and spend limits and per-user cost controls. Same source. No numbers or response fields are documented.
- Amp Free daily grant (historical):
  - $10 of credits per day, "replenishes hourly"; when used up, wait for the hourly reset or buy credits. Source: https://ampcode.com/news/amp-free-frontier (fetched with curl).
  - The grant was later reduced or paused for many users (update 2026-05-23, https://ampcode.com/news/amp-free-is-ad-free), and the program is now marked paused (https://ampcode.com/news/amp-free).
  - There is no documented field or command exposing the grant balance or reset time. `amp usage` is said to show only a balance.
- Relation to tokens and context: token `usage` in stream-json is per-message and independent of credits. Credits are in dollars. No context-window occupancy field is documented.

## Gaps and unverified items

- The `amp usage` output format and any `--json` flag are `UNVERIFIED`. The CLI binary is downloaded at install time and was not installed, because `amp` is not on this machine and the npm tarball holds only a wrapper (`@ampcode/cli`).
- The exact error text, error subtype or exit code for out-of-credits is not documented and not verified. It probably arrives as `result.error` (string), but that is `UNVERIFIED`.
- The rate-limit (HTTP 429) behavior for Amp Free ("There are some rate limits", https://ampcode.com/news/amp-free) has no documented surface.
- The `docs/orbs/sizes-and-costs` and `docs/enterprise/workspace-entitlements` pages were not read.
- Not checked: the Amp CLI's own settings or config for a status line showing balance.

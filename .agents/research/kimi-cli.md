# Kimi CLI: quota / usage / rate-limit reporting

Source: https://github.com/MoonshotAI/kimi-cli, shallow clone of `main` at commit `9ab1286b8fe4e6bcd116949a27ce5e0ac3389c82` (paths below are repo-relative).

## Summary

- Kimi CLI has a human-facing quota view: the `/usage` slash command (alias `/status`). It is shell-UI only and works only for the "Kimi Code" managed platform. It polls a REST endpoint `GET {base_url}/usages` on demand and renders progress bars. (`src/kimi_cli/ui/shell/usage.py:37-104`; docs: `docs/en/reference/slash-commands.md:93-97`)
- The ACP mode reports NO quota, usage or rate-limit data to the client. `/usage` is registered in the shell registry, not in the soul registry that ACP advertises. `StatusUpdate` and `StepRetry` are dropped by the ACP session. 429 is retried silently; terminal failures become a JSON-RPC `internal_error` with a stringified message.
- The `/usages` response schema is not defined in the repo. The parser is defensive and tolerant of several key spellings (see Fields). `UNVERIFIED` against a live server.

## Fields

### A. `/usages` REST payload (as parsed by the CLI; server schema not in repo)

Parsed by `_parse_usage_payload` / `_to_usage_row` / `_limit_label` / `_reset_hint` in `src/kimi_cli/ui/shell/usage.py:107-221`.

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `usage` | object (row) | yes | Summary row; default label "Weekly limit" | usage.py:113-116 |
| `limits` | array of items | yes | Additional rows | usage.py:118-137 |
| `limits[].detail` | object (row) | yes | If present, the row data is read from here, else from the item itself | usage.py:125-128 |
| `limits[].window` | object | yes | Window descriptor used only for label generation | usage.py:129-133 |
| `window.duration` (also on item/detail) | int | yes | Window length | usage.py:173 |
| `window.timeUnit` (also on item/detail) | string containing `MINUTE`/`HOUR`/`DAY` (substring match, e.g. `TIME_UNIT_MINUTE`) | yes | Unit; 300 MINUTE becomes label "5h limit" | usage.py:171-184 |
| row `limit` | int (coerced via `int()`) | yes | Quota size; missing becomes 0 | usage.py:143, 155 |
| row `used` | int | yes | Consumed | usage.py:145 |
| row `remaining` | int | yes | Fallback: `used = limit - remaining` if `used` absent | usage.py:146-149 |
| row `name` / `title` / `scope` | string | yes | Label (priority: item `name`/`title`/`scope`, then detail; row label uses `name`, `title`, else default) | usage.py:153, 167-169 |
| row `reset_at` / `resetAt` / `reset_time` / `resetTime` | ISO-8601 string (nanosecond fractions truncated) | yes | Absolute reset time; rendered as "resets in X" | usage.py:190-193, 202-221 |
| row `reset_in` / `resetIn` / `ttl` / `window` | int seconds | yes | Relative reset | usage.py:194-197 |

Derived (client side only): `% left = (limit - used) / limit`, color red at <=10%, yellow at <=30%, else green (usage.py:282-295). Row dropped if both `used` and `limit` are missing (usage.py:150-151).

No overage, plan name, status or severity field is parsed.

### B. Wire / internal events (not delivered over ACP)

| name | type | optional? | meaning | source |
|---|---|---|---|---|
| `StepRetry.n` | int | no | step number | `src/kimi_cli/wire/types.py:82-96` |
| `StepRetry.next_attempt`, `max_attempts` | int | no | retry counters (`max_retries_per_step` default 3, `src/kimi_cli/config.py:84`) | wire/types.py:85-90 |
| `StepRetry.wait_s` | float | no | backoff seconds (tenacity `wait_exponential_jitter(initial=0.3, max=5, jitter=0.5)`) | kimisoul.py:1227, 1760-1773 |
| `StepRetry.error_type` | string | no | exception class name | wire/types.py:93 |
| `StepRetry.status_code` | int or null | yes | HTTP status, e.g. 429 | wire/types.py:95 |
| `StatusUpdate.context_usage` (float %), `context_tokens`, `max_context_tokens`, `token_usage`, `message_id`, `plan_mode`, `mcp_status` | various, all nullable ("None means no change") | yes | Context-window occupancy and per-step token counts only; NO quota fields | wire/types.py:176-195 |
| `TokenUsage.input_other`, `output`, `input_cache_read`, `input_cache_creation` | int | first two required | per-step token usage | `packages/kosong/src/kosong/chat_provider/__init__.py:99-109` |

## Delivery mechanism

- Quota (`/usage`): pull / poll, user-initiated. One HTTPS GET to `{platform.base_url}/usages` with `Authorization: Bearer <api key or OAuth token>` (usage.py:82-104). No background polling, no push, no cache. Only for provider key that resolves to `KIMI_CODE_PLATFORM_ID`; otherwise prints "Usage is available on Kimi Code platform only." (usage.py:50-53, 82-92). Error mapping: 401 "Authorization failed", 404 "Usage endpoint not available" (usage.py:59-66).
- Not exposed over ACP: `acp/server.py:183-192` builds `available_commands_update` from `soul_slash_registry.list_commands()` only. The soul registry (`src/kimi_cli/soul/slash.py`: init, compact, clear/reset, yolo, afk, plan, add-dir, export, import, ...) does not contain `usage`. `/usage` is defined with the shell registry (`from kimi_cli.ui.shell.slash import registry`, usage.py:21, 37). Whether any ACP client can still reach it by typing `/usage` is not verified, but it is not advertised.
- Rate-limit (429) handling: 429 is in the retryable set `{429, 500, 502, 503, 504}` (kimisoul.py:1647-1658) and retried with exponential jitter backoff up to `max_retries_per_step`. No `Retry-After` / `x-ratelimit-*` header is read (grep for `retry-after`/`ratelimit` headers finds nothing relevant in `src`; `UNVERIFIED` for the `kosong` provider packages).
- In the shell UI, `StepRetry` renders as "Retrying after rate limit . attempt N/M . Xs" (`src/kimi_cli/ui/shell/visualize/_live_view.py:80-97`).
- In ACP, `StepRetry` and `StatusUpdate` are explicitly `pass` (`src/kimi_cli/acp/session.py:174-175, 184-185, 271-272`), so the client sees nothing during retries and no context/token usage either. No `usage_update` or `PromptResponse` usage is sent (grep of `src/kimi_cli/acp` finds none; `PromptResponse` carries only `stop_reason`: session.py:235, 238, 248).
- After retries are exhausted, an `APIStatusError` (incl. 429) becomes `acp.RequestError.internal_error({"error": str(e)})` (session.py:224-229). Only 401 on an OAuth session maps to `auth_required`. No structured 429 / quota code.
- Print mode (non-ACP): exit code `RETRYABLE` for 429/5xx (`src/kimi_cli/ui/print/__init__.py:438-447`).
- Telemetry classifies 429 as `"rate_limit"` (kimisoul.py:127-128); internal only.

## Windows and semantics

- The CLI does not hardcode windows. Labels come from the server: default summary label "Weekly limit" (usage.py:116), and `limits[]` windows derived from `duration` + `timeUnit` (e.g. 300 minutes becomes "5h limit"; code comment at usage.py:171). So the Kimi Code plan appears to expose a weekly summary plus rolling windows such as 5h, but the exact set is determined by the server response (`UNVERIFIED` live).
- Reset semantics: absolute timestamp or relative seconds, rendered as "resets in ..." or "reset" when past (usage.py:217-218).
- Quota units are opaque integers (`used`/`limit`/`remaining`); no unit field (tokens vs requests unknown).
- Relation to tokens/context: separate. Context-window occupancy and token counts live in `StatusUpdate` (shell status bar only: `src/kimi_cli/ui/shell/prompt.py:2255`); quota is a distinct REST call. No link between them.

## Gaps and unverified items

- The actual `/usages` response schema and example payload are not in the repo (only the tolerant parser). Server docs not located.
- Whether `/usage` can be invoked via ACP prompt text: not verified (only the advertised command list was checked).
- Whether kosong provider adapters (`packages/kosong`) surface rate-limit headers or `Retry-After`: not examined.
- Only the Kimi Code platform has `/usages`; other providers (Moonshot open platform, OpenAI-compatible, etc.) report nothing.
- No overage, plan/subscription tier, severity or status fields are consumed anywhere.

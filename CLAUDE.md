# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A Hermes Agent plugin that bridges Hermes sessions with BiBLE Atlas — an external memory/knowledge/skill storage service. The plugin automatically recalls relevant context before each LLM turn, captures conversation turns to BiBLE Atlas, and exposes 7 agent tools for manual search/save/retrieval.

## Commands

```bash
uv sync                          # install all deps (prod + dev) into .venv
uv run ruff check bible_hermes_plugin/   # lint
uv run basedpyright bible_hermes_plugin/ # type-check
uv run ruff format bible_hermes_plugin/  # format
uv run ruff check --fix bible_hermes_plugin/  # auto-fix lint issues
uv run pytest                              # run all tests
uv run pytest --cov=bible_hermes_plugin --cov-report=term-missing  # tests + coverage
uv run pytest tests/test_config.py -k "test_resolve"  # run a single test
```

Or via Makefile: `make lint`, `make typecheck`, `make check` (lint+typecheck), `make fix`, `make fmt`, `make test`.

## Architecture

### Entry point: `register(ctx)`

`bible_hermes_plugin/__init__.py` — Hermes calls this once at startup. It wires in everything:

1. CLI commands (`hermes bible setup|status`) — always registered, even before config
2. Resolves `BibleHermesConfig` from `BIBLE_ATLAS_BASE_URL` env var + `~/.hermes/config.yaml` `bible:` section
3. Creates `BibleAtlasClient` (synchronous httpx HTTP client for the BiBLE Atlas API)
4. Creates `SessionCaptureStore` (thread-safe in-memory buffer for conversation turns)
5. Registers 5 hooks, 7 agent tools, 1 slash command (`/bible`)

**Graceful degradation**: If `BIBLE_ATLAS_BASE_URL` is not set, only the CLI command is registered. All hooks and tools are skipped with a warning until the user runs `hermes bible setup --base-url <url> --write`.

### Recall pipeline (pre_llm_call hook)

```
user message + last 6 history turns
  → _build_recall_query() — joins recent text + user msg, strips code blocks/blobs, truncates to 2000 chars
  → _run_parallel_searches() — ThreadPoolExecutor calls memory/skill/knowledge search on BibleAtlasClient concurrently
  → normalize_hits() — extracts RecallHit objects from raw API payloads (`ranking.py`)
  → filter_rank_and_trim() — deduplicates, filters by min_score, scores with domain+recency+overlap boosts, slices to top_k
  → render_relevant_memories() — renders hits into `<relevant-memories>` XML block within injection_token_budget (`injection.py`)
  → injected as `{"context": "<relevant-memories>...</relevant-memories>"}` into the LLM call
```

Key files: `recall.py` → `ranking.py` → `injection.py`

### Session capture (post_llm_call hook)

```
post_llm_call → extracts user + assistant + tool_calls from conversation history
  → SessionCaptureStore.capture_turn() — appends CapturedTurn to per-session buffer
  → triggers async flush (daemon thread) when:
      - pending turns >= capture_commit_threshold_turns (default 8), OR
      - buffered chars >= capture_commit_threshold_chars (default 16K), OR
      - force_capture config is true (flushes every turn), OR
      - session ends/resets (blocking flush)
  → flush calls client.save_memory() with multipart import
  → hard cap at 4× threshold_chars — drops oldest turns to bound memory
```

Key file: `capture.py`

### Bypass

`bypass.py` — Sessions whose ID matches any regex in `bypass_session_patterns` skip both recall and capture. Checked in `on_session_start`, `pre_llm_call`, and `post_llm_call`.

### HTTP client

`http_client.py` — `BibleAtlasClient` wraps httpx with:
- JSON envelope unwrapping (`{status, result}`)
- Bearer auth via `BIBLE_ATLAS_TOKEN`
- Structured error mapping (HTTP status → `BIBLE_*` error codes)
- Multipart memory import with auto-generated metadata
- Task polling for async imports

### Config

`config.py` — `BibleHermesConfig` dataclass with all settings. Env vars take precedence over `~/.hermes/config.yaml` `bible:` section values. All fields have defaults matching `plugin.yaml`'s `config_schema`.

### Tools

`tools/` package — 7 agent tools using a factory pattern:
- `tools/memory.py` — `bible_memory_search`, `bible_memory_save`, `bible_memory_get`
- `tools/knowledge.py` — `bible_knowledge_search`, `bible_knowledge_list`
- `tools/skill.py` — `bible_skill_search`, `bible_skill_get`

Each tool has a schema dict (OpenAI function-calling format) and a handler factory that takes `BibleAtlasClient`. Handlers return JSON strings via `ok()`/`fail()` helpers from `tools/helpers.py`.

### Logging

`logging_utils.py` — Structured `ActionLogger` with start/done/fail tracking and duration. Logs go to `$HERMES_HOME/logs/bible-hermes-plugin.log`. Secret keys (token, authorization, api_key, password) are redacted.

## Key design decisions

- **Non-fatal failures**: All hook exceptions are caught and logged as warnings — a plugin failure never breaks the LLM turn.
- **Thread safety**: `SessionCaptureStore` uses per-session locks for flush and a global lock for session creation/lookup.
- **Duplicate detection**: Flush uses SHA256 hash of pending turns to skip re-committing identical content.
- **Type checker suppression**: `basedpyright` is set to lenient on external unknowns (`reportUnknownMemberType = false` etc.) since the Hermes `ctx` object has no stubs. Internal code is strictly checked.
- **Python 3.10 floor**: Uses `from __future__ import annotations`; `X | Y` union syntax is explicitly disabled (UP007 in ruff ignore) for compat.

## File manifest

| File | Role |
|------|------|
| `bible_hermes_plugin/__init__.py` | Entry point — `register(ctx)` wires all hooks/tools/CLI |
| `bible_hermes_plugin/config.py` | `BibleHermesConfig` dataclass + `resolve_config()` |
| `bible_hermes_plugin/http_client.py` | `BibleAtlasClient` — synchronous httpx wrapper |
| `bible_hermes_plugin/recall.py` | Recall pipeline — build query, parallel search, rank, render |
| `bible_hermes_plugin/ranking.py` | Hit normalization, dedup, scoring, prompt-injection detection |
| `bible_hermes_plugin/injection.py` | Renders `RecallHit[]` → `<relevant-memories>` XML |
| `bible_hermes_plugin/capture.py` | `SessionCaptureStore` — thread-safe turn buffer + async flush |
| `bible_hermes_plugin/bypass.py` | Session bypass check (regex on session_id) |
| `bible_hermes_plugin/cli.py` | `hermes bible setup|status` CLI commands |
| `bible_hermes_plugin/logging_utils.py` | `ActionLogger`, file-based logging with secret redaction |
| `bible_hermes_plugin/tools/__init__.py` | `register_tools()` — wires all 7 tools |
| `bible_hermes_plugin/tools/helpers.py` | `ok()`/`fail()` result builders, arg validation, hit extraction |
| `bible_hermes_plugin/tools/memory.py` | Memory tools (search/save/get) |
| `bible_hermes_plugin/tools/knowledge.py` | Knowledge tools (search/list) |
| `bible_hermes_plugin/tools/skill.py` | Skill tools (search/get) |
| `plugin.yaml` | Plugin manifest — name, tools, hooks, env vars, config schema |
| `pyproject.toml` | Build config (hatchling), deps, tool settings |
| `Makefile` | Shortcuts for lint/typecheck/test/fix/fmt |
| `deploy.sh` | Deployment script |
| `docs/bible-hermes-plugin-evolution.md` | Evolution analysis — architecture insights, future directions, design decisions |

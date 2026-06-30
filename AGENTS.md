# XutheringWavesUID — Agent Guide

This is a **GsCore bot plugin** for Wuthering Waves (鸣潮) game data queries. It runs as a plugin inside [gsuid_core](https://github.com/Genshin-bots/gsuid_core) and supports multi-platform bots (QQ, Discord, Telegram, KOOK, FeiShu, etc.).

## Architecture

- **Plugin name**: `XutheringWavesUID` — registered in `XutheringWavesUID/__init__.py` with `force_prefix=["ww"]`, `allow_empty_prefix=False`
- All user-facing commands **require** the `ww` prefix (e.g., `ww帮助`, `ww查询`, `ww签到日历`)
- Version: `3.4.1` (in `XutheringWavesUID/version.py`)
- Every submodule `wutheringwaves_*` is a self-contained command group
- Build system: `pdm-backend` (pyproject.toml), Python >=3.10, <4.0
- **No tests** — no test framework, no test directory

## Trigger patterns (gsuid_core SV)

See any `wutheringwaves_*/__init__.py` for examples. Common patterns:
- `SV("name").on_fullmatch(...)` — exact match commands (`ww帮助`, `ww签到日历`)
- `SV("name").on_prefix(...)` — prefix match (`ww设置`)
- `SV("name", priority=N).on_regex(...)` — regex match

## AI / RAG integration

- Knowledge points registered via `ai_entity()` in `wutheringwaves_ai_rag/__init__.py` (characters, weapons, echoes, sonatas, tower seasons, guides, commands)
- AI tools registered via `@ai_tools` in `wutheringwaves_ai_rag/tools/`
- `to_ai` parameter on trigger decorators exposes command behavior to the AI
- Scheduled task at 04:01 daily re-registers season-sensitive KP tags (in `wutheringwaves_ai_rag/__init__.py:812`)

## Key commands (all prefixed with `ww`)

| Command | Module | Notes |
|---------|--------|-------|
| `ww帮助` | `wutheringwaves_help` | Full help image; `to_ai` annotated |
| `ww查询`/`ww卡片`/`wwkp` | `wutheringwaves_roleinfo` | Account overview card |
| `ww设置` | `wutheringwaves_config` | Prefix-based, many sub-options |
| `ww下载全部资源` | `wutheringwaves_resource` | Downloads game data; pm=1 (owner) |
| `ww签到日历` | `wutheringwaves_sign` | Sign-in calendar |
| `ww强制下载全部资源` | `wutheringwaves_resource` | Force re-download + may restart |

## Setup & dependencies

**Required** (auto-installed by core from pyproject.toml):
- `pypinyin`, `rapidfuzz`, `playwright`, `opencv-python`

**Optional but recommended** (manual install needed for playwright browser):
```bash
# Linux/Mac
source .venv/bin/activate && uv pip install playwright opencv-python fonttools pypinyin rapidfuzz && uv run playwright install chromium
# Windows
.venv\Scripts\activate; uv pip install playwright opencv-python fonttools pypinyin rapidfuzz; uv run playwright install chromium
```

## Startup flow

1. `__init__.py` registers plugin, copies build files (.pyd/.so), installs Bot.send hooks, starts activity buffer flush loop
2. `on_core_start` → `wutheringwaves_resource.startup()` → downloads/verifies resources, reloads modules, may restart if builds updated
3. `wutheringwaves_ai_rag.register_all()` runs at top level (inline) to register all knowledge points
4. Scheduled tasks: resource download (configurable time), RAG re-registration (04:01)

## Build files

- `.pyd` (Windows) / `.so` (Linux) build artifacts are extracted at startup via `copy_build_files()` in `utils/download_utils.py`
- Builds stored in `BUILD_ROOT` + `BUILD_TEMP`; cross-process locking via `.build_copy.lock` (file lock)
- Gitignored: `*.pyd`, `*.so`, `.build_copy.lock`, `waves_build/`

## Code linting

- `.pre-commit-config.yaml`: `ruff` (lint+fix) + `ruff-format`
- No CI workflow file found; `.ruff_cache/` is gitignored

## Non-obvious quirks

- Folder must be named `XutheringWavesUID` (not `WutheringWavesUID`) — checked at startup
- `requirements.txt` is empty / unused — actual deps are in `pyproject.toml` under `[project].dependencies`
- 3 lockfiles maintained (pdm.lock, poetry.lock, uv.lock) but only pdm-backend is used for build
- Bot.send and Bot.target_send are monkey-patched in `utils/bot_send_hook.py` for group binding and activity tracking
- Activity records are buffered in memory and flushed every 60s to avoid DB corruption under high concurrency
- Web panel editor at `/waves/panel-edit/` (Basic Auth, username `admin`) — enabled when `WavesPanelEditPassword` config is non-empty
- International (intl) UID support exists but some features explicitly reject intl UIDs with `intl_unavailable_msg`

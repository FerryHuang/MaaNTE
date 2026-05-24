# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MaaNTE is a game automation assistant for "异环" (NTE / Neverness to Everness), powered by [MaaFramework](https://github.com/MaaXYZ/MaaFramework). It automates tasks like fishing, coffee-making, rhythm games, piano, dodge mechanics, and reward claiming. The game must run at **1280x720 windowed** (any Windows display scaling is supported). Primary target is Windows; Linux/macOS for development only.

## Common Commands

```bash
# Build the distributable package (downloads MaaFramework, MXU, Python, assembles install dir)
python build.py --mode=mxu                    # MXU variant (current)
python build.py --mode=all                    # Both GUI variants
python build.py --skip-download               # Rebuild without re-downloading deps
python build.py --compress=false              # Skip zip/tar creation

# Format code
black agent/                                  # Python formatting
npx prettier --write "**/*.json"              # JSON/YAML formatting (uses .prettierrc)
npx prettier --write "**/*.{yml,yaml}"

# Validate pipeline/interface JSON
npx @nekosu/maa-tools validate

# Run agent for development (requires MaaFramework in deps/ and game running)
python agent/main.py <socket_id>              # Normally launched by MXU GUI, not directly
```

## Architecture

### Two-Layer Design

**Layer 1 — Pipeline JSON (declarative)**: `assets/resource/base/pipeline/`
- Defines automation flows as node graphs in JSON
- Each node: recognition method (TemplateMatch, OCR, Custom) + action (Click, Swipe, Custom) + next-node transitions
- Feature modules: Fish, MakeCoffee, WithdrawMoney, ClaimRewards, RealTime, Rhythm, AutoPiano, PinkPawHeist, Tetris, SoundDodge, AutoFScroll, FountainCheckin, Furniture, Movement
- Uses Pipeline v2 format: `recognition` and `action` are nested dictionaries

**Layer 2 — Python Agent (imperative)**: `agent/`
- Custom actions registered with `@AgentServer.custom_action("name")` decorator in `agent/custom/action/`
- Handles complex logic that can't be expressed declaratively (image algorithms, state machines, audio detection, MIDI playback)
- **Pipeline controls flow; Python handles per-node logic.** Never write business workflows in Python.

### Entry Point: `agent/main.py`

Startup flow: MXU GUI launches agent as child process with a socket ID → `main()` checks admin privilege (Windows), creates venv (Linux/dev), installs deps → `agent()` imports `AgentServer`, starts it with the socket ID, joins the server loop.

### Key Utilities: `agent/utils/`

| Module | Purpose |
|--------|---------|
| `logger.py` | Dual-backend logging (loguru preferred, stdlib fallback). Client-aware formatting: MXU=HTML, MFAA=plain, terminal=ANSI. |
| `pienv.py` | Parses MaaFramework PI environment variables into typed dataclasses |
| `i18n.py` | Internationalization — loads locale JSONs, provides `T()` translation function |
| `screen.py` | Screen coordinate mapping with baseline 1280x720 and scaling factors |
| `maafocus.py` | Sends user-visible messages to MXU GUI via MaaFramework pipeline focus protocol |
| `win32_process.py` | Windows process/window management (finding game window, getting client size, DPI scale detection) |

### Resource Structure: `assets/resource/`

- `base/pipeline/` — Pipeline JSON files organized by feature
- `base/image/` — Template images for recognition (per-feature subdirectories)
- `base/model/` — OCR/detection models (gitignored; copied from `MaaCommonAssets` submodule via `tools/ci/configure.py`)
- `base/sounds/` — Audio files and precomputed feature `.npy` files
- `locales/` — 5 languages × 2 domains (interface/ for GUI, agent/ for in-agent messages)
- `tasks/` — Task definition JSONs that declare UI options and pipeline overrides

### Git Submodules

- `assets/MaaCommonAssets` — OCR models from `MaaXYZ/MaaCommonAssets`
- `assets/MaaNTEModels` — Custom detection models from `1bananachicken/MaaNTEModels`

Clone with `git clone --recursive`. Initialize submodules separately with `git submodule update --init --recursive`.

## Key Conventions

- **All coordinates, ROIs, and template images are based on 1280×720.** The `screen.py` utility handles scaling.
- **Pipeline nodes use PascalCase naming** with task/module prefix (e.g., `FishNewEntrance`, `TetrisEntrance`).
- **CustomAction registration**: each action class uses `@AgentServer.custom_action("ActionName")` and is imported in `agent/custom/action/__init__.py`.
- **User-visible messages** use `maafocus.Print(context, msg)` or `maafocus.PrintT(context, key, *args)` (auto-translated). `logger.*` is for developer logs only — in MXU mode, console level is WARNING so INFO/DEBUG don't reach users.
- **Avoid hard delays** in Pipeline JSON. Use recognition nodes or `pre_wait_freezes`/`post_wait_freezes` instead of `pre_delay`/`post_delay`/`timeout`. When delays aren't needed, explicitly set `rate_limit`, `pre_delay`, `post_delay` to 0 (protocol defaults: rate_limit=1000ms, pre/post_delay=200ms).
- **Python**: formatted with Black. **JSON/YAML**: formatted with Prettier (4-space indent, `multilineArraysWrapThreshold: 1`). **Markdown**: markdownlint.

## Development Workflow

1. Fork and clone with `--recursive`
2. Download [MaaFramework release](https://github.com/MaaXYZ/MaaFramework/releases) to `deps/`
3. Python >= 3.11 required
4. Use VS Code with the `maa-support` extension for pipeline debugging
5. New features target the `dev` branch
6. No formal test suite — validation is manual testing against the game + CI build verification

## CI/CD

- **`install.yml`**: Main build/release workflow. Triggered on tag pushes (`v*.*.*`) and PRs touching `assets/**` or `**.py`.
- **`i18n-sync.yml`**: Daily OCR i18n sync from translation repo, creates PRs automatically.
- **`sync_schema_files.yml`**: Daily JSON schema sync from MaaFramework.
- Release artifacts are distributed via GitHub Releases and MirrorChyan.

## Skills

Project-specific Claude Code skills are in `.claude/skills/`:
- `python-action-guide` — CustomAction coding patterns and conventions
- `pipeline-guide` — Pipeline JSON writing guide with v2 format reference
- `maa-logging` — Logging conventions (logger vs maafocus separation)
- `maante-issue-log-analysis` — Issue log analysis workflow
- `dmp-analysis` — Windows crash dump analysis

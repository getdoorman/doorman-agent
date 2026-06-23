# Kanari Rename Design

**Date:** 2026-06-23
**Scope:** kanari-agent repo (formerly doorman-agent)

## Goal

Rename all `doorman*` references to `kanari*` across the Python package, CLI, env vars, config paths, and documentation. Clean rename — no backward compatibility layer.

## What Changes

### Package and entry points (`pyproject.toml`)
- `name`: `"doorman-agent"` → `"kanari-agent"`
- `packages`: `doorman_agent` → `kanari_agent`
- Entry points: `doorman` → `kanari`, `doorman-agent` → `kanari-agent`
- `repository` URL → `https://github.com/getkanari/kanari-agent`
- `[tool.bandit] targets`: `src/doorman_agent` → `src/kanari_agent`

### Module directory
- `src/doorman_agent/` → `src/kanari_agent/`
- All imports: `from doorman_agent.X import Y` → `from kanari_agent.X import Y`

### Public symbols
- `DoormanAgent` → `KanariAgent`
- `DoormanStampPlugin` → `KanariStampPlugin`
- `DOORMAN_TS_HEADER = "doorman_sent_ts"` → `KANARI_TS_HEADER = "kanari_sent_ts"`

### Internal functions and constants
- `load_doorman_config` → `load_kanari_config`
- `save_doorman_config` → `save_kanari_config`
- `DOORMAN_CONFIG_PATH` → `KANARI_CONFIG_PATH = Path.home() / ".kanari" / "config"`

### Environment variables
- `DOORMAN_API_KEY` → `KANARI_API_KEY`
- `DOORMAN_API_URL` → `KANARI_API_URL`
- `DOORMAN_LOCAL_MODE` → `KANARI_LOCAL_MODE`
- `DOORMAN_SANITIZE_TASK_SIGNATURES` → `KANARI_SANITIZE_TASK_SIGNATURES`

### CLI strings and User-Agent
- All `"doorman login"`, `"doorman agent"`, etc. → `"kanari login"`, `"kanari agent"`, etc.
- `f"doorman-agent {AGENT_VERSION}"` → `f"kanari-agent {AGENT_VERSION}"`
- `User-Agent` header: `doorman-agent/{version}` → `kanari-agent/{version}`

### Test files
- All `patch("doorman_agent.*")` → `patch("kanari_agent.*")`
- All `from doorman_agent.X import Y` → `from kanari_agent.X import Y`

### Documentation
- `README.md`, `CHANGELOG.md`, `CONTRIBUTING.md`, `CLAUDE.md`, `ASSISTANT.md`
- `config.example.yaml`: env var names

### Local filesystem (post-commit)
- `/Users/herchila/Projects/doorman-agent` → `/Users/herchila/Projects/kanari-agent`

## Constraints
- No backward compatibility (pre-launch, no real users)
- Python ≥ 3.9, all type hints preserved
- All 276 tests must pass after rename
- `pre-commit run --all-files` must pass
- Directory rename happens last (after all commits) to avoid breaking paths

## Out of Scope
- doorman-backend repo (internal, low priority)
- doorman-landing (separate repo)
- LemonSqueezy billing branch (merge separately after rename)

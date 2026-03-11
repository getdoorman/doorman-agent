# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Doorman Agent is a lightweight, privacy-first monitoring agent for Celery/Redis queues. It collects metrics from Celery workers and Redis queues, then either:
- **API mode**: Sends metrics to doorman.com for analysis and alerting
- **Local mode**: Logs metrics as structured JSON to stdout (no API calls)

**Python version**: 3.9+ (supports up to 3.13)
**Package manager**: Poetry
**Status**: Alpha (v0.1.0a3)
**License**: Apache 2.0

---

## Repository Structure

```
doorman-agent/
├── src/doorman_agent/          # Main source package
│   ├── __init__.py             # Version detection
│   ├── agent.py                # DoormanAgent: main orchestrator + monitoring loop
│   ├── collector.py            # MetricsCollector: Redis + Celery metrics
│   ├── api_client.py           # APIClient: HTTP communication + privacy sanitization
│   ├── config.py               # load_config(): YAML + env var loading, AGENT_VERSION
│   ├── models.py               # Pydantic data models (Config, SystemMetrics, etc.)
│   ├── logger.py               # StructuredLogger: structured JSON output
│   ├── audit.py                # run_audit(): one-time health report with rich formatting
│   ├── simulator.py            # SimulationEnvironment: local demo without real infra
│   └── cli.py                  # main(): CLI argument parsing and mode dispatch
├── tests/
│   ├── test_config.py          # Config loading, env vars, defaults (~30 tests)
│   ├── test_collector.py       # Metrics collection, auto-discovery (~35 tests)
│   ├── test_api_client.py      # Privacy sanitization, HTTP, API calls (~35 tests)
│   ├── test_agent.py           # Orchestration, modes, failure tracking (~20 tests)
│   ├── test_audit.py           # Audit analysis, trend detection, formatting (~60 tests)
│   ├── test_cli.py             # CLI mode dispatch (~8 tests)
│   └── json_response_mock.json # Test fixture for API responses
├── pyproject.toml              # Poetry config, dependencies, tool settings
├── .pre-commit-config.yaml     # Ruff + mypy pre-commit hooks
├── config.example.yaml         # Example configuration file
├── Makefile                    # Convenience targets
├── Dockerfile                  # Docker image
├── docker-compose.yml          # Docker Compose for local dev
├── celery-doorman.service      # Systemd service file
├── README.md                   # User-facing documentation
└── CONTRIBUTING.md             # Developer guidelines
```

---

## Development Commands

### Setup
```bash
# Install with dev dependencies
poetry install --with dev

# Install pre-commit hooks (required before first commit)
pre-commit install
```

### Testing
```bash
# Run all tests
poetry run pytest

# Run tests with coverage report
poetry run pytest --cov=doorman_agent --cov-report=html

# Run a single test file
poetry run pytest tests/test_config.py

# Run a specific test
poetry run pytest tests/test_config.py::TestLoadConfigDefaults::test_default_api_url

# Run tests matching a keyword
poetry run pytest -k "test_sanitize"
```

### Linting and Type Checking
```bash
# Run all pre-commit hooks (ruff + mypy)
pre-commit run --all-files

# Run ruff linter only
ruff check .

# Run ruff with auto-fix
ruff check --fix .

# Run ruff formatter
ruff format .

# Run mypy type checker
mypy src/doorman_agent
```

### Running the Agent
```bash
# Local mode — no API calls, logs metrics as JSON to stdout
poetry run doorman-agent --config config.yaml --local

# Single check (useful for testing/debugging)
poetry run doorman-agent --config config.yaml --once

# Audit mode — one-time health report with exit code
poetry run doorman-agent --audit --config config.yaml

# Deep audit — includes Redis + Celery configuration analysis
poetry run doorman-agent --audit --deep --config config.yaml

# Simulation mode — spins up local Redis + Celery workers (no real infra needed)
poetry run doorman-agent --simulate --workers 2 --enqueue 50

# API mode (production)
export DOORMAN_API_KEY=your-api-key
poetry run doorman-agent --config config.yaml
```

---

## Architecture

### Core Components

#### 1. `DoormanAgent` (`agent.py`)
Main orchestrator that manages the monitoring loop and coordinates all components.

- Runs two modes: **API mode** (sends to doorman.com) or **local mode** (logs JSON only)
- Handles graceful shutdown via SIGTERM/SIGINT signal handling
- Tracks consecutive API failures (max 10); continues collecting even if API is down
- On each cycle: collect → log status → log locally or send to API

**Key methods:**
- `check_once()` — executes one complete monitoring cycle
- `run()` — main event loop; respects `check_interval_seconds`
- `_log_metrics_locally(metrics)` — dumps `SystemMetrics` as structured JSON

#### 2. `MetricsCollector` (`collector.py`)
Collects metrics from Redis and Celery infrastructure.

- **Queue depth**: Redis `LLEN` per queue name
- **Task age**: Extracts timestamps from Celery message headers via `LINDEX -1`
- **Worker stats**: Via Celery `control.inspect()` (active, reserved, pool stats)
- **Stuck task detection**: Tasks running longer than `max_task_runtime_seconds`
- **Saturation**: `(active_tasks / total_concurrency) * 100`
- **Auto-discovery**: If `monitored_queues` is empty, calls `inspector.active_queues()`

**Key methods:**
- `connect()` — establishes Redis and Celery connections
- `discover_queues()` — queries workers for their subscribed queues
- `collect()` — aggregates everything into a `SystemMetrics` object

#### 3. `APIClient` (`api_client.py`)
Handles communication with the doorman.com API, applying privacy sanitization before any data leaves the host.

**Privacy transformations (applied in `build_payload()`):**
- Worker IDs: `celery@prod-worker-1` → `w-a1b2c3d4` (SHA-256, first 8 hex chars)
- Task IDs: `uuid-12345` → `t-a1b2c3d4e5f6` (SHA-256, first 12 hex chars)
- Task signatures: emails, UUIDs, and numeric IDs replaced with `[email]`, `[uuid]`, `[id]`
- Queue names: emails and UUIDs redacted
- Task arguments: **never accessed or collected**

Uses **stdlib `urllib`** only — no external HTTP library dependency.

**Key methods:**
- `validate_api_key()` — `GET /api/v1/auth/validate` on startup
- `send_metrics(metrics)` — `POST /api/v1/metrics` with sanitized payload
- `send_heartbeat()` — `POST /api/v1/heartbeat`
- `build_payload(metrics)` — transforms `SystemMetrics` → sanitized dict

#### 4. Configuration System (`config.py`, `models.py`)
Loads and validates configuration. Load order (later overrides earlier):
1. Hardcoded defaults (in Pydantic models)
2. YAML config file
3. Environment variables

**Key environment variables:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `DOORMAN_API_KEY` | — | API key for doorman.com |
| `DOORMAN_API_URL` | `https://api.doorman.com` | Custom API endpoint |
| `REDIS_URL` | `redis://localhost:6379/0` | Redis connection |
| `CELERY_BROKER_URL` | `redis://localhost:6379/0` | Celery broker |
| `DOORMAN_LOCAL_MODE` | `false` | Set `true` to disable API calls |
| `CHECK_INTERVAL` | `30` | Seconds between monitoring cycles |
| `DOORMAN_SANITIZE_TASK_SIGNATURES` | `true` | Set `false` to skip sanitization |

#### 5. `StructuredLogger` (`logger.py`)
Emits structured JSON to stdout for log aggregation (Datadog, CloudWatch, etc.).

```json
{
  "timestamp": "2026-01-01T00:00:00+00:00",
  "level": "INFO",
  "message": "Health check completed",
  "logger": "doorman-agent",
  "pending_tasks": 42,
  "saturation_pct": 65.5
}
```

Supports extra fields: `logger.info("msg", key1=val1, key2=val2)`

#### 6. Audit Report Generator (`audit.py`)
One-time health check with rich-formatted output and meaningful exit codes.

**Exit codes:**
- `0` — HEALTHY: All systems operational
- `1` — WARNING: Issues detected (congested queues, high saturation)
- `2` — CRITICAL: Severe issues (dead workers, stuck tasks, ghost workers)

**Detection rules:**
- **Ghost workers**: backlog > 0 AND saturation < 50% → CRITICAL
- **High saturation**: > 90% → WARNING
- **Worker at capacity**: `active_tasks >= concurrency` → WARNING
- **Stuck tasks**: runtime > `max_task_runtime_seconds` → CRITICAL

**Deep audit** (`--deep`) also checks:
- Redis: `maxmemory`, eviction policy, persistence, connection pool usage
- Celery: `task_acks_late`, `task_reject_on_worker_lost`, `prefetch_multiplier`

#### 7. Simulation Environment (`simulator.py`)
Self-contained demo that starts a local Redis server (port 6399) and spawns Celery workers — no real infrastructure required.

```bash
# Healthy: 2 workers processing normally
doorman-agent --simulate --workers 2

# Backlog: no workers, 50 queued tasks
doorman-agent --simulate --workers 0 --enqueue 50

# Overloaded: 1 worker, 100 queued tasks
doorman-agent --simulate --workers 1 --enqueue 100
```

### Data Flow

```
MetricsCollector.collect()
  → connects to Redis + Celery
  → discovers queues if monitored_queues is empty
  → gets queue depths (Redis LLEN)
  → gets oldest task age (Redis LINDEX -1, reads headers)
  → inspects workers (active, reserved, stats via Celery inspector)
  → detects stuck tasks (time_start > max_task_runtime_seconds)
  → returns SystemMetrics

DoormanAgent.check_once()
  → calls collector.collect()
  → logs basic status line
  → if local_mode: logs full metrics JSON to stdout
  → if API mode:
      → APIClient.build_payload() applies privacy transformations
      → POST to /api/v1/metrics
      → tracks consecutive failures
```

---

## Data Models (`models.py`)

All models use **Pydantic v2**.

```python
class QueueMetrics:
    name: str
    depth: int = 0
    oldest_task_age_seconds: float | None = None

class WorkerMetrics:
    name: str
    active_tasks: int = 0
    concurrency: int = 0
    last_heartbeat: str | None = None
    is_alive: bool = True

class SystemMetrics:
    timestamp: str                          # ISO 8601
    total_pending_tasks: int = 0
    total_active_tasks: int = 0
    total_workers: int = 0
    alive_workers: int = 0
    total_concurrency: int = 0
    saturation_pct: float = 0.0             # (active / concurrency) * 100
    max_latency_sec: float | None = None
    queues: list[QueueMetrics] = []
    workers: list[WorkerMetrics] = []
    stuck_tasks: list[dict[str, Any]] = []
    redis_connected: bool = False
    celery_connected: bool = False

class AlertThresholds:
    max_queue_size: int = 1000
    max_wait_time_seconds: int = 60
    max_task_runtime_seconds: int = 1800    # 30 minutes
    worker_heartbeat_timeout_seconds: int = 120
    critical_queues: list[str] = []

class PrivacyConfig:
    sanitize_task_signatures: bool = True
```

---

## Configuration Reference (`config.example.yaml`)

```yaml
api_key: your-key-here          # or set DOORMAN_API_KEY env var
redis_url: redis://localhost:6379/0
celery_broker_url: redis://localhost:6379/0
celery_app_name: tasks
check_interval_seconds: 30
monitored_queues: []            # empty = auto-discover from workers

thresholds:
  max_queue_size: 1000
  max_wait_time_seconds: 60
  max_task_runtime_seconds: 1800

privacy:
  sanitize_task_signatures: true
```

---

## Testing Guidelines

- **Framework**: `pytest` with `unittest.mock` for all Redis/Celery interactions
- **No real connections**: All tests use mocks; no Redis/Celery server required
- **Environment variables**: Use `monkeypatch.setenv()` / `monkeypatch.delenv()`
- **Parametrized tests**: Used extensively in `test_config.py` and `test_api_client.py`
- **Test class naming**: `TestClassName` with `test_method_name` methods

When adding new functionality:
1. Add unit tests in the corresponding `tests/test_*.py` file
2. Mock external calls (Redis, Celery, HTTP) — never make real network calls
3. Test both success and failure paths
4. For privacy-sensitive logic, add tests in `test_api_client.py`

---

## Code Style and Conventions

- **Linter + formatter**: Ruff (replaces flake8, black, isort)
- **Type checker**: mypy (strict mode — type hints required on all definitions)
- **Line length**: 100 characters
- **Quotes**: Double quotes (`"`)
- **Import sorting**: Ruff handles this automatically
- Pre-commit hooks enforce all rules; run `pre-commit run --all-files` before committing

**Do not skip pre-commit hooks.** Fix all ruff and mypy errors before committing.

---

## Key Design Principles

1. **Privacy by default**: All worker names, task IDs, and queue names are hashed/sanitized before leaving the host. Task arguments are never accessed.

2. **Graceful degradation**: The agent continues collecting metrics even if the API is unreachable. Consecutive failures are tracked and logged.

3. **Auto-discovery**: If `monitored_queues` is empty in config, queues are discovered from active Celery workers — no manual list required.

4. **No heavy dependencies**: Uses only stdlib `urllib` for HTTP. Synchronous design (no async). No ORM or database drivers.

5. **Structured logging**: All output is structured JSON for compatibility with log aggregation systems.

6. **Two operating modes**: Local mode for development/debugging; API mode for production monitoring.

---

## API Endpoints (doorman.com)

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/api/v1/auth/validate` | Validate API key on startup |
| `POST` | `/api/v1/metrics` | Send collected metrics |
| `POST` | `/api/v1/heartbeat` | Send periodic heartbeat |

**Request headers:**
```
Authorization: Bearer {api_key}
Content-Type: application/json
User-Agent: doorman-agent/{version}
X-Agent-Session: {session_id}
```

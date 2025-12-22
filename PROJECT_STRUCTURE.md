# PentestGPT Project Structure

This guide provides a comprehensive overview of the PentestGPT project structure, including directory layout, file organization, and dependency management.

## Table of Contents

- [Directory Layout](#directory-layout)
- [Core Modules](#core-modules)
- [Interface Components](#interface-components)
- [Configuration Files](#configuration-files)
- [Dependencies](#dependencies)
- [Build System](#build-system)

---

## Directory Layout

```
PentestGPT/
├── pentestgpt/              # Main Python package
│   ├── core/                # Core agent logic and orchestration
│   │   ├── agent.py         # PentestAgent - main AI agent wrapper
│   │   ├── backend.py       # Framework-agnostic backend interface
│   │   ├── controller.py    # AgentController - lifecycle management
│   │   ├── events.py        # EventBus - pub/sub event system
│   │   ├── session.py       # Session persistence and state
│   │   ├── config.py        # Configuration management (Pydantic)
│   │   ├── tracer.py        # Activity tracking system
│   │   └── langfuse.py      # Telemetry integration
│   │
│   ├── interface/           # User interface layer
│   │   ├── main.py          # CLI entry point and argument parsing
│   │   ├── tui.py           # Textual-based TUI application
│   │   └── components/      # TUI components
│   │       ├── activity_feed.py   # Real-time activity display
│   │       ├── splash.py          # Splash screen
│   │       └── renderers.py       # Tool-specific output renderers
│   │
│   ├── prompts/             # System prompts and templates
│   │   └── pentesting.py    # CTF_SYSTEM_PROMPT - main system prompt
│   │
│   ├── tools/               # Extensible tool framework
│   │   ├── base.py          # BaseTool abstract class
│   │   └── registry.py      # ToolRegistry singleton
│   │
│   ├── benchmark/           # Benchmark management system
│   │   ├── cli.py           # CLI for benchmark operations
│   │   ├── registry.py      # BenchmarkRegistry - discovery
│   │   ├── docker.py        # Docker container management
│   │   └── config.py        # Benchmark configuration
│   │
│   └── __init__.py          # Package initialization
│
├── benchmark/               # Benchmark suites (git submodules)
│   └── xbow-validation-benchmarks/   # 104 vulnerability challenges
│       ├── XBEN-001-24/     # Individual benchmark
│       │   ├── benchmark.json       # Metadata
│       │   ├── docker-compose.yml   # Container definition
│       │   └── src/                 # Challenge source code
│       └── ...
│
├── tests/                   # Test suite
│   ├── unit/                # Unit tests (fast, isolated)
│   │   ├── test_events.py
│   │   ├── test_session.py
│   │   ├── test_backend_interface.py
│   │   └── test_benchmark_registry.py
│   │
│   ├── integration/         # Integration tests (with mocks)
│   │   ├── test_controller.py
│   │   └── test_benchmark_cli.py
│   │
│   ├── docker/              # Docker-specific tests
│   │   └── test_docker_build.py
│   │
│   └── conftest.py          # Pytest configuration and fixtures
│
├── workspace/               # Runtime workspace (Docker mount)
│   └── pentestgpt-debug.log # Agent debug logs
│
├── scripts/                 # Helper scripts
│   ├── config.sh            # Authentication configuration script
│   └── ccr-config-template.json  # Claude Code Routing template
│
├── legacy/                  # Archived v0.15 (multi-LLM version)
│   └── pentestgpt/          # Legacy implementation
│
├── docs/                    # Documentation (this directory)
│   ├── PROJECT_STRUCTURE.md
│   ├── FEATURES.md
│   ├── ARCHITECTURE.md
│   └── CODE_GUIDE.md
│
├── Dockerfile               # Ubuntu 24.04 container definition
├── docker-compose.yml       # Container orchestration
├── Makefile                 # Development commands
├── pyproject.toml           # Project metadata and dependencies
├── uv.lock                  # Lock file (uv package manager)
├── CLAUDE.md                # Claude Code instructions
├── README.md                # User-facing documentation
└── LICENSE.md               # MIT License
```

---

## Core Modules

### `pentestgpt/core/` - Agent Core Layer

This is the heart of PentestGPT's autonomous agent system.

| File | Purpose | Key Classes/Functions |
|------|---------|---------------------|
| **agent.py** | Main AI agent wrapper with Claude Code SDK integration | `PentestAgent`, `execute()`, flag detection patterns |
| **backend.py** | Framework-agnostic backend protocol | `AgentBackend` (ABC), `ClaudeCodeBackend`, `AgentMessage` |
| **controller.py** | Agent lifecycle orchestration (5-state model) | `AgentController`, `AgentState` enum, pause/resume logic |
| **events.py** | Pub/sub event bus for decoupled communication | `EventBus` singleton, `Event`, `EventType` enum |
| **session.py** | Session persistence and state tracking | `SessionStore`, `SessionInfo`, `SessionStatus` |
| **config.py** | Pydantic-based configuration management | `PentestGPTConfig`, `.env` file support |
| **tracer.py** | Activity tracking for debugging/monitoring | `Tracer` singleton, `get_global_tracer()` |
| **langfuse.py** | Telemetry integration (opt-in) | `init_langfuse()`, `shutdown_langfuse()` |

**Key Relationships:**
- `AgentController` orchestrates the entire agent lifecycle
- `PentestAgent` wraps the Claude Code SDK and emits events
- `EventBus` connects the agent to the TUI without tight coupling
- `SessionStore` persists state to `~/.pentestgpt/sessions/`

---

### `pentestgpt/interface/` - User Interface Layer

Provides both CLI and TUI interfaces.

| File | Purpose | Key Components |
|------|---------|---------------|
| **main.py** | CLI entry point, argument parsing, mode selection | `main()`, argument parser, initialization |
| **tui.py** | Textual framework TUI application | `PentestGPTApp`, `HelpScreen`, `QuitScreen` |
| **components/activity_feed.py** | Real-time activity display widget | `ActivityFeed`, message rendering |
| **components/splash.py** | Animated splash screen | `SplashScreen` |
| **components/renderers.py** | Tool-specific output formatters | Tool renderers for Bash, Read, etc. |

**Keyboard Shortcuts:**
- `F1` - Help
- `Ctrl+P` - Pause/Resume
- `Ctrl+Q` - Quit
- `↑/↓` - Scroll
- `Enter` - Send instruction (when paused)

---

### `pentestgpt/prompts/` - System Prompts

Contains carefully crafted prompts for penetration testing.

| File | Purpose | Key Content |
|------|---------|-------------|
| **pentesting.py** | CTF challenge solving system prompt | `CTF_SYSTEM_PROMPT` - methodology, persistence directives, fallback strategies |

**Prompt Design Principles:**
- Never give up until flags are captured
- Systematic methodology (recon → vuln discovery → exploitation → flag extraction)
- Category-specific techniques (web, pwn, crypto, forensics, privilege escalation)
- Fallback strategies when stuck

---

### `pentestgpt/tools/` - Tool Framework

Extensible plugin system for tools.

| File | Purpose | Key Classes |
|------|---------|-------------|
| **base.py** | Abstract base class for tools | `BaseTool`, `TerminalTool` |
| **registry.py** | Global tool registry | `ToolRegistry` singleton, `get_registry()` |

**Extension Pattern:**
```python
from pentestgpt.tools.base import BaseTool

class MyTool(BaseTool):
    async def execute(self, **kwargs):
        # Implementation
        return {"success": True, "result": ...}

# Register
from pentestgpt.tools.registry import get_registry
get_registry().register(MyTool())
```

---

### `pentestgpt/benchmark/` - Benchmark System

Manages vulnerability challenge containers.

| File | Purpose | Key Components |
|------|---------|---------------|
| **cli.py** | Command-line interface | `main()`, `list`, `start`, `stop`, `status` commands |
| **registry.py** | Benchmark discovery | `BenchmarkRegistry`, discovers `benchmark.json` files |
| **docker.py** | Container lifecycle | Start/stop Docker Compose services |
| **config.py** | Configuration | Paths, ports, Docker settings |

**CLI Commands:**
```bash
pentestgpt-benchmark list                     # List all benchmarks
pentestgpt-benchmark list --tags sqli         # Filter by tag
pentestgpt-benchmark start XBEN-001-24        # Start benchmark
pentestgpt-benchmark status                   # Check running
pentestgpt-benchmark stop XBEN-001-24         # Stop benchmark
```

---

## Configuration Files

### `pyproject.toml`

**Purpose:** Project metadata, dependencies, build configuration, tool settings

**Sections:**
- `[project]` - Name, version, authors, dependencies
- `[project.scripts]` - CLI entry points (`pentestgpt`, `pentestgpt-benchmark`)
- `[tool.uv]` - Development dependencies
- `[tool.ruff]` - Linter/formatter configuration
- `[tool.mypy]` - Type checker settings
- `[tool.pytest.ini_options]` - Test configuration, markers

**Key Dependencies:**
```toml
dependencies = [
    "textual>=0.47.0",           # TUI framework
    "rich>=13.7.0",              # Terminal formatting
    "pydantic>=2.5.0",           # Data validation
    "pydantic-settings>=2.1.0",  # Settings management
    "claude-agent-sdk>=0.1.0",   # Claude Code integration
    "python-dotenv>=1.0.0",      # .env file support
    "anthropic>=0.75.0",         # Anthropic API client
    "langfuse>=2.0.0",           # Telemetry
]
```

---

### `Makefile`

**Purpose:** Development workflow automation

**Primary Targets:**

```makefile
# Docker workflow
make install        # Build image, install dependencies
make config         # Configure authentication
make connect        # Main entry point - attach to container
make stop           # Stop container
make clean-docker   # Remove everything

# Development
make test           # Run all tests
make test-cov       # Tests with coverage
make test-unit      # Unit tests only
make lint           # Ruff linter
make format         # Ruff formatter
make typecheck      # MyPy type checking
make ci             # Full CI simulation

# Testing by file
make test-session   # Session tests
make test-controller # Controller tests
```

---

### `Dockerfile`

**Purpose:** Defines the PentestGPT container environment

**Base Image:** `ubuntu:24.04`

**Key Components:**
- Non-root user: `pentester` (UID 1000, sudo access)
- Working directory: `/workspace`
- Pre-installed tools: nmap, netcat, curl, wget, git, ripgrep, tmux, python3, uv
- Volume mounts: `./workspace:/workspace`, `claude-config` for LLM config

---

### `docker-compose.yml`

**Purpose:** Container orchestration and configuration

**Key Configuration:**
- Service name: `pentestgpt`
- Container name: `pentestgpt`
- Volumes: workspace mount, config persistence
- Network: `bridge` mode with `host.docker.internal` for host access
- Environment: Authentication variables, working directory
- Interactive: `stdin_open: true`, `tty: true`

---

## Dependencies

### Production Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| **textual** | >=0.47.0 | TUI framework (reactive, CSS-based) |
| **rich** | >=13.7.0 | Terminal formatting and styling |
| **pydantic** | >=2.5.0 | Data validation and settings |
| **pydantic-settings** | >=2.1.0 | Environment-based configuration |
| **claude-agent-sdk** | >=0.1.0 | Claude Code CLI integration |
| **python-dotenv** | >=1.0.0 | `.env` file support |
| **anthropic** | >=0.75.0 | Anthropic API client (for direct API usage) |
| **langfuse** | >=2.0.0 | Telemetry and observability |

### Development Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| **ruff** | >=0.1.0 | Fast Python linter and formatter |
| **mypy** | >=1.7.0 | Static type checker |
| **pytest** | >=7.4.0 | Testing framework |
| **pytest-asyncio** | >=0.21.0 | Async test support |
| **pytest-cov** | >=4.1.0 | Coverage reporting |

---

## Build System

### Package Manager: `uv`

**Why uv?**
- Fast, modern Python package manager
- Resolves dependencies quickly
- Compatible with `pip` and `pyproject.toml`
- Lock file for reproducible builds

**Common Commands:**
```bash
uv sync              # Install all dependencies
uv add <package>     # Add new dependency
uv remove <package>  # Remove dependency
uv run <command>     # Run command in venv
uv build             # Build wheel/sdist
```

---

### Entry Points

Defined in `pyproject.toml` under `[project.scripts]`:

```toml
[project.scripts]
pentestgpt = "pentestgpt.interface.main:main"
pentestgpt-benchmark = "pentestgpt.benchmark.cli:main"
```

**Result:**
- `pentestgpt` command → `pentestgpt.interface.main:main()`
- `pentestgpt-benchmark` command → `pentestgpt.benchmark.cli:main()`

---

## File Locations

### Runtime Files

| Path | Purpose |
|------|---------|
| `~/.pentestgpt/sessions/` | Saved session JSON files |
| `~/.pentestgpt/user_id` | Persistent telemetry user ID |
| `/workspace/pentestgpt-debug.log` | Agent debug log (Docker) |
| `/tmp/pentestgpt-debug.log` | Fallback debug log location |

### Configuration Files

| Path | Purpose |
|------|---------|
| `.env` | Environment variables (if used) |
| `.env.auth` | Authentication configuration (Docker) |
| `claude-config/` | Claude Code CLI config (Docker volume) |

---

## Test Organization

### Test Markers

Defined in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
markers = [
    "unit: Unit tests (fast, no external dependencies)",
    "integration: Integration tests (may use mocks)",
    "docker: Docker tests (requires Docker daemon)",
    "slow: Slow tests (skip with -m 'not slow')",
]
```

**Usage:**
```bash
pytest -m unit              # Run only unit tests
pytest -m "not slow"        # Skip slow tests
pytest -m docker            # Docker tests only
```

### Test Structure

- `tests/unit/` - Fast, isolated tests (no I/O)
- `tests/integration/` - Tests with mocked dependencies
- `tests/docker/` - Tests requiring Docker daemon
- `tests/conftest.py` - Shared fixtures and configuration

---

## Next Steps

- **[FEATURES.md](./FEATURES.md)** - Detailed feature documentation
- **[ARCHITECTURE.md](./ARCHITECTURE.md)** - Design patterns and data flow
- **[CODE_GUIDE.md](./CODE_GUIDE.md)** - Code walkthroughs and explanations

# PentestGPT Architecture Guide

This guide provides an in-depth look at PentestGPT's architecture, including design patterns, data flow, component interactions, and architectural decisions.

## Table of Contents

- [Architectural Overview](#architectural-overview)
- [Design Patterns](#design-patterns)
- [Core Components](#core-components)
- [Data Flow](#data-flow)
- [Event-Driven Architecture](#event-driven-architecture)
- [Session Lifecycle](#session-lifecycle)
- [Backend Abstraction](#backend-abstraction)
- [Extension Points](#extension-points)
- [Performance Considerations](#performance-considerations)

---

## Architectural Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Interface                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  TUI (Textual)                                            │  │
│  │  - ActivityFeed    - SplashScreen    - HelpScreen        │  │
│  │  - StatusBar       - Input           - QuitScreen        │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↕ (EventBus - Pub/Sub)
┌─────────────────────────────────────────────────────────────────┐
│                       Controller Layer                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  AgentController (Orchestrator)                           │  │
│  │  - 5-State Lifecycle (IDLE→RUNNING→PAUSED→COMPLETED)     │  │
│  │  - Pause/Resume Control                                   │  │
│  │  - Session Management                                     │  │
│  │  - Flag Detection                                         │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────────┐
│                         Agent Layer                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  PentestAgent                                             │  │
│  │  - Task Execution                                         │  │
│  │  - Activity Tracking (Tracer)                            │  │
│  │  - Flag Detection                                         │  │
│  │  - Walkthrough Generation                                │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────────┐
│                       Backend Layer                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  AgentBackend (Abstract Interface)                        │  │
│  │  ├─ ClaudeCodeBackend (Claude SDK)                       │  │
│  │  ├─ (Future) OpenAIBackend                               │  │
│  │  └─ (Future) LocalLLMBackend                             │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ↕
┌─────────────────────────────────────────────────────────────────┐
│                      External Services                          │
│  - Claude Code CLI                                              │
│  - LLM APIs (Anthropic, OpenRouter)                            │
│  - Local LLM Servers (LM Studio, Ollama)                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      Persistence Layer                          │
│  - SessionStore (~/.pentestgpt/sessions/)                      │
│  - Configuration (~/.config/, .env)                             │
│  - Debug Logs (/workspace/pentestgpt-debug.log)                │
│  - Telemetry (Langfuse)                                        │
└─────────────────────────────────────────────────────────────────┘
```

### Layered Architecture

PentestGPT follows a **layered architecture** with clear separation of concerns:

1. **Presentation Layer** (`pentestgpt/interface/`)
   - TUI components (Textual framework)
   - CLI entry point
   - User input handling

2. **Application Layer** (`pentestgpt/core/controller.py`)
   - Business logic orchestration
   - State management
   - Session lifecycle

3. **Domain Layer** (`pentestgpt/core/agent.py`)
   - Core agent logic
   - Task execution
   - Flag detection

4. **Infrastructure Layer** (`pentestgpt/core/backend.py`)
   - External service integration
   - LLM communication
   - Tool execution

5. **Persistence Layer** (`pentestgpt/core/session.py`)
   - Session storage
   - Configuration management
   - Logging

---

## Design Patterns

### 1. Singleton Pattern

**Used For:** Global state management with thread-safe access

**Implementations:**

```python
# EventBus Singleton
class EventBus:
    _instance = None
    _lock = threading.Lock()

    @classmethod
    def get(cls) -> "EventBus":
        with cls._lock:
            if cls._instance is None:
                cls._instance = cls()
            return cls._instance
```

**Singletons in PentestGPT:**
- `EventBus.get()` - Global event bus
- `get_global_tracer()` - Global activity tracer
- `get_registry()` - Global tool registry

**Rationale:**
- Single source of truth for events, activities, tools
- Thread-safe access from multiple components
- Easy testing (can reset singleton)

---

### 2. Observer Pattern (Pub/Sub)

**Used For:** Decoupled communication between TUI and agent

**Implementation:** EventBus

```python
# Subscribe to events
bus = EventBus.get()
bus.subscribe(EventType.MESSAGE, self._on_message)
bus.subscribe(EventType.STATE_CHANGED, self._on_state_change)

# Emit events
bus.emit_message("Starting enumeration", "info")
bus.emit_state("running", "Target: 10.10.11.234")
```

**Benefits:**
- TUI and agent are completely decoupled
- No direct dependencies between components
- Easy to add new subscribers (logging, telemetry, etc.)
- Thread-safe event propagation

**Event Types:**
- `STATE_CHANGED` - Agent state transitions
- `MESSAGE` - Text output from agent
- `TOOL` - Tool execution events
- `FLAG_FOUND` - Flag detection events
- `USER_COMMAND` - User control commands (pause/resume/stop)
- `USER_INPUT` - User instruction input

---

### 3. Abstract Factory Pattern

**Used For:** Framework-agnostic backend creation

**Implementation:**

```python
class AgentBackend(ABC):
    @abstractmethod
    async def connect(self) -> None: ...

    @abstractmethod
    async def query(self, prompt: str) -> None: ...

    @abstractmethod
    def receive_messages(self) -> AsyncIterator[AgentMessage]: ...

# Concrete implementation
class ClaudeCodeBackend(AgentBackend):
    async def connect(self):
        # Claude SDK specific logic
        ...
```

**Benefits:**
- Easy to swap LLM backends
- Future support for OpenAI, Gemini, local LLMs
- Testing with mock backends
- Consistent interface across backends

---

### 4. Strategy Pattern

**Used For:** Different execution modes (TUI vs Raw vs Non-Interactive)

**Implementation:**

```python
# main.py - mode selection
if args.raw:
    # Raw streaming mode
    await run_raw_mode(controller)
elif args.non_interactive:
    # Non-interactive mode
    await run_non_interactive(controller)
else:
    # TUI mode (default)
    app = PentestGPTApp(...)
    await app.run_async()
```

**Benefits:**
- Same agent logic, different presentation
- Easy to add new modes
- Separation of execution from presentation

---

### 5. State Pattern

**Used For:** Agent lifecycle management

**States:**
```python
class AgentState(Enum):
    IDLE = "idle"
    RUNNING = "running"
    PAUSED = "paused"
    COMPLETED = "completed"
    ERROR = "error"
```

**State Transitions:**
```python
IDLE → RUNNING (start)
RUNNING → PAUSED (user pause or agent request)
PAUSED → RUNNING (user resume)
RUNNING → COMPLETED (flags found)
RUNNING → ERROR (fatal error)
```

**State-Specific Behavior:**
- `IDLE` - Waiting to start
- `RUNNING` - Active task execution, cannot accept new instructions
- `PAUSED` - Accepts user input, maintains context
- `COMPLETED` - Final state, session saved
- `ERROR` - Error state, session saved with error details

---

### 6. Command Pattern

**Used For:** Tool execution abstraction

**Implementation:**

```python
class BaseTool(ABC):
    @abstractmethod
    async def execute(self, *args, **kwargs) -> dict[str, Any]:
        """Execute the tool with given arguments."""
        pass

# Usage
tool = registry.get("terminal_execute")
result = await tool.execute(command="nmap -sV 10.10.11.234")
```

**Benefits:**
- Uniform tool interface
- Easy to add new tools
- Trackable execution (via EventBus)
- Async execution support

---

## Core Components

### 1. AgentController

**Purpose:** Central orchestrator for the entire agent lifecycle

**Responsibilities:**
- State management (5-state model)
- Session creation and persistence
- Pause/resume control
- Flag detection coordination
- Error handling and recovery

**Key Methods:**

```python
async def start(self) -> None:
    """Start agent execution"""

async def pause(self) -> None:
    """Pause agent at message boundary"""

async def resume(self, instruction: str | None) -> None:
    """Resume with optional new instruction"""

async def stop(self) -> None:
    """Stop agent and save session"""
```

**State Machine:**
```python
def _transition_state(self, new_state: AgentState, details: str = ""):
    if self._is_valid_transition(self.state, new_state):
        self.state = new_state
        EventBus.get().emit_state(new_state.value, details)
        self._update_session_status()
```

---

### 2. PentestAgent

**Purpose:** Wraps LLM agent with penetration testing capabilities

**Responsibilities:**
- Task execution via Claude Code SDK
- Flag detection in outputs
- Walkthrough generation
- Activity tracking
- Debug logging

**Execution Flow:**

```python
async def execute(self, task: str) -> dict[str, Any]:
    # 1. Initialize Claude SDK client
    options = ClaudeAgentOptions(...)
    self.client = ClaudeSDKClient(options)

    # 2. Create system prompt
    system_prompt = get_ctf_prompt(task)

    # 3. Query agent
    await self.client.query(system_prompt)

    # 4. Stream messages and detect flags
    async for message in self.client.stream():
        self._process_message(message)
        self._detect_flags(message)

    # 5. Return results
    return {"walkthrough": self.walkthrough_steps, "flags": self.flags_found}
```

**Flag Detection:**
```python
FLAG_PATTERNS = [
    r"flag\{[^\}]+\}",
    r"HTB\{[^\}]+\}",
    # ... more patterns
]

def _detect_flags(self, text: str) -> list[str]:
    flags = []
    for pattern in self.FLAG_PATTERNS:
        matches = re.findall(pattern, text)
        flags.extend(matches)
    return flags
```

---

### 3. EventBus

**Purpose:** Decoupled communication between components

**Architecture:**
```python
class EventBus:
    _handlers: dict[EventType, list[Callable]]

    def subscribe(self, event_type, handler):
        """Register handler for event type"""

    def emit(self, event: Event):
        """Emit event to all subscribers"""
```

**Thread Safety:**
```python
def emit(self, event: Event) -> None:
    with self._handler_lock:
        handlers = self._handlers.get(event.type, []).copy()

    for handler in handlers:
        with contextlib.suppress(Exception):
            handler(event)  # Don't let one handler break others
```

**Event Flow:**
```
Agent → EventBus → [TUI, Tracer, Langfuse]
                     ↓       ↓        ↓
                  Display  Log   Telemetry
```

---

### 4. SessionStore

**Purpose:** Persist and restore agent sessions

**Storage Format:**
```python
@dataclass
class SessionInfo:
    session_id: str
    target: str
    status: SessionStatus
    flags_found: list[dict]
    # ... more fields
```

**File Operations:**
```python
def save(self, session: SessionInfo) -> None:
    path = self.SESSIONS_DIR / f"{session.session_id}.json"
    with open(path, "w") as f:
        json.dump(session.to_dict(), f, indent=2)

def load(self, session_id: str) -> SessionInfo:
    path = self.SESSIONS_DIR / f"{session_id}.json"
    with open(path) as f:
        data = json.load(f)
    return SessionInfo.from_dict(data)
```

---

## Data Flow

### 1. Startup Flow

```
1. CLI Entry (main.py)
   ├─> Parse arguments
   ├─> Load configuration
   └─> Initialize telemetry

2. Create Components
   ├─> EventBus singleton
   ├─> Tracer singleton
   ├─> AgentController
   │   └─> Backend (ClaudeCodeBackend)
   └─> TUI App (PentestGPTApp)

3. Subscribe to Events
   ├─> TUI subscribes to STATE_CHANGED, MESSAGE, TOOL, FLAG_FOUND
   ├─> Tracer subscribes for logging
   └─> Langfuse subscribes for telemetry

4. Start Agent
   └─> Controller.start()
       └─> Agent.execute(task)
```

---

### 2. Message Flow

```
┌─────────────┐
│ LLM Backend │
└──────┬──────┘
       │ Messages (text, tool_use, result)
       ↓
┌─────────────┐
│ PentestAgent│
└──────┬──────┘
       │ Parse messages
       │ Detect flags
       │ Track activity
       ↓
┌─────────────┐
│  EventBus   │
└──────┬──────┘
       │ Emit events
       ├──────────────┬──────────────┬──────────────┐
       ↓              ↓              ↓              ↓
   ┌───────┐    ┌────────┐    ┌────────┐    ┌─────────┐
   │  TUI  │    │ Tracer │    │Langfuse│    │ Session │
   └───────┘    └────────┘    └────────┘    └─────────┘
   Display      Log/Debug     Telemetry     Persist
```

---

### 3. Pause/Resume Flow

```
User presses Ctrl+P
       ↓
TUI emits USER_COMMAND (pause)
       ↓
EventBus propagates to Controller
       ↓
Controller sets pause flag
       ↓
Agent completes current message
       ↓
Controller transitions to PAUSED state
       ↓
EventBus emits STATE_CHANGED (paused)
       ↓
TUI shows input field
       ↓
User types instruction and presses Enter
       ↓
TUI emits USER_INPUT
       ↓
Controller receives input
       ↓
Controller.resume(instruction)
       ↓
Agent continues with new context
```

---

### 4. Flag Detection Flow

```
Agent receives message from LLM
       ↓
PentestAgent._detect_flags(text)
       ↓
Regex patterns match flag
       ↓
EventBus.emit_flag(flag, context)
       ↓
┌──────────┬──────────┬──────────┐
↓          ↓          ↓          ↓
TUI      Tracer   Session   Langfuse
Display    Log      Save    Telemetry
```

---

## Event-Driven Architecture

### Event Types and Usage

**STATE_CHANGED:**
```python
# Emitted by: Controller
# Data: {"state": "running", "details": "Target: 10.10.11.234"}
# Subscribers: TUI, Langfuse

bus.emit_state("running", "Target: 10.10.11.234")
```

**MESSAGE:**
```python
# Emitted by: Agent
# Data: {"text": "Starting nmap scan", "type": "info"}
# Subscribers: TUI, Tracer

bus.emit_message("Starting nmap scan", "info")
```

**TOOL:**
```python
# Emitted by: Agent
# Data: {"status": "start", "name": "bash", "args": {"command": "nmap ..."}}
# Subscribers: TUI, Tracer, Langfuse

bus.emit_tool("start", "bash", {"command": "nmap -sV 10.10.11.234"})
```

**FLAG_FOUND:**
```python
# Emitted by: Agent
# Data: {"flag": "flag{example}", "context": "user.txt"}
# Subscribers: TUI, Session, Langfuse

bus.emit_flag("flag{example}", "Found in /home/user/user.txt")
```

**USER_COMMAND:**
```python
# Emitted by: TUI
# Data: {"command": "pause"}
# Subscribers: Controller

bus.emit_command("pause")
```

**USER_INPUT:**
```python
# Emitted by: TUI
# Data: {"text": "Focus on port 8080"}
# Subscribers: Controller

bus.emit_input("Focus on port 8080")
```

---

### Event Handler Pattern

**TUI Event Handlers:**
```python
def _setup_event_handlers(self) -> None:
    bus = EventBus.get()
    bus.subscribe(EventType.MESSAGE, self._handle_message)
    bus.subscribe(EventType.STATE_CHANGED, self._handle_state)
    bus.subscribe(EventType.TOOL, self._handle_tool)
    bus.subscribe(EventType.FLAG_FOUND, self._handle_flag)

def _handle_message(self, event: Event) -> None:
    text = event.data.get("text", "")
    msg_type = event.data.get("type", "info")
    self.activity_feed.add_message(text, msg_type)

def _handle_flag(self, event: Event) -> None:
    flag = event.data.get("flag", "")
    self.activity_feed.add_flag(flag)
    self.show_flag_notification(flag)
```

---

## Session Lifecycle

### Session Creation

```python
# 1. Controller starts
controller = AgentController(config, backend)

# 2. Create session
session = SessionInfo(
    session_id=str(uuid.uuid4()),
    target=target,
    created_at=datetime.now(),
    status=SessionStatus.RUNNING,
    model=config.model
)

# 3. Save to disk
SessionStore().save(session)
```

---

### Session Updates

```python
# During execution
session.flags_found.append({"flag": "flag{...}", "context": "..."})
session.updated_at = datetime.now()
session.total_cost_usd += message_cost

# Save periodically
SessionStore().save(session)
```

---

### Session Resume

```python
# Load session
session = SessionStore().load(session_id)

# Restore state
controller.session = session
controller.target = session.target

# Resume backend if supported
if backend.supports_resume:
    await backend.resume(session.backend_session_id)
```

---

## Backend Abstraction

### Interface Design

```python
class AgentBackend(ABC):
    """Framework-agnostic backend interface"""

    async def connect(self) -> None:
        """Establish connection to agent"""

    async def query(self, prompt: str) -> None:
        """Send query to agent"""

    def receive_messages(self) -> AsyncIterator[AgentMessage]:
        """Stream messages from agent"""

    async def resume(self, session_id: str) -> bool:
        """Resume previous session"""
```

---

### Message Abstraction

```python
class MessageType(Enum):
    TEXT = "text"
    TOOL_START = "tool_start"
    TOOL_RESULT = "tool_result"
    RESULT = "result"
    ERROR = "error"

@dataclass
class AgentMessage:
    type: MessageType
    content: Any
    tool_name: str | None = None
    tool_args: dict | None = None
```

**Benefits:**
- Backend-agnostic message handling
- Easy to add new backends
- Consistent testing interface

---

### Current Implementation: ClaudeCodeBackend

```python
class ClaudeCodeBackend(AgentBackend):
    async def connect(self):
        from claude_agent_sdk import ClaudeSDKClient
        options = ClaudeAgentOptions(...)
        self._client = ClaudeSDKClient(options)

    async def query(self, prompt: str):
        await self._client.query(prompt)

    async def receive_messages(self):
        async for message in self._client.stream():
            yield self._convert_to_agent_message(message)
```

---

## Extension Points

### 1. Adding New Tools

**Create Tool:**
```python
# pentestgpt/tools/sqlmap.py
from pentestgpt.tools.base import BaseTool

class SQLMapTool(BaseTool):
    def __init__(self):
        super().__init__(
            name="sqlmap",
            description="Automated SQL injection tool"
        )

    async def execute(self, url: str, **kwargs) -> dict:
        # Execute sqlmap
        result = await run_sqlmap(url)
        return {"success": True, "result": result}
```

**Register Tool:**
```python
# pentestgpt/tools/registry.py
from pentestgpt.tools.sqlmap import SQLMapTool

class ToolRegistry:
    def _register_default_tools(self):
        self.register(TerminalTool())
        self.register(SQLMapTool())  # Add new tool
```

---

### 2. Adding New Event Types

**Define Event:**
```python
# pentestgpt/core/events.py
class EventType(Enum):
    # Existing events...
    CREDENTIAL_FOUND = auto()  # New event
```

**Emit Event:**
```python
# Anywhere in code
bus = EventBus.get()
bus.emit(Event(
    EventType.CREDENTIAL_FOUND,
    {"username": "admin", "password": "password123"}
))
```

**Subscribe:**
```python
# In TUI or other component
bus.subscribe(EventType.CREDENTIAL_FOUND, self._handle_credential)
```

---

### 3. Adding New Backend

**Implement Interface:**
```python
# pentestgpt/core/openai_backend.py
class OpenAIBackend(AgentBackend):
    async def connect(self):
        import openai
        self.client = openai.AsyncClient(api_key=...)

    async def query(self, prompt: str):
        await self.client.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}]
        )

    async def receive_messages(self):
        # Stream and convert to AgentMessage
        ...
```

**Use New Backend:**
```python
# pentestgpt/interface/main.py
if args.backend == "openai":
    backend = OpenAIBackend(...)
else:
    backend = ClaudeCodeBackend(...)
```

---

## Performance Considerations

### 1. Async/Await Throughout

**All I/O is async:**
```python
# Agent execution
async def execute(self, task: str):
    await self.client.query(task)
    async for message in self.client.stream():
        await self._process_message(message)

# Controller
async def start(self):
    await self.backend.connect()
    await self.backend.query(prompt)
```

**Benefits:**
- Non-blocking I/O
- Efficient resource usage
- Concurrent operations

---

### 2. Event Bus Optimization

**Handler Isolation:**
```python
for handler in handlers:
    with contextlib.suppress(Exception):
        handler(event)  # Failures don't break other handlers
```

**Thread Safety:**
```python
with self._handler_lock:
    handlers = self._handlers.get(event.type, []).copy()
# Process outside lock to minimize lock time
```

---

### 3. Session Persistence Strategy

**Write-Through Cache:**
- Sessions saved immediately on updates
- No risk of data loss
- Fast reads from memory

**Async Writes:**
```python
async def save_async(self, session: SessionInfo):
    await asyncio.to_thread(self._write_to_disk, session)
```

---

### 4. TUI Performance

**Lazy Rendering:**
- Only visible items rendered
- Scroll performance optimized
- CSS-based styling (compiled once)

**Message Batching:**
```python
def add_messages(self, messages: list[str]):
    # Batch update instead of individual updates
    self.messages.extend(messages)
    self.refresh()
```

---

## Architectural Decisions

### Why EventBus Instead of Direct Calls?

**Decision:** Use pub/sub event bus for TUI-Agent communication

**Rationale:**
- **Decoupling:** TUI doesn't depend on Agent internals
- **Extensibility:** Easy to add new subscribers (logging, telemetry)
- **Testing:** Can test components independently
- **Thread Safety:** Centralized synchronization

**Trade-offs:**
- Slightly more complex code
- Debugging requires tracing events
- Small performance overhead (acceptable for this use case)

---

### Why Abstract Backend Interface?

**Decision:** Use abstract `AgentBackend` protocol instead of direct Claude SDK usage

**Rationale:**
- **Future-Proof:** Support multiple LLM providers
- **Testing:** Mock backends for unit tests
- **Flexibility:** Switch backends without changing agent logic

**Trade-offs:**
- Extra abstraction layer
- Potential over-engineering for single backend

**Conclusion:** Worth it for long-term maintainability

---

### Why 5-State Model?

**Decision:** Use IDLE/RUNNING/PAUSED/COMPLETED/ERROR states

**Rationale:**
- **Clear Semantics:** Each state has well-defined meaning
- **UI State Mapping:** Easy to reflect in TUI
- **Error Handling:** Explicit error state
- **Pause/Resume:** Clean separation of active/paused

**Alternative Considered:** Simple running/stopped binary
**Rejected Because:** Insufficient for pause/resume and error tracking

---

### Why File-Based Sessions?

**Decision:** Store sessions as JSON files in `~/.pentestgpt/sessions/`

**Rationale:**
- **Simplicity:** No database dependency
- **Portability:** Easy to backup, inspect, share
- **Debugging:** Human-readable JSON
- **Reliability:** No DB connection issues

**Trade-offs:**
- Not suitable for thousands of sessions (acceptable for this use case)
- No concurrent access control (single-user tool)

---

## Next Steps

- **[PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md)** - Project organization
- **[FEATURES.md](./FEATURES.md)** - Feature documentation
- **[CODE_GUIDE.md](./CODE_GUIDE.md)** - Code walkthroughs and explanations

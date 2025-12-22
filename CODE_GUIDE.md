# PentestGPT Code Guide

This guide provides detailed code explanations and walkthroughs of key modules in PentestGPT, helping developers understand implementation details.

## Table of Contents

- [Entry Point & CLI](#entry-point--cli)
- [Agent Controller](#agent-controller)
- [Pentest Agent](#pentest-agent)
- [Event Bus System](#event-bus-system)
- [Backend Abstraction](#backend-abstraction)
- [Session Management](#session-management)
- [TUI Application](#tui-application)
- [Activity Tracking](#activity-tracking)
- [Telemetry Integration](#telemetry-integration)
- [Tool Framework](#tool-framework)
- [Configuration Management](#configuration-management)

---

## Entry Point & CLI

**File:** `pentestgpt/interface/main.py`

### Main Function

```python
def main() -> int:
    """Main entry point for PentestGPT CLI."""
    parser = argparse.ArgumentParser(
        description="PentestGPT - AI-Powered CTF Challenge Solver",
        formatter_class=argparse.RawDescriptionHelpFormatter,
    )

    # Required arguments
    parser.add_argument(
        "--target",
        type=str,
        required=True,
        help="Target IP address or URL",
    )

    # Optional arguments
    parser.add_argument(
        "--instruction",
        type=str,
        help="Additional context or hints for the agent",
    )
    parser.add_argument(
        "--non-interactive",
        action="store_true",
        help="Run without TUI (for automation)",
    )
    parser.add_argument(
        "--raw",
        action="store_true",
        help="Raw output mode (no TUI, streaming output)",
    )
    parser.add_argument(
        "--debug",
        action="store_true",
        help="Enable debug logging",
    )
    parser.add_argument(
        "--no-telemetry",
        action="store_true",
        help="Disable Langfuse telemetry",
    )
    parser.add_argument(
        "--resume",
        type=str,
        metavar="SESSION_ID",
        help="Resume a previous session",
    )

    args = parser.parse_args()

    # Initialize telemetry (opt-out via --no-telemetry)
    init_langfuse(disabled=args.no_telemetry)

    # Mode selection
    if args.raw:
        return asyncio.run(run_raw_mode(args))
    elif args.non_interactive:
        return asyncio.run(run_non_interactive(args))
    else:
        return asyncio.run(run_tui_mode(args))
```

**Key Points:**
- `argparse` for CLI argument parsing
- Telemetry opt-in/opt-out logic
- Mode selection (TUI vs Raw vs Non-Interactive)
- Async entry via `asyncio.run()`

---

### Mode Implementations

**TUI Mode:**
```python
async def run_tui_mode(args) -> int:
    """Run with interactive TUI."""
    from pentestgpt.interface.tui import PentestGPTApp

    app = PentestGPTApp(
        target=args.target,
        custom_instruction=args.instruction,
        debug=args.debug,
        resume_session=args.resume,
    )

    try:
        await app.run_async()
        return 0
    except Exception as e:
        logger.error(f"TUI error: {e}")
        return 1
    finally:
        shutdown_langfuse()
```

**Raw Mode:**
```python
async def run_raw_mode(args) -> int:
    """Run in raw streaming mode (no TUI)."""
    # Create controller
    config = PentestGPTConfig(...)
    backend = ClaudeCodeBackend(...)
    controller = AgentController(config, backend)

    # Subscribe to events for console output
    bus = EventBus.get()
    bus.subscribe(EventType.MESSAGE, lambda e: print(e.data["text"]))
    bus.subscribe(EventType.FLAG_FOUND, lambda e: print(f"FLAG: {e.data['flag']}"))

    # Run agent
    await controller.start()

    return 0
```

**Non-Interactive Mode:**
```python
async def run_non_interactive(args) -> int:
    """Run to completion without user input."""
    # Similar to raw mode but waits for completion
    await controller.start()

    # Wait until completed or error
    while controller.state in [AgentState.RUNNING, AgentState.PAUSED]:
        await asyncio.sleep(1)

    return 0 if controller.state == AgentState.COMPLETED else 1
```

---

## Agent Controller

**File:** `pentestgpt/core/controller.py`

### Class Structure

```python
class AgentController:
    """
    Central orchestrator with lifecycle management.

    Features:
    - Framework-agnostic via AgentBackend
    - Pause/resume/stop control
    - Instruction injection
    - Session persistence
    """

    def __init__(
        self,
        config: PentestGPTConfig,
        backend: AgentBackend,
        session_store: SessionStore | None = None,
    ):
        self.config = config
        self.backend = backend
        self.session_store = session_store or SessionStore()

        self.state = AgentState.IDLE
        self.session: SessionInfo | None = None
        self._pause_requested = False
        self._stop_requested = False

        # Subscribe to user commands
        EventBus.get().subscribe(EventType.USER_COMMAND, self._handle_user_command)
```

---

### Start Method

```python
async def start(self) -> None:
    """Start agent execution."""
    if self.state != AgentState.IDLE:
        raise RuntimeError(f"Cannot start from state {self.state}")

    # Create session
    self.session = SessionInfo(
        session_id=str(uuid.uuid4()),
        target=self.config.target,
        created_at=datetime.now(),
        status=SessionStatus.RUNNING,
        model=self.config.model,
    )
    self.session_store.save(self.session)

    # Transition to RUNNING
    self._transition_state(AgentState.RUNNING, f"Target: {self.config.target}")

    try:
        # Connect to backend
        await self.backend.connect()

        # Build task prompt
        task = f"Target: {self.config.target}"
        if self.config.custom_instruction:
            task += f"\n\nAdditional Context: {self.config.custom_instruction}"

        # Query agent
        await self.backend.query(task)

        # Process messages
        await self._process_messages()

        # Complete successfully
        self._transition_state(AgentState.COMPLETED, "Task completed")
        self._finalize_session(SessionStatus.COMPLETED)

    except Exception as e:
        logger.error(f"Agent error: {e}", exc_info=True)
        self._transition_state(AgentState.ERROR, str(e))
        self._finalize_session(SessionStatus.ERROR, str(e))
        raise
```

**Key Points:**
- State validation before start
- Session creation and persistence
- Backend connection
- Message processing loop
- Error handling with state transitions

---

### Message Processing Loop

```python
async def _process_messages(self) -> None:
    """Process messages from backend."""
    async for message in self.backend.receive_messages():
        # Check for stop request
        if self._stop_requested:
            break

        # Check for pause request
        if self._pause_requested:
            await self._handle_pause()
            self._pause_requested = False

        # Process message based on type
        if message.type == MessageType.TEXT:
            self._handle_text_message(message)
        elif message.type == MessageType.TOOL_START:
            self._handle_tool_start(message)
        elif message.type == MessageType.TOOL_RESULT:
            self._handle_tool_result(message)
        elif message.type == MessageType.RESULT:
            # Agent completed task
            break
        elif message.type == MessageType.ERROR:
            raise RuntimeError(message.content)

        # Update session periodically
        self._update_session()
```

**Control Flow:**
- Streams messages from backend
- Checks for pause/stop requests
- Routes messages by type
- Periodic session updates

---

### Pause/Resume Implementation

```python
async def pause(self) -> None:
    """Request pause at next message boundary."""
    if self.state != AgentState.RUNNING:
        raise RuntimeError(f"Cannot pause from state {self.state}")
    self._pause_requested = True

async def _handle_pause(self) -> None:
    """Transition to paused state and wait for resume."""
    self._transition_state(AgentState.PAUSED, "Waiting for user input")

    # Wait for resume
    while self.state == AgentState.PAUSED:
        await asyncio.sleep(0.1)

async def resume(self, instruction: str | None = None) -> None:
    """Resume agent with optional new instruction."""
    if self.state != AgentState.PAUSED:
        raise RuntimeError(f"Cannot resume from state {self.state}")

    # Add instruction to session
    if instruction:
        self.session.user_instructions.append(instruction)
        await self.backend.query(instruction)

    # Transition back to running
    self._transition_state(AgentState.RUNNING, "Resumed")
```

**Pause/Resume Flow:**
1. User requests pause → `_pause_requested = True`
2. Controller pauses at message boundary → `PAUSED` state
3. User provides instruction (optional)
4. Resume transitions back to `RUNNING`

---

### Flag Detection

```python
FLAG_PATTERNS = [
    r"flag\{[^\}]+\}",
    r"FLAG\{[^\}]+\}",
    r"HTB\{[^\}]+\}",
    r"CTF\{[^\}]+\}",
    r"[A-Za-z0-9_]+\{[^\}]+\}",
    r"\b[a-f0-9]{32}\b",
]

def _detect_flags(self, text: str) -> None:
    """Detect flags in text and emit events."""
    for pattern in self.FLAG_PATTERNS:
        matches = re.finditer(pattern, text)
        for match in matches:
            flag = match.group()
            # Check if already found
            if not any(f["flag"] == flag for f in self.session.flags_found):
                # New flag!
                self.session.flags_found.append({
                    "flag": flag,
                    "context": text[:200],  # Capture context
                    "timestamp": datetime.now().isoformat(),
                })
                EventBus.get().emit_flag(flag, text[:200])
                logger.info(f"FLAG DETECTED: {flag}")
```

**Flag Detection Strategy:**
- Multiple regex patterns for different formats
- Deduplication (don't report same flag twice)
- Context capture for debugging
- Event emission for UI/telemetry

---

## Pentest Agent

**File:** `pentestgpt/core/agent.py`

### Initialization

```python
class PentestAgent:
    """Enhanced Claude Code agent with activity tracking and streaming updates."""

    def __init__(
        self,
        config: PentestGPTConfig,
        tracer: Tracer | None = None,
    ):
        self.config = config
        self.tracer = tracer or get_global_tracer()
        self.client: ClaudeSDKClient | None = None
        self.walkthrough_steps: list[str] = []
        self.flags_found: list[dict[str, str]] = []
```

---

### Execute Method

```python
async def execute(self, task: str) -> dict[str, Any]:
    """Execute a penetration testing task."""
    logger.debug("=" * 80)
    logger.debug(f"EXECUTE START - Task: {task[:100]}...")
    self.tracer.track_agent_status("starting", "Initializing agent")

    try:
        # 1. Setup Claude Code client
        options = ClaudeAgentOptions(
            cwd=str(self.config.working_directory),
            permission_mode=self.config.permission_mode,
            model=self.config.model,
        )

        logger.debug(f"Creating ClaudeSDKClient with options: {options}")
        self.client = ClaudeSDKClient(options)

        # 2. Build system prompt
        system_prompt = get_ctf_prompt(task)
        logger.debug(f"System prompt length: {len(system_prompt)} chars")

        # 3. Query agent
        logger.debug("Querying agent...")
        await self.client.query(system_prompt)

        # 4. Stream and process messages
        logger.debug("Starting message stream...")
        async for message in self.client.stream():
            await self._process_message(message)

            # Check for completion
            if isinstance(message, ResultMessage):
                break

        # 5. Return results
        logger.debug("Agent execution completed")
        return {
            "success": True,
            "walkthrough": self.walkthrough_steps,
            "flags": self.flags_found,
        }

    except CLINotFoundError:
        error = "Claude CLI not found. Please install with 'npm install -g @anthropic-ai/claude-cli'"
        logger.error(error)
        self.tracer.track_message(error, "error")
        return {"success": False, "error": error}

    except CLIConnectionError as e:
        error = f"Claude CLI connection error: {e}"
        logger.error(error)
        self.tracer.track_message(error, "error")
        return {"success": False, "error": error}

    except Exception as e:
        error = f"Unexpected error: {e}"
        logger.error(error, exc_info=True)
        self.tracer.track_message(error, "error")
        return {"success": False, "error": error}
```

**Execution Flow:**
1. Create Claude SDK client with options
2. Build CTF-specific system prompt
3. Query agent with prompt
4. Stream messages and process
5. Return results with walkthrough and flags

---

### Message Processing

```python
async def _process_message(self, message: Any) -> None:
    """Process a message from Claude SDK."""
    if isinstance(message, AssistantMessage):
        # Text message from agent
        for block in message.content:
            if isinstance(block, TextBlock):
                text = block.text
                logger.debug(f"Assistant message: {text[:100]}...")
                self.tracer.track_message(text, "info")
                EventBus.get().emit_message(text, "info")

                # Detect flags
                self._detect_flags_in_text(text)

            elif isinstance(block, ToolUseBlock):
                # Tool execution started
                tool_name = block.name
                tool_args = block.input
                logger.debug(f"Tool use: {tool_name} with args {tool_args}")

                activity_id = self.tracer.track_tool_start(tool_name, tool_args)
                EventBus.get().emit_tool("start", tool_name, tool_args)

    elif isinstance(message, ResultMessage):
        # Final result from agent
        logger.debug("Received result message - task completed")
        self.tracer.track_message("Task completed", "success")
        EventBus.get().emit_message("Task completed", "success")
```

**Message Types:**
- `AssistantMessage` - Text or tool use from agent
  - `TextBlock` - Text content (analysis, reasoning, results)
  - `ToolUseBlock` - Tool execution (bash command, file read, etc.)
- `ResultMessage` - Task completion signal

---

### Flag Detection Implementation

```python
def _detect_flags_in_text(self, text: str) -> None:
    """Scan text for CTF flags using regex patterns."""
    for pattern in self.FLAG_PATTERNS:
        matches = re.finditer(pattern, text, re.IGNORECASE)
        for match in matches:
            flag = match.group()

            # Avoid duplicates
            if any(f["flag"] == flag for f in self.flags_found):
                continue

            # Extract context (surrounding text)
            start = max(0, match.start() - 50)
            end = min(len(text), match.end() + 50)
            context = text[start:end]

            # Record flag
            flag_info = {
                "flag": flag,
                "context": context,
                "timestamp": datetime.now().isoformat(),
            }
            self.flags_found.append(flag_info)

            # Emit events
            EventBus.get().emit_flag(flag, context)
            self.tracer.track_message(f"FLAG FOUND: {flag}", "success")

            logger.info(f"🚩 FLAG DETECTED: {flag}")
            logger.debug(f"   Context: {context}")
```

**Detection Strategy:**
- Scan every text message with regex patterns
- Extract surrounding context for debugging
- Emit flag event for UI and telemetry
- Log to debug file

---

## Event Bus System

**File:** `pentestgpt/core/events.py`

### EventBus Implementation

```python
class EventBus:
    """Minimal thread-safe event bus for pub/sub communication."""

    _instance: Optional["EventBus"] = None
    _lock = threading.Lock()

    def __init__(self) -> None:
        self._handlers: dict[EventType, list[Callable]] = {}
        self._handler_lock = threading.Lock()

    @classmethod
    def get(cls) -> "EventBus":
        """Get singleton EventBus instance (thread-safe)."""
        with cls._lock:
            if cls._instance is None:
                cls._instance = cls()
            return cls._instance

    def subscribe(self, event_type: EventType, handler: Callable[[Event], None]) -> None:
        """Subscribe a handler to an event type."""
        with self._handler_lock:
            if event_type not in self._handlers:
                self._handlers[event_type] = []
            if handler not in self._handlers[event_type]:
                self._handlers[event_type].append(handler)

    def emit(self, event: Event) -> None:
        """Emit an event to all subscribers."""
        # Copy handlers list while holding lock
        with self._handler_lock:
            handlers = self._handlers.get(event.type, []).copy()

        # Call handlers outside lock to avoid deadlocks
        for handler in handlers:
            # Don't let one handler break others
            with contextlib.suppress(Exception):
                handler(event)
```

**Thread Safety:**
- Lock acquisition for shared data (`_handlers`)
- Copy handler list before iteration
- Release lock before calling handlers (avoid deadlock)
- Exception suppression (isolated failures)

---

### Convenience Methods

```python
def emit_state(self, state: str, details: str = "") -> None:
    """Emit a state change event."""
    self.emit(Event(EventType.STATE_CHANGED, {"state": state, "details": details}))

def emit_message(self, text: str, msg_type: str = "info") -> None:
    """Emit a message event."""
    self.emit(Event(EventType.MESSAGE, {"text": text, "type": msg_type}))

def emit_tool(self, status: str, name: str, args: dict | None = None) -> None:
    """Emit a tool event."""
    self.emit(Event(
        EventType.TOOL,
        {"status": status, "name": name, "args": args or {}}
    ))

def emit_flag(self, flag: str, context: str = "") -> None:
    """Emit a flag found event."""
    self.emit(Event(EventType.FLAG_FOUND, {"flag": flag, "context": context}))
```

**Benefits:**
- Type-safe event creation
- Consistent event data structure
- Reduced boilerplate

---

## Backend Abstraction

**File:** `pentestgpt/core/backend.py`

### Abstract Interface

```python
class AgentBackend(ABC):
    """Abstract interface for agent backends."""

    @abstractmethod
    async def connect(self) -> None:
        """Establish connection to the agent."""
        ...

    @abstractmethod
    async def disconnect(self) -> None:
        """Close connection."""
        ...

    @abstractmethod
    async def query(self, prompt: str) -> None:
        """Send a query/instruction to the agent."""
        ...

    @abstractmethod
    def receive_messages(self) -> AsyncIterator[AgentMessage]:
        """Async iterator yielding messages from agent."""
        ...

    @property
    @abstractmethod
    def session_id(self) -> str | None:
        """Current session ID (if backend supports sessions)."""
        ...

    @property
    def supports_resume(self) -> bool:
        """Whether this backend supports session resume."""
        return False

    @abstractmethod
    async def resume(self, session_id: str) -> bool:
        """Resume a previous session. Returns success."""
        ...
```

---

### ClaudeCodeBackend Implementation

```python
class ClaudeCodeBackend(AgentBackend):
    """Claude Code SDK implementation of AgentBackend."""

    def __init__(self, working_directory: str, system_prompt: str, model: str):
        self._cwd = working_directory
        self._system_prompt = system_prompt
        self._model = model
        self._client: Any = None
        self._session_id: str | None = None

    async def connect(self) -> None:
        """Connect to Claude Code CLI."""
        from claude_agent_sdk import ClaudeAgentOptions, ClaudeSDKClient

        options = ClaudeAgentOptions(
            cwd=self._cwd,
            model=self._model,
            permission_mode="bypassPermissions",
        )
        self._client = ClaudeSDKClient(options)

    async def query(self, prompt: str) -> None:
        """Send query to agent."""
        if not self._client:
            raise RuntimeError("Not connected - call connect() first")
        await self._client.query(prompt)

    async def receive_messages(self) -> AsyncIterator[AgentMessage]:
        """Stream messages from Claude SDK."""
        if not self._client:
            raise RuntimeError("Not connected")

        async for message in self._client.stream():
            yield self._convert_message(message)

    def _convert_message(self, sdk_message: Any) -> AgentMessage:
        """Convert Claude SDK message to AgentMessage."""
        if isinstance(sdk_message, AssistantMessage):
            # Extract text content
            text_content = ""
            for block in sdk_message.content:
                if isinstance(block, TextBlock):
                    text_content += block.text

            return AgentMessage(
                type=MessageType.TEXT,
                content=text_content,
            )

        elif isinstance(sdk_message, ResultMessage):
            return AgentMessage(
                type=MessageType.RESULT,
                content=sdk_message.content,
            )

        # ... handle other message types

    @property
    def session_id(self) -> str | None:
        return self._session_id
```

**Key Abstraction:**
- Hides Claude SDK details from rest of system
- Easy to add new backends (OpenAI, Gemini, etc.)
- Testable with mock backends

---

## Session Management

**File:** `pentestgpt/core/session.py`

### SessionInfo Data Class

```python
@dataclass
class SessionInfo:
    """Session state - framework agnostic."""

    session_id: str
    target: str
    created_at: datetime
    status: SessionStatus = SessionStatus.RUNNING
    backend_session_id: str | None = None
    updated_at: datetime | None = None
    task: str = ""
    user_instructions: list[str] = field(default_factory=list)
    flags_found: list[dict[str, str]] = field(default_factory=list)
    total_cost_usd: float = 0.0
    model: str = ""
    last_error: str | None = None

    def to_dict(self) -> dict[str, Any]:
        """Serialize to JSON-compatible dict."""
        return {
            "session_id": self.session_id,
            "target": self.target,
            "created_at": self.created_at.isoformat(),
            "status": self.status.value,
            # ... all fields
        }

    @classmethod
    def from_dict(cls, data: dict[str, Any]) -> "SessionInfo":
        """Deserialize from dict."""
        return cls(
            session_id=data["session_id"],
            target=data["target"],
            created_at=datetime.fromisoformat(data["created_at"]),
            status=SessionStatus(data["status"]),
            # ... all fields
        )
```

---

### SessionStore Implementation

```python
class SessionStore:
    """Simple file-based session persistence."""

    SESSIONS_DIR = Path.home() / ".pentestgpt" / "sessions"

    def __init__(self):
        self.SESSIONS_DIR.mkdir(parents=True, exist_ok=True)

    def save(self, session: SessionInfo) -> None:
        """Save session to JSON file."""
        session_file = self.SESSIONS_DIR / f"{session.session_id}.json"
        with open(session_file, "w") as f:
            json.dump(session.to_dict(), f, indent=2)
        logger.debug(f"Session saved: {session_file}")

    def load(self, session_id: str) -> SessionInfo:
        """Load session from JSON file."""
        session_file = self.SESSIONS_DIR / f"{session_id}.json"
        if not session_file.exists():
            raise FileNotFoundError(f"Session not found: {session_id}")

        with open(session_file) as f:
            data = json.load(f)
        return SessionInfo.from_dict(data)

    def list_sessions(self) -> list[SessionInfo]:
        """List all saved sessions."""
        sessions = []
        for session_file in self.SESSIONS_DIR.glob("*.json"):
            try:
                with open(session_file) as f:
                    data = json.load(f)
                sessions.append(SessionInfo.from_dict(data))
            except Exception as e:
                logger.warning(f"Error loading {session_file}: {e}")
        return sorted(sessions, key=lambda s: s.created_at, reverse=True)

    def delete(self, session_id: str) -> bool:
        """Delete a session."""
        session_file = self.SESSIONS_DIR / f"{session_id}.json"
        if session_file.exists():
            session_file.unlink()
            return True
        return False
```

**Design Choices:**
- File-based storage (no DB dependency)
- JSON format (human-readable, debuggable)
- UUID session IDs (unique, collision-free)
- Directory: `~/.pentestgpt/sessions/`

---

## TUI Application

**File:** `pentestgpt/interface/tui.py`

### App Structure

```python
class PentestGPTApp(App[None]):
    """Main TUI application for PentestGPT CTF Solver."""

    CSS_PATH = "styles.tcss"  # Textual CSS styling
    TITLE = "PentestGPT - CTF Challenge Solver"

    # Reactive state
    show_splash: reactive[bool] = reactive(default=True)
    agent_state: reactive[str] = reactive(default="idle")

    # Keyboard bindings
    BINDINGS = [
        Binding("f1", "toggle_help", "Help", priority=True),
        Binding("ctrl+p", "toggle_pause", "Pause/Resume", priority=True),
        Binding("ctrl+q", "request_quit", "Quit", priority=True),
    ]

    def __init__(self, target: str, custom_instruction: str | None = None):
        super().__init__()
        self.target = target
        self.custom_instruction = custom_instruction

        # Create controller
        config = PentestGPTConfig(target=target)
        backend = ClaudeCodeBackend(...)
        self.controller = AgentController(config, backend)

        # Subscribe to events
        self._setup_event_handlers()
```

---

### Lifecycle Hooks

```python
def compose(self) -> ComposeResult:
    """Create child widgets."""
    yield Header()
    yield ActivityFeed(id="activity_feed")
    yield Footer()

def on_mount(self) -> None:
    """Called when app is mounted."""
    # Show splash screen
    self.push_screen(SplashScreen())

    # Start agent in background
    asyncio.create_task(self._start_agent())

async def _start_agent(self) -> None:
    """Start agent execution."""
    await asyncio.sleep(2)  # Wait for splash

    # Dismiss splash
    self.show_splash = False
    self.pop_screen()

    # Start controller
    await self.controller.start()
```

---

### Event Handlers

```python
def _setup_event_handlers(self) -> None:
    """Subscribe to EventBus events."""
    bus = EventBus.get()
    bus.subscribe(EventType.MESSAGE, self._on_message)
    bus.subscribe(EventType.STATE_CHANGED, self._on_state_change)
    bus.subscribe(EventType.TOOL, self._on_tool)
    bus.subscribe(EventType.FLAG_FOUND, self._on_flag)

def _on_message(self, event: Event) -> None:
    """Handle MESSAGE events."""
    text = event.data.get("text", "")
    msg_type = event.data.get("type", "info")

    # Add to activity feed
    feed = self.query_one("#activity_feed", ActivityFeed)
    feed.add_message(text, msg_type)

def _on_state_change(self, event: Event) -> None:
    """Handle STATE_CHANGED events."""
    state = event.data.get("state", "")
    self.agent_state = state

    # Update UI based on state
    if state == "paused":
        self._show_input_field()
    elif state == "completed":
        self._show_completion_dialog()

def _on_flag(self, event: Event) -> None:
    """Handle FLAG_FOUND events."""
    flag = event.data.get("flag", "")

    # Highlight flag in activity feed
    feed = self.query_one("#activity_feed", ActivityFeed)
    feed.add_flag(flag)

    # Show notification
    self.notify(f"🚩 FLAG FOUND: {flag}", severity="success")
```

---

### Action Handlers (Keyboard)

```python
def action_toggle_help(self) -> None:
    """Show help screen (F1)."""
    self.push_screen(HelpScreen())

def action_toggle_pause(self) -> None:
    """Pause/resume agent (Ctrl+P)."""
    if self.controller.state == AgentState.RUNNING:
        asyncio.create_task(self.controller.pause())
    elif self.controller.state == AgentState.PAUSED:
        # Get instruction from input field
        instruction = self.get_input_value()
        asyncio.create_task(self.controller.resume(instruction))

def action_request_quit(self) -> None:
    """Request quit (Ctrl+Q)."""
    self.push_screen(QuitScreen())
```

---

## Activity Tracking

**File:** `pentestgpt/core/tracer.py`

### Tracer Implementation

```python
class Tracer:
    """Lightweight tracer for tracking agent activity."""

    def __init__(self):
        self._lock = threading.Lock()
        self._activities: list[dict[str, Any]] = []
        self._on_activity_callback: Callable | None = None

    def track_message(self, message: str, message_type: str = "info") -> None:
        """Track a simple message."""
        activity = {
            "type": "message",
            "message": message,
            "message_type": message_type,
            "timestamp": datetime.now(),
        }

        with self._lock:
            self._activities.append(activity)

        if self._on_activity_callback:
            self._on_activity_callback(activity)

    def track_tool_start(self, tool_name: str, args: dict) -> int:
        """Track tool execution start. Returns activity ID."""
        activity = {
            "type": "tool",
            "tool_name": tool_name,
            "args": args,
            "status": "running",
            "timestamp": datetime.now(),
        }

        with self._lock:
            activity_id = len(self._activities)
            self._activities.append(activity)

        if self._on_activity_callback:
            self._on_activity_callback(activity)

        return activity_id

    def track_tool_complete(self, activity_id: int, result: Any) -> None:
        """Mark tool execution as complete."""
        with self._lock:
            if 0 <= activity_id < len(self._activities):
                self._activities[activity_id]["status"] = "completed"
                self._activities[activity_id]["result"] = result
```

**Usage Pattern:**
```python
tracer = get_global_tracer()

# Message tracking
tracer.track_message("Starting nmap scan", "info")

# Tool tracking
activity_id = tracer.track_tool_start("bash", {"command": "nmap -sV 10.10.11.234"})
# ... execute tool ...
tracer.track_tool_complete(activity_id, {"output": "..."})
```

---

## Telemetry Integration

**File:** `pentestgpt/core/langfuse.py`

### Initialization

```python
def init_langfuse(disabled: bool = False) -> bool:
    """Initialize Langfuse client for telemetry."""
    global _langfuse_client, _user_id

    # Check opt-out
    if disabled:
        return False

    env_value = os.getenv("LANGFUSE_ENABLED", "true").lower()
    if env_value in ("0", "false", "no", "off"):
        return False

    # Set credentials (hardcoded for PentestGPT project)
    os.environ.setdefault("LANGFUSE_PUBLIC_KEY", "pk-lf-...")
    os.environ.setdefault("LANGFUSE_SECRET_KEY", "sk-lf-...")
    os.environ.setdefault("LANGFUSE_HOST", "https://us.cloud.langfuse.com")

    try:
        from langfuse import get_client
        _langfuse_client = get_client()
        _user_id = _get_or_create_user_id()
        _subscribe_to_events()
        logger.info(f"Langfuse telemetry initialized")
        return True
    except Exception as e:
        logger.warning(f"Langfuse initialization failed: {e}")
        return False
```

---

### Event Handlers

```python
def _subscribe_to_events() -> None:
    """Subscribe to EventBus for telemetry."""
    bus = EventBus.get()
    bus.subscribe(EventType.STATE_CHANGED, _handle_state)
    bus.subscribe(EventType.MESSAGE, _handle_message)
    bus.subscribe(EventType.TOOL, _handle_tool)
    bus.subscribe(EventType.FLAG_FOUND, _handle_flag)

def _handle_state(event: Event) -> None:
    """Track state changes as Langfuse spans."""
    global _current_span

    state = event.data.get("state")

    if state == "running":
        # Start new span
        _current_span = _langfuse_client.start_span(
            name=f"pentestgpt:{target}",
            input={"target": target},
            metadata={"user_id": _user_id}
        )

    elif state in ("completed", "error"):
        # End span
        if _current_span:
            _current_span.update(output={"final_state": state})
            _current_span.end()
        _langfuse_client.flush()

def _handle_tool(event: Event) -> None:
    """Track tool executions."""
    if not _current_span:
        return

    name = event.data.get("name")
    args = event.data.get("args", {})

    # Create nested span for tool
    tool_span = _current_span.start_span(
        name=f"tool-{name}",
        input=args,
        metadata={"tool_name": name}
    )
    tool_span.end()
```

**Privacy:**
- Only metadata tracked (no sensitive data)
- User ID persistent across sessions
- Opt-out via `--no-telemetry` or env var

---

## Tool Framework

**File:** `pentestgpt/tools/base.py`, `pentestgpt/tools/registry.py`

### BaseTool Abstract Class

```python
class BaseTool(ABC):
    """Base class for all tools in PentestGPT."""

    def __init__(self, name: str, description: str):
        self.name = name
        self.description = description

    @abstractmethod
    async def execute(self, *args: Any, **kwargs: Any) -> dict[str, Any]:
        """Execute the tool with given arguments."""
        pass

    def to_dict(self) -> dict[str, Any]:
        """Convert tool info to dictionary for tracer."""
        return {
            "tool_name": self.name,
            "description": self.description,
        }
```

---

### ToolRegistry

```python
class ToolRegistry:
    """Registry for managing and accessing tools."""

    def __init__(self):
        self._tools: dict[str, BaseTool] = {}
        self._register_default_tools()

    def _register_default_tools(self):
        """Register default built-in tools."""
        self.register(TerminalTool())

    def register(self, tool: BaseTool):
        """Register a new tool."""
        self._tools[tool.name] = tool

    def get(self, name: str) -> BaseTool | None:
        """Get a tool by name."""
        return self._tools.get(name)

# Global registry
_global_registry: ToolRegistry | None = None

def get_registry() -> ToolRegistry:
    """Get the global tool registry."""
    global _global_registry
    if _global_registry is None:
        _global_registry = ToolRegistry()
    return _global_registry
```

---

## Configuration Management

**File:** `pentestgpt/core/config.py`

### Pydantic Settings

```python
from pydantic_settings import BaseSettings

class PentestGPTConfig(BaseSettings):
    """Configuration for PentestGPT agent."""

    # Required
    target: str

    # Optional
    working_directory: Path = Path("/workspace")
    model: str = "claude-sonnet-4-5"
    permission_mode: str = "bypassPermissions"
    custom_instruction: str | None = None
    debug: bool = False

    # Load from .env file
    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"
        env_prefix = "PENTESTGPT_"

# Usage
config = PentestGPTConfig(target="10.10.11.234")
# Or from environment:
# PENTESTGPT_TARGET=10.10.11.234
# PENTESTGPT_MODEL=claude-opus-4
```

**Benefits:**
- Type validation
- Environment variable support
- `.env` file loading
- Default values
- IDE autocomplete

---

## Summary

This code guide covered the implementation details of PentestGPT's key modules:

1. **Entry Point** - CLI argument parsing and mode selection
2. **Agent Controller** - Central orchestration and state management
3. **Pentest Agent** - LLM integration and flag detection
4. **Event Bus** - Decoupled pub/sub communication
5. **Backend Abstraction** - Framework-agnostic LLM interface
6. **Session Management** - File-based persistence
7. **TUI Application** - Textual-based user interface
8. **Activity Tracking** - Debug and monitoring system
9. **Telemetry** - Anonymous usage analytics
10. **Tool Framework** - Extensible tool system
11. **Configuration** - Pydantic settings management

For architectural concepts and design patterns, see [ARCHITECTURE.md](./ARCHITECTURE.md).

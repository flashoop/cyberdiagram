# PentestGPT Features Guide

This guide provides comprehensive documentation of all PentestGPT features, organized by functional area.

## Table of Contents

- [Core Features](#core-features)
- [Agent Capabilities](#agent-capabilities)
- [User Interface](#user-interface)
- [Session Management](#session-management)
- [Benchmark System](#benchmark-system)
- [Telemetry & Observability](#telemetry--observability)
- [Configuration & Authentication](#configuration--authentication)
- [Advanced Features](#advanced-features)

---

## Core Features

### 1. Autonomous CTF Solving

**Overview:** PentestGPT uses an AI agent to autonomously solve Capture The Flag (CTF) challenges and perform penetration testing.

**Capabilities:**
- **Autonomous Operation** - Agent works independently without constant user input
- **Multi-Category Support** - Web, Binary Exploitation, Reverse Engineering, Crypto, Forensics, Privilege Escalation
- **Flag Detection** - Automatically identifies and extracts flags using regex patterns
- **Persistence** - Never gives up until flags are captured

**Supported Flag Formats:**
```regex
flag{...}           # Generic CTF format
FLAG{...}           # Uppercase variant
HTB{...}            # Hack The Box
CTF{...}            # Standard CTF
[A-Za-z0-9_]{...}   # Custom prefixes
[a-f0-9]{32}        # 32-char hex (HTB user/root flags)
```

**Example Usage:**
```bash
# Interactive TUI mode
pentestgpt --target 10.10.11.234

# With challenge context
pentestgpt --target example.com --instruction "WordPress site, focus on plugin vulns"

# Non-interactive mode (for automation)
pentestgpt --target 10.10.11.100 --non-interactive
```

---

### 2. Intelligent Methodology

**Systematic Approach:**

The agent follows a proven CTF methodology:

1. **Challenge Analysis** - Understand challenge type, category, available information
2. **Reconnaissance** - Enumerate target (ports, services, directories, files)
3. **Vulnerability Discovery** - Identify exploitable weaknesses
4. **Exploitation** - Execute attacks to gain access
5. **Flag Extraction** - Locate and capture flags
6. **Documentation** - Generate walkthrough as it works

**Key Principles:**
- Move quickly but systematically
- Try obvious things first (low-hanging fruit)
- Chain vulnerabilities (one finding leads to another)
- Be creative and try unconventional approaches
- Never give up - complexity is not a reason to stop

**Common Attack Vectors:**

| Category | Techniques |
|----------|------------|
| **Web Exploitation** | SQLi, XSS, SSRF, LFI/RFI, Auth bypass, Command injection |
| **Binary Exploitation** | Buffer overflows, ROP chains, Format strings, Heap exploitation |
| **Reverse Engineering** | Binary analysis, Decompilation, Debugging, Unpacking |
| **Cryptography** | Cipher breaking, Hash cracking, Weak crypto analysis |
| **Forensics** | File analysis, Steganography, Memory dumps, PCAP analysis |
| **Privilege Escalation** | SUID binaries, Kernel exploits, Sudo abuse, Misconfigurations |

---

### 3. Fallback Strategies

**When Stuck:** The agent has built-in fallback strategies:

**Reverse Shell Not Working?**
- Try different shells: bash, sh, python, php, perl, nc, socat
- Try different encodings: URL encode, base64, hex
- Try different ports: 80, 443, 8080, 4444, 1234
- Try bind shell instead of reverse shell
- Check firewall rules

**Privilege Escalation Stuck?**
- Run enumeration scripts: linpeas.sh, winPEAS
- Check SUID binaries: `find / -perm -4000`
- Check sudo rights: `sudo -l`
- Check capabilities: `getcap -r /`
- Check cron jobs, writable files, kernel version
- Look for credentials in configs, history files

**Enumeration Complete But No Flags?**
- Re-enumerate with aggressive settings
- Check non-standard ports
- Look for hidden subdirectories (`../../../`)
- Check source code line by line
- Fuzz parameters
- Look for race conditions, second-order vulns

---

## Agent Capabilities

### 1. Real-Time Tool Execution

**Built-in Tools:**
- **Bash** - Execute shell commands (nmap, curl, netcat, etc.)
- **File Operations** - Read files, analyze source code, search for secrets
- **Network Tools** - Port scanning, service enumeration, packet capture
- **Web Tools** - Directory fuzzing, parameter testing, API enumeration
- **Exploitation** - Payload generation, reverse shells, privilege escalation

**Tool Integration:**
- Tools execute in real-time
- Results streamed to UI
- Activity tracked and logged
- Failures trigger fallback strategies

---

### 2. Flag Detection System

**Automatic Flag Recognition:**

The agent continuously scans outputs for flags using pattern matching.

**Detection Patterns:**
```python
FLAG_PATTERNS = [
    r"flag\{[^\}]+\}",           # flag{...}
    r"FLAG\{[^\}]+\}",           # FLAG{...}
    r"HTB\{[^\}]+\}",            # HTB{...}
    r"CTF\{[^\}]+\}",            # CTF{...}
    r"[A-Za-z0-9_]+\{[^\}]+\}",  # Generic CTF format
    r"\b[a-f0-9]{32}\b",         # 32-char hex (HTB flags)
]
```

**When Flag Found:**
1. Event emitted to UI
2. Flag saved to session
3. Context captured (where/how found)
4. Walkthrough updated
5. Session marked as successful

**Common Flag Locations:**
- File contents (user.txt, root.txt)
- Command outputs
- Database contents
- Environment variables
- Source code comments
- Cookies, JWT tokens, API responses
- Encoded strings (base64, hex, rot13)

---

### 3. Claude Code Integration

**Backend:** Uses Claude Code SDK for powerful AI reasoning

**Features:**
- **Advanced Reasoning** - Multi-step problem solving
- **Tool Use** - Can execute bash commands, read files, search code
- **Context Management** - Maintains conversation history
- **Error Recovery** - Learns from failures and adjusts approach

**Configuration:**
```python
# Working directory
working_directory = "/workspace"

# Permission mode (bypass for autonomous operation)
permission_mode = "bypassPermissions"

# Model selection
model = "claude-sonnet-4-5"  # or custom model
```

---

## User Interface

### 1. Terminal User Interface (TUI)

**Framework:** Built with [Textual](https://textual.textualize.io/) - modern Python TUI framework

**Features:**
- **Real-time Activity Feed** - Scrollable view of agent actions
- **State Indicators** - Visual status (IDLE, RUNNING, PAUSED, COMPLETED, ERROR)
- **Tool Execution Display** - See commands being executed
- **Flag Highlights** - Flags displayed prominently when found
- **Keyboard Controls** - Full keyboard navigation

**TUI Components:**

| Component | Purpose |
|-----------|---------|
| **ActivityFeed** | Real-time display of agent messages and tool execution |
| **SplashScreen** | Animated startup screen with branding |
| **HelpScreen** | Modal help dialog with keyboard shortcuts |
| **QuitScreen** | Confirmation dialog on exit |
| **StatusBar** | Current state, target, model information |

**Visual Design:**
- Color-coded messages (success=green, error=red, info=cyan, warning=yellow)
- Tool execution boxes with command/output
- Flag detection with highlighted borders
- Responsive layout with CSS-based styling

---

### 2. Keyboard Controls

**Interactive Mode:**

| Key | Action |
|-----|--------|
| `F1` | Show help screen |
| `Ctrl+P` | Pause/Resume agent |
| `Ctrl+Q` | Quit (with confirmation) |
| `Ctrl+C` | Quit (with confirmation) |
| `↑/↓` | Scroll activity feed |
| `PgUp/PgDn` | Fast scroll |
| `Enter` | Send instruction (when paused) |

**Pause/Resume Workflow:**
1. Press `Ctrl+P` to pause agent
2. Agent completes current action, then pauses
3. Input field appears for instructions
4. Type instruction and press Enter
5. Agent resumes with new context

---

### 3. Output Modes

**TUI Mode (Default):**
```bash
pentestgpt --target 10.10.11.234
```
- Full interactive interface
- Real-time activity feed
- Keyboard controls
- Visual state indicators

**Raw Mode:**
```bash
pentestgpt --target 10.10.11.234 --raw
```
- No TUI, streaming output only
- Useful for debugging
- Can pipe to files
- Shows all agent communication

**Non-Interactive Mode:**
```bash
pentestgpt --target 10.10.11.234 --non-interactive
```
- Runs to completion without user input
- Suitable for automation/scripting
- Logs all activity to debug log

---

## Session Management

### 1. Session Persistence

**Overview:** All sessions are automatically saved to disk for resumption.

**Storage Location:** `~/.pentestgpt/sessions/<session_id>.json`

**Session Data:**
```json
{
  "session_id": "abc123def456",
  "target": "10.10.11.234",
  "created_at": "2025-01-15T10:30:00",
  "status": "completed",
  "backend_session_id": "claude_session_xyz",
  "updated_at": "2025-01-15T11:45:00",
  "task": "CTF challenge solving",
  "user_instructions": ["focus on web vulnerabilities"],
  "flags_found": [
    {"flag": "flag{example}", "context": "user.txt"}
  ],
  "total_cost_usd": 0.42,
  "model": "claude-sonnet-4-5",
  "last_error": null
}
```

**Session States:**
- `RUNNING` - Agent actively working
- `PAUSED` - Agent paused, waiting for input
- `COMPLETED` - Challenge solved, flags captured
- `ERROR` - Fatal error occurred

---

### 2. Session Resume

**Resume Previous Session:**
```bash
pentestgpt --resume abc123def456
```

**How It Works:**
1. Loads session from `~/.pentestgpt/sessions/abc123def456.json`
2. Restores target, model, instructions
3. Reconnects to backend session (if supported)
4. Continues from last state

**Use Cases:**
- Continue after interruption
- Review previous attempts
- Learn from successful walkthroughs
- Cost tracking across sessions

---

### 3. Session Listing

**View All Sessions:**
```bash
ls -lh ~/.pentestgpt/sessions/
```

**Session Naming:**
- Format: `<session_id>.json`
- Session ID: Auto-generated UUID
- Human-readable metadata inside JSON

---

## Benchmark System

### 1. Benchmark Management

**Overview:** Built-in benchmark system with 100+ vulnerability challenges.

**CLI Tool:** `pentestgpt-benchmark`

**Commands:**

```bash
# List all benchmarks
pentestgpt-benchmark list

# Show with tags
pentestgpt-benchmark list --show-tags

# Filter by difficulty
pentestgpt-benchmark list --levels 1        # Easy
pentestgpt-benchmark list --levels 2        # Medium
pentestgpt-benchmark list --levels 3        # Hard

# Filter by vulnerability type
pentestgpt-benchmark list --tags sqli
pentestgpt-benchmark list --tags xss
pentestgpt-benchmark list --tags idor,ssrf

# Start a benchmark
pentestgpt-benchmark start XBEN-001-24

# Check running benchmarks
pentestgpt-benchmark status

# Stop a benchmark
pentestgpt-benchmark stop XBEN-001-24
```

---

### 2. Benchmark Structure

**Location:** `benchmark/xbow-validation-benchmarks/`

**Individual Benchmark:**
```
XBEN-001-24/
├── benchmark.json          # Metadata
├── docker-compose.yml      # Container definition
├── src/                    # Challenge source code
│   ├── app.py
│   └── templates/
├── flag.txt                # Solution flag
└── README.md               # Challenge description
```

**Metadata (`benchmark.json`):**
```json
{
  "id": "XBEN-001-24",
  "title": "SQL Injection Challenge",
  "difficulty": 1,
  "tags": ["sqli", "web"],
  "port": 8000,
  "description": "Basic SQL injection vulnerability"
}
```

---

### 3. Vulnerability Categories

**Available Tags:**
- `sqli` - SQL Injection
- `xss` - Cross-Site Scripting
- `idor` - Insecure Direct Object Reference
- `ssti` - Server-Side Template Injection
- `ssrf` - Server-Side Request Forgery
- `lfi` - Local File Inclusion
- `rce` - Remote Code Execution
- `auth-bypass` - Authentication Bypass
- `privilege-escalation` - Privilege Escalation

**Difficulty Levels:**
- **Level 1** - Easy (beginner-friendly)
- **Level 2** - Medium (intermediate)
- **Level 3** - Hard (advanced)

---

### 4. Running Benchmarks

**Workflow:**

1. **Start Benchmark:**
```bash
pentestgpt-benchmark start XBEN-037-24
```
Output:
```
Starting benchmark XBEN-037-24...
Container started on http://0.0.0.0:8000
Target: http://0.0.0.0:8000
```

2. **Connect to Container:**
```bash
make connect  # or docker attach pentestgpt
```

3. **Run PentestGPT:**
```bash
pentestgpt --target http://host.docker.internal:8000
```

4. **Stop Benchmark:**
```bash
pentestgpt-benchmark stop XBEN-037-24
```

**Note:** Use `host.docker.internal` to access benchmark from within Docker container.

---

## Telemetry & Observability

### 1. Activity Tracking

**Tracer System:**

PentestGPT includes a built-in activity tracer for debugging and monitoring.

**Tracked Activities:**
- Agent messages (info, success, error, warning)
- Tool executions (start, complete, result)
- State changes (idle, running, paused, completed, error)
- Flag discoveries

**Usage:**
```python
from pentestgpt.core.tracer import get_global_tracer

tracer = get_global_tracer()
tracer.track_message("Starting enumeration", "info")
tracer.track_tool_start("nmap", {"target": "10.10.11.234"})
```

**Integration:**
- TUI subscribes to tracer callbacks
- Real-time updates to activity feed
- Debug logs written to `/workspace/pentestgpt-debug.log`

---

### 2. Langfuse Telemetry

**Overview:** Optional telemetry integration using [Langfuse](https://langfuse.com) for improving PentestGPT.

**Enabled by Default** - Help improve the tool by sharing anonymous usage data.

**Data Collected:**
- Session metadata (target type, duration, completion status)
- Tool execution patterns (which tools used, not actual commands)
- Flag detection events (that a flag was found, not the flag content)
- Model usage and performance metrics

**Data NOT Collected:**
- Command outputs
- Credentials or sensitive information
- Actual flag values
- File contents
- Network traffic

**Opt-Out:**

```bash
# Via command line flag
pentestgpt --target 10.10.11.234 --no-telemetry

# Via environment variable
export LANGFUSE_ENABLED=false
pentestgpt --target 10.10.11.234
```

**User Identification:**
- Persistent UUID stored in `~/.pentestgpt/user_id`
- Allows tracking usage patterns per user
- Anonymous - no personal information collected

---

### 3. Debug Logging

**Log Locations:**
- Primary: `/workspace/pentestgpt-debug.log` (Docker)
- Fallback: `/tmp/pentestgpt-debug.log`

**Enable Debug Mode:**
```bash
pentestgpt --target 10.10.11.234 --debug
```

**Log Contents:**
- Detailed agent execution flow
- LLM API requests/responses
- Tool execution details
- Error stack traces
- Event bus activity

**Log Format:**
```
2025-01-15 10:30:15 [INFO] pentestgpt.core.agent: Starting reconnaissance
2025-01-15 10:30:16 [DEBUG] pentestgpt.core.agent: Executing: nmap -sV 10.10.11.234
2025-01-15 10:30:20 [INFO] pentestgpt.core.agent: Found open ports: 22, 80
```

---

## Configuration & Authentication

### 1. Authentication Options

**Configure via:** `make config`

**Available Options:**

1. **Claude Login (OAuth)**
   - Recommended for Claude subscribers
   - Browser-based authentication
   - No API key required

2. **OpenRouter**
   - Access multiple LLM providers
   - Get key from [openrouter.ai](https://openrouter.ai/keys)
   - Cost-effective alternative

3. **Anthropic API Key**
   - Direct API access
   - Get key from [console.anthropic.com](https://console.anthropic.com/)
   - Pay-per-use pricing

4. **Local LLM**
   - Route to local LLM server (LM Studio, Ollama)
   - No API costs
   - Requires local setup

**Configuration Storage:**
- Docker: Persisted in `claude-config` volume
- Local: `~/.config/claude/` (depends on backend)
- Auth file: `.env.auth` (generated by `make config`)

---

### 2. Local LLM Setup

**Requirements:**
- Local LLM server with OpenAI-compatible API
- Examples: LM Studio, Ollama, text-generation-webui

**Configuration:**

1. **Start LLM Server:**
```bash
# LM Studio: Enable server mode (port 1234)
# Ollama: ollama serve (port 11434)
```

2. **Configure PentestGPT:**
```bash
make config  # Select option 4: Local LLM
```

3. **Customize Routing:**

Edit `scripts/ccr-config-template.json`:
```json
{
  "localLLM": {
    "api_base_url": "http://host.docker.internal:1234",
    "models": ["openai/gpt-oss-20b", "qwen/qwen3-coder-30b"]
  },
  "router": {
    "default": "openai/gpt-oss-20b",
    "background": "openai/gpt-oss-20b",
    "think": "qwen/qwen3-coder-30b",
    "longContext": "qwen/qwen3-coder-30b"
  }
}
```

**Note:** Use `host.docker.internal` to access host services from Docker.

---

### 3. Environment Configuration

**Configuration File:** `.env` (optional)

**Example:**
```bash
# Working directory
PENTESTGPT_WORKING_DIRECTORY=/workspace

# Model selection
PENTESTGPT_MODEL=claude-sonnet-4-5

# Permission mode
PENTESTGPT_PERMISSION_MODE=bypassPermissions

# Telemetry
LANGFUSE_ENABLED=true
```

**Pydantic Settings:**
- Automatic loading from `.env`
- Environment variables override file
- Type validation
- Default values

---

## Advanced Features

### 1. Custom Instructions

**Provide Context to Agent:**

```bash
pentestgpt --target example.com \
  --instruction "WordPress site version 5.8, focus on plugin vulnerabilities"
```

**Use Cases:**
- Hint at vulnerability type
- Provide credentials
- Specify constraints
- Focus enumeration

**Mid-Session Instructions:**
1. Pause agent with `Ctrl+P`
2. Type additional instruction
3. Press Enter to resume

---

### 2. Multi-Stage Exploitation

**Agent Capabilities:**
- Chain multiple vulnerabilities
- Maintain shell persistence
- Pivot through network
- Escalate privileges systematically

**Example Flow:**
1. Initial recon (nmap)
2. Web enumeration (gobuster)
3. Exploit SQLi to get creds
4. SSH access with creds
5. Privilege escalation (SUID binary)
6. Capture flags

---

### 3. Tool Extensibility

**Add Custom Tools:**

```python
# pentestgpt/tools/custom.py
from pentestgpt.tools.base import BaseTool

class SQLMapTool(BaseTool):
    def __init__(self):
        super().__init__(
            name="sqlmap",
            description="Automated SQL injection tool"
        )

    async def execute(self, url: str, **kwargs):
        # Implementation
        return {
            "success": True,
            "result": "SQL injection found"
        }

# Register tool
from pentestgpt.tools.registry import get_registry
get_registry().register(SQLMapTool())
```

**Tool Features:**
- Async execution
- Error handling
- Result formatting
- EventBus integration

---

### 4. State Management

**5-State Lifecycle:**

```
IDLE → RUNNING → PAUSED → COMPLETED
             ↓
           ERROR
```

**State Transitions:**
- `IDLE → RUNNING` - Agent starts
- `RUNNING → PAUSED` - User pauses or agent requests input
- `PAUSED → RUNNING` - User resumes
- `RUNNING → COMPLETED` - Flags captured successfully
- `RUNNING → ERROR` - Fatal error occurs

**Pause/Resume:**
- Pause at message boundaries (clean state)
- Resume maintains full context
- No loss of progress

---

## Next Steps

- **[PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md)** - Project organization
- **[ARCHITECTURE.md](./ARCHITECTURE.md)** - Design patterns and data flow
- **[CODE_GUIDE.md](./CODE_GUIDE.md)** - Code walkthroughs and explanations

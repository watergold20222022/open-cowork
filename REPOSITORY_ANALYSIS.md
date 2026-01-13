# Kimi CLI Repository - Deep Analysis & Understanding

## Executive Summary

**Kimi CLI** is an AI-powered software engineering agent that runs in the terminal. It's built by Moonshot AI (creators of the Kimi chatbot) and designed as a local-first, Python-based agentic system that can read/write code, execute shell commands, search the web, and autonomously plan multi-step workflows.

**Current Status**: Technical Preview (v0.76 as of Jan 2026)  
**Tech Stack**: Python 3.12+ (tooling for 3.14), kosong (LLM framework), fastmcp (MCP integration), prompt-toolkit (TUI)

---

## 1. Project Architecture Overview

### 1.1 Core Philosophy

Kimi CLI implements an **agentic loop** architecture where:
- The **Soul** (`KimiSoul`) is the autonomous decision-making core that orchestrates LLM interactions and tool execution
- **Tools** are the agent's hands (file operations, shell commands, web fetch, multiagent spawning)
- **Context** is the conversation memory with checkpointing support for long-running workflows
- **Wire** is the event bus connecting the soul to various UI frontends (shell, ACP, print, wire-over-stdio)

### 1.2 Key Components

```
┌──────────────────────────────────────────────────────────────┐
│                         CLI Entry                            │
│                    (src/kimi_cli/cli.py)                     │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                      KimiCLI (app.py)                        │
│  • Load config, select LLM provider/model                    │
│  • Create Runtime (session, builtins, LLM instance)          │
│  • Load Agent spec (YAML) with system prompt & tools         │
│  • Restore Context from session file                         │
│  • Build KimiSoul instance                                   │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                   KimiSoul (kimisoul.py)                     │
│  Core agent loop:                                            │
│  1. Accept user input (text/slash commands)                  │
│  2. Append to context                                        │
│  3. Call LLM (via kosong)                                    │
│  4. Execute tool calls (via KimiToolset)                     │
│  5. Handle approvals (via Runtime.approval)                  │
│  6. Perform compaction when context limit approaches         │
│  7. Emit Wire events for UI consumption                      │
└─────────────────────────┬────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌─────────┐    ┌──────────┐    ┌──────────┐
    │  Shell  │    │   ACP    │    │  Print   │
    │   UI    │    │  Server  │    │   UI     │
    └─────────┘    └──────────┘    └──────────┘
```

---

## 2. Major Subsystems

### 2.1 Agent Specification System (`agentspec.py`)

Agents are defined in YAML files under `src/kimi_cli/agents/`:
- **`default/`**: The base agent shipped with Kimi CLI
- **`okabe/`**: Alternative agent with DMail tool (time-travel debugging via checkpoints)

**Agent Spec Structure:**
```yaml
version: 1
agent:
  extend: base_agent.yaml  # Inheritance support
  name: "Agent Name"
  system_prompt_path: ./system.md
  system_prompt_args:
    CUSTOM_ARG: "value"
  tools:
    - "kimi_cli.tools.file:ReadFile"
    - "kimi_cli.tools.shell:Shell"
  exclude_tools:
    - "kimi_cli.tools.multiagent:Task"
  subagents:
    coder:
      path: ./coder.yaml
      description: "Coding specialist"
```

**System Prompt Injection:**  
Built-in args like `KIMI_NOW`, `KIMI_WORK_DIR`, `KIMI_WORK_DIR_LS`, `KIMI_AGENTS_MD`, `KIMI_SKILLS` are automatically injected into prompts via Jinja2 templates.

### 2.2 Runtime & Session Management

**Runtime** (`src/kimi_cli/soul/agent.py`):
- Holds config, LLM instance, session, approval handler, labor market (subagent registry)
- Provides `builtin_args` for system prompt rendering
- Manages environment and skills directories

**Session** (`src/kimi_cli/session.py`):
- Each working directory gets a metadata file at `~/.kimi/sessions/{workdir_md5}/`
- Session data stored as `{session_id}/context.jsonl`
- Sessions are **local-only** (no cloud sync)
- Support for `Session.create()`, `Session.continue_()`, `Session.find()`, `Session.list()`

**Context** (`src/kimi_cli/soul/context.py`):
- JSONL-based conversation history
- Checkpointing support for DMail tool (time-travel replies)
- Compaction triggers when approaching token limits

### 2.3 Tooling Architecture

**KimiToolset** (`src/kimi_cli/soul/toolset.py`):
- Loads tools by import path (e.g., `kimi_cli.tools.file:ReadFile`)
- Dependency injection: tools can request `Runtime`, `Session`, `Approval`, `Config`, etc.
- MCP tool integration via `fastmcp`

**Built-in Tools** (`src/kimi_cli/tools/`):
- **File**: `ReadFile`, `WriteFile`, `StrReplaceFile`, `ListDirectory`, `Glob`, `Grep`
- **Shell**: `Shell` (bash/PowerShell executor)
- **Web**: `FetchURL` (web scraping via trafilatura)
- **Todo**: `SetTodoList` (task tracking)
- **Multiagent**: `Task` (spawn subagents)
- **DMail**: `SendDMail` (checkpoint-based time travel)
- **Think**: `Think` (explicit reasoning tool)

**Approval System** (`src/kimi_cli/soul/approval.py`):
- Mediates user approvals for tool actions (especially file writes)
- Supports "approve for session" mode
- YOLO mode bypasses all approvals

### 2.4 LLM Integration (`llm.py`)

**Supported Providers** (via kosong):
- `kimi`: Moonshot AI (Kimi Code API)
- `anthropic`: Claude
- `openai`: OpenAI + compatible APIs
- `gemini`: Google Gemini Developer API
- `vertexai`: Vertex AI

**Model Capabilities** (tracked per model):
- `vision`, `video_in`, `image_generation`, `thinking`, `always_thinking`

**Thinking Mode**:
- Toggleable via config or CLI flag
- Models like Claude can expose internal reasoning before tool calls

### 2.5 Subagent System (`LaborMarket`)

**Fixed Subagents** (defined in agent spec):
- Pre-configured specialists (e.g., `coder`, `reviewer`)
- Isolated context to avoid polluting main agent history

**Dynamic Subagents** (created via `CreateSubagent` tool):
- Runtime agent spawning
- Each gets a fresh context and works in parallel

**Execution Model**:
- Subagents run in separate contexts (stored as `{main_context}_sub_N.jsonl`)
- Results returned to main agent when complete
- Used for parallel task decomposition

### 2.6 UI Frontends

**Shell UI** (`src/kimi_cli/ui/shell/`):
- Default interactive TUI (prompt-toolkit)
- **Shell mode toggle** (`Ctrl-X`): switch between agent mode and direct shell execution
- Slash command autocomplete
- Status bar showing current model, YOLO mode indicator

**ACP Server** (`src/kimi_cli/acp/`):
- Agent Client Protocol integration for IDE usage (Zed, JetBrains)
- Command: `kimi acp`
- Exposes slash commands, file diffs, and terminal output to clients
- Uses ACPKaos for editor-buffer-aware file operations

**Print UI** (`src/kimi_cli/ui/print.py`):
- Non-interactive mode for scripting
- `--final-message-only` / `--quiet`: only output final result

**Wire UI** (`src/kimi_cli/ui/wire/`):
- Stdio-based JSON protocol for programmatic integration
- Command: `kimi --wire`

### 2.7 MCP (Model Context Protocol) Support

**MCP Server Management**:
```bash
# Add HTTP MCP server
kimi mcp add --transport http context7 https://mcp.context7.com/mcp \
  --header "CONTEXT7_API_KEY: ctx7sk-..."

# Add stdio MCP server
kimi mcp add --transport stdio chrome-devtools -- npx chrome-devtools-mcp@latest

# OAuth-based MCP server
kimi mcp add --transport http --auth oauth linear https://mcp.linear.app/mcp
```

**MCP Config Storage**: `~/.kimi/mcp.json`

**Ad-hoc MCP**: `kimi --mcp-config-file /path/to/mcp.json`

---

## 3. Monorepo Structure

### 3.1 Workspace Layout

This is a **uv workspace** monorepo:

```
open-cowork/
├── src/kimi_cli/          # Main Kimi CLI package
├── packages/
│   ├── kosong/            # LLM framework (chat providers, message handling)
│   └── kaos/              # OS abstraction layer (PyKAOS)
├── sdks/
│   └── kimi-sdk/          # SDK for programmatic usage
├── docs/                  # User documentation (English + Chinese)
├── examples/              # Custom soul/tool examples
├── tests/                 # Unit tests
└── tests_ai/              # Integration/AI tests
```

**Workspace Configuration** (`pyproject.toml`):
```toml
[tool.uv.workspace]
members = ["packages/kosong", "packages/kaos", "sdks/kimi-sdk"]

[tool.uv.sources]
kosong = { workspace = true }
pykaos = { workspace = true }
```

### 3.2 Workspace Packages

**kosong** (`packages/kosong/`):
- LLM provider abstraction (Anthropic, OpenAI, Kimi, Google, etc.)
- Message protocols, tool schemas, streaming
- Published to PyPI as `kosong`

**pykaos** (`packages/kaos/`):
- OS abstraction layer for file/path operations
- LocalKaos (local FS) + ACPKaos (ACP-backed FS for IDE integration)
- Published to PyPI as `pykaos`

**kimi-sdk** (`sdks/kimi-sdk/`):
- Programmatic API for embedding Kimi CLI in custom applications

### 3.3 Release Strategy

**Versioning**:
- `kimi-cli`: `0.76`, `0.77` (pure numeric)
- `kosong`: `kosong-0.37.0` (prefixed tags)
- `pykaos`: `pykaos-0.6.0` (prefixed tags)

**Release Process** (from AGENTS.md):
1. Create release branch (e.g., `bump-0.76`)
2. Update `CHANGELOG.md` (rename `[Unreleased]` → `[0.76] - YYYY-MM-DD`)
3. Update `pyproject.toml` version
4. Run `uv sync` to align lockfile
5. Merge PR, tag `git tag 0.76`, push tags
6. GitHub Actions handles publishing to PyPI

---

## 4. Key Design Patterns & Conventions

### 4.1 Wire Event System

**Purpose**: Decouple agent logic from UI rendering.

**Wire Messages** (`src/kimi_cli/wire/types.py`):
- `TurnBegin`: Marks start of agent turn
- `StatusUpdate`: Progress updates (token usage, message ID)
- `StepBegin`: Tool execution start
- `ToolResult`: Tool execution result (with display blocks for diffs)
- `ApprovalRequest`: User approval needed
- `ApprovalRequestResolved`: Approval decision
- `CompactionBegin` / `CompactionEnd`: Context compaction events
- `StepInterrupted`: Execution interruption

**Wire Transport**:
- `WireUISide` / `WireSoulSide`: Async queue-based communication
- JSON serialization for wire-over-stdio mode

### 4.2 Slash Commands

**Two-Level Registry**:
- **Soul-level** (`src/kimi_cli/soul/slash.py`): `/init`, `/compact`, `/yolo`, `/usage`, `/skill:<name>`
- **Shell-level** (`src/kimi_cli/ui/shell/slash.py`): `/help`, `/exit`, `/version`, `/changelog`, `/sessions`, `/model`, `/mcp`

**Skill Loading**:
- Skills in `~/.kimi/skills/` or `~/.claude/skills/`
- `/skill:<skill-name>` loads `SKILL.md` as a user prompt
- Built-in skill: `skill-creator` (meta-skill for generating new skills)

### 4.3 Compaction Strategy

**Simple Compaction** (`src/kimi_cli/soul/compaction.py`):
- Triggered when context approaches `max_context_size - RESERVED_TOKENS`
- Summarizes old messages into a condensed system message
- Preserves recent history for continuity

**Checkpoint-based Compaction**:
- DMail tool creates checkpoints
- Allows "time-travel" replies from specific conversation states

### 4.4 KAOS Abstraction

**Purpose**: Abstract OS operations for ACP integration.

**LocalKaos** (default):
- Direct filesystem access
- Used when running standalone in terminal

**ACPKaos** (ACP mode):
- Redirects `read_text`, `write_text`, `exec` to ACP client
- Allows IDE to observe file edits and terminal output
- Falls back to LocalKaos for unsupported operations

---

## 5. Development Workflow

### 5.1 Common Commands

```bash
# Setup development environment
make prepare         # Install deps + git hooks (prek)

# Code quality
make format          # Run ruff format
make check           # Run ruff lint + pyright + ty
make test            # Run pytest
make ai-test         # Run AI integration tests

# Build
make build           # Build wheel
make build-bin       # Build PyInstaller binary
```

### 5.2 Git Hooks (prek)

- Auto-format on commit
- Type checks on pre-push
- Workspace-aware: only runs hooks for changed packages

### 5.3 Commit Message Convention

**Format**: `<type>(<scope>): <subject>`

**Types**: `feat`, `fix`, `test`, `refactor`, `chore`, `style`, `docs`, `perf`, `build`, `ci`, `revert`

**Example**: `feat(acp): add file diff preview in approval prompts`

---

## 6. Integration Points

### 6.1 IDE Integration (ACP)

**Zed Configuration** (`~/.config/zed/settings.json`):
```json
{
  "agent_servers": {
    "Kimi CLI": {
      "command": "kimi",
      "args": ["acp"],
      "env": {}
    }
  }
}
```

**Features**:
- File diffs in approval prompts
- Terminal output routed to IDE
- Slash command autocomplete
- Model switching
- MCP server delegation to IDE

### 6.2 Zsh Integration

**Plugin**: `zsh-kimi-cli` (external repo)
- `Ctrl-X` to toggle agent mode in Zsh
- Seamless shell ↔ agent switching

### 6.3 Programmatic Usage (SDK)

```python
from kimi_cli.app import KimiCLI
from kimi_cli.session import Session
from kaos.path import KaosPath

session = await Session.create(KaosPath.cwd())
instance = await KimiCLI.create(session)
await instance.run_print("Analyze this codebase")
```

---

## 7. Technology Stack Deep Dive

### 7.1 Core Dependencies

- **typer**: CLI framework
- **prompt-toolkit**: Interactive TUI (shell mode)
- **kosong**: LLM provider abstraction (workspace package)
- **pykaos**: OS abstraction (workspace package)
- **fastmcp**: MCP protocol implementation
- **aiohttp** / **httpx**: Async HTTP clients
- **trafilatura**: Web content extraction
- **ripgrepy**: Ripgrep Python bindings for fast grep
- **loguru**: Structured logging
- **pydantic**: Data validation
- **jinja2**: Template rendering (system prompts)

### 7.2 Build & Packaging

- **uv**: Package manager + workspace orchestrator
- **uv_build**: Build backend
- **PyInstaller**: Binary distribution (standalone executables)

### 7.3 Testing

- **pytest** + **pytest-asyncio**: Test runner
- **inline-snapshot**: Snapshot testing for outputs
- **ruff**: Linting + formatting
- **pyright** + **ty**: Type checking

---

## 8. Special Features & Workflows

### 8.1 Ralph Loop (Automated Iteration)

**Purpose**: Continuously run agent until no more tool calls.

**Mechanism**:
- Agent loops until it outputs text without tool calls
- `RALPH_SAFEWORD` (`<safeword>STOP</safeword>`) breaks the loop
- Configurable via `max_ralph_iterations` in loop control config

**Use Cases**:
- Automated refactoring workflows
- Multi-step code generation
- Unattended batch processing

### 8.2 DMail System (Time Travel)

**Concept**: Checkpoint conversation state for "time-travel" replies.

**Implementation**:
- `SendDMail` tool creates checkpoint in context
- Allows agent to "reply" from a past state
- Useful for exploring alternative solution paths

### 8.3 Skills System

**Structure**:
```
~/.kimi/skills/my-skill/
├── SKILL.md          # Instructions loaded on demand
├── scripts/          # Reusable scripts
├── references/       # Domain documentation
└── assets/           # Templates, examples
```

**Usage**: `/skill:my-skill` loads `SKILL.md` as context

**Built-in Skill**: `skill-creator` (generates new skills via LLM)

---

## 9. Configuration Management

### 9.1 Config File (`~/.kimi/config.toml`)

```toml
default_model = "kimi-k1.5"
default_thinking = false

[providers.kimi]
type = "kimi"
base_url = "https://api.moonshot.cn/v1"
api_key = "sk-..."

[models.kimi-k1.5]
provider = "kimi"
model = "moonshot-v1-auto"
max_context_size = 128000

[loop_control]
max_steps_per_turn = 50
max_retries_per_step = 5
max_ralph_iterations = 10
```

### 9.2 Environment Variable Overrides

**Priority**: CLI args > env vars > config file

**Supported Env Vars**:
- `KIMI_API_KEY`, `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, etc.
- Auto-detected by provider augmentation logic

---

## 10. Observability & Debugging

### 10.1 Logging

**Location**: `~/.kimi/logs/kimi.log`  
**Rotation**: Daily at 06:00, retained for 10 days  
**Levels**: TRACE (debug mode) / INFO (normal)

### 10.2 Session Replay

**Context Files**: `~/.kimi/sessions/{workdir_md5}/{session_id}/context.jsonl`  
**Replay**: Load session, inspect conversation history, continue execution

### 10.3 Wire Message Tracing

**Wire JSONL**: `~/.kimi/sessions/{session_id}/wire.jsonl`  
**Contents**: All Wire events for debugging UI behavior

---

## 11. Security & Safety

### 11.1 Approval System

- File writes require user approval (unless YOLO mode)
- Shell commands require approval
- "Approve for session" mode: remember approvals per session

### 11.2 Sandbox Constraints

- No built-in sandboxing (agent runs with user privileges)
- KAOS abstraction allows future sandboxing layers
- MCP tools run in separate processes

### 11.3 Sensitive Data Handling

- API keys stored in config with `SecretStr` (Pydantic)
- Logs redact sensitive fields
- No telemetry/analytics

---

## 12. Comparison to Claude Cowork

| Feature | Kimi CLI | Claude Cowork |
|---------|----------|---------------|
| **Platform** | Cross-platform (macOS/Linux/Windows) | macOS only (Research Preview) |
| **Pricing** | Free + bring-your-own-LLM | Claude Max ($100-200/month) |
| **Architecture** | Terminal-based agent | Desktop app with folder access |
| **Sub-agents** | Full support (fixed + dynamic) | Independent sub-agents with fresh context |
| **IDE Integration** | ACP (Zed, JetBrains) | None (standalone) |
| **MCP Support** | Full MCP client support | Unknown |
| **Session Sync** | Local-only (no sync) | Local-only (no sync) |
| **Thinking Mode** | Multi-provider support | Claude-specific |
| **Skills** | Custom skills via `SKILL.md` | No equivalent |
| **Open Source** | Yes (GitHub: MoonshotAI/kimi-cli) | No (proprietary) |

---

## 13. Future Directions (from KLIPs)

### KLIP-3: User Documentation

- Migrate docs to VitePress
- Multi-language support (EN/ZH)
- Interactive tutorials

### KLIP-6: Auto-refresh Models

- Periodically fetch provider model lists
- Auto-detect new models (e.g., `gpt-4.5-turbo`)

### KLIP-7: Kimi SDK

- Standalone SDK for embedding Kimi CLI
- Simplified API for library usage

---

## 14. Key Learnings & Design Insights

### 14.1 Why "Soul" Abstraction?

- Decouples agent logic from UI/transport
- Enables multiple frontends (shell, ACP, wire, print)
- Testable without UI dependencies

### 14.2 Why Workspace Monorepo?

- Tight integration between `kimi-cli`, `kosong`, `pykaos`
- Simultaneous cross-package development
- Independent release cycles (different semver ranges)

### 14.3 Why KAOS?

- Abstract OS operations for cross-platform + ACP support
- Future-proof for sandboxing/security layers
- Allows ACPKaos to redirect file ops to IDE without tool changes

### 14.4 Why MCP Integration?

- Leverage external tooling (browsers, databases, APIs)
- Avoid reinventing every tool
- Community ecosystem benefits

---

## 15. Getting Started (Quick Reference)

### 15.1 Installation

```bash
# Via uv (recommended)
uv tool install kimi-cli

# Via pipx
pipx install kimi-cli

# From source
git clone https://github.com/MoonshotAI/kimi-cli.git
cd kimi-cli
make prepare
uv run kimi
```

### 15.2 First-Time Setup

```bash
kimi  # Launch shell
/setup  # Configure API keys
/model  # Select default model
```

### 15.3 Common Workflows

```bash
# Start agent session
kimi

# Non-interactive (scripting)
kimi print "Generate a README for this project"

# IDE integration (Zed/JetBrains)
kimi acp

# Resume session
kimi --session <session-id>

# YOLO mode (no approvals)
kimi --yolo
```

---

## 16. Troubleshooting

### Issue: "LLM provider not set"
**Solution**: Run `/setup` or set `KIMI_API_KEY` / `ANTHROPIC_API_KEY` env var.

### Issue: "MCP server connection failed"
**Solution**: Check `kimi mcp list`, retry `kimi mcp auth <server>`.

### Issue: "Context file corrupted"
**Solution**: Delete `~/.kimi/sessions/{workdir_md5}/{session_id}/context.jsonl`, start fresh.

### Issue: Shell mode not working
**Solution**: Ensure `Ctrl-X` binding isn't conflicting with terminal emulator.

---

## Conclusion

Kimi CLI is a **production-grade agentic framework** with:
- **Modular architecture** (Soul/Runtime/Toolset/Wire)
- **Multi-provider LLM support** (Kimi, Claude, GPT, Gemini)
- **Rich tooling ecosystem** (file ops, shell, web, MCP)
- **IDE integration** (ACP for Zed/JetBrains)
- **Advanced features** (subagents, skills, time-travel, compaction)

It represents a **local-first alternative to Claude Cowork**, with superior extensibility, cross-platform support, and open-source ethos. The monorepo structure and KAOS abstraction position it well for future enhancements (sandboxing, cloud sync, enterprise features).

**Target Audience**: Developers who want AI coding assistance that:
1. Runs locally with full control
2. Integrates with existing workflows (terminal, IDE, Zsh)
3. Supports custom tools/skills
4. Works with multiple LLM providers

**Differentiators vs. Claude Cowork**:
- ✅ Cross-platform (not macOS-only)
- ✅ Free/BYOLLM (not $100-200/month)
- ✅ Open source (not proprietary)
- ✅ IDE integration (Claude Cowork has none)
- ✅ MCP ecosystem (unknown in Claude Cowork)

---

*Document created: 2026-01-13*  
*Repository version: 0.76*  
*Analysis based on codebase snapshot and official documentation*

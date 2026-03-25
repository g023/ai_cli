# Just Intelligent CLI (AI Powered CLI AGENT)

## License

MIT Licensed

Author: g023 (github.com/g023)

url: https://github.com/g023/ai_cli


An offline first focused production-ready agentic CLI application powered by local *Ollama* models. Just Intelligent CLI acts as a personal AI agent that can control your computer, build programs, create documents, and manage projects — all from your terminal. This is a work-in-progress so a good bet would be to run it in a container and isolate from your main environment, but I'm not your mom. **DEFAULT USES A 4b Qwen 3.5 MODEL**.

*Database will be created on run if it doesn't exist*

**Agents are unpredictable so use with caution.**

## Features

- **Agentic Tool Use** — The AI autonomously calls tools (shell commands, file operations, search) to accomplish tasks
- **Multi-Turn Conversations** — Persistent sessions with automatic context management
- **Hierarchical Context Pruning** — Smart memory management that prunes old context while preserving recent state
- **Re-Prime Strategy** — Throws away stale conversations and re-primes with structured state instead of lossy summarization
- **Task Planning** — Pre-think decomposition breaks complex tasks into steps before execution
- **Plugin Architecture** — Drop Python files in `./plugins/` to add custom tools
- **Session Persistence** — SQLite-backed session history with cross-session memory
- **Multiple Modes** — Interactive REPL, one-shot prompts, NDJSON protocol for integrations
- **Model Aliases** — Short aliases for long model names
- **Safety Checks** — Some dangerous operations are blocked before execution
- **Streaming Output** — Real-time streaming from Ollama with clean tool-call handling

## Requirements

- Python 3.6+ (uses f-strings and modern typing)
- [Ollama](https://ollama.ai) running locally (`ollama serve`)
- `requests` library (`pip install requests`)
- SQLite3 (included with Python)
- A pulled model (default: `ollama pull qwen3.5:4b-q4_K_M`)

## Quick Start

```bash
# Interactive REPL
python3 jicli.py

# One-shot prompt
python3 jicli.py -p "Create a Python script that sorts a CSV file by the second column"

# Use a specific model
python3 jicli.py -m cascade -p "Refactor this codebase to use async/await"

# List available models
python3 jicli.py --list-models
```

## Usage

```
jicli [-h] [-p PROMPT] [-m MODEL] [-t MAX_TURNS] [-v] [--ndjson]
      [--resume] [--plan] [--cwd CWD] [--db DB] [--list-models]
      [--list-sessions] [--memory] [--allowed-tools TOOLS]
      [--disallowed-tools TOOLS] [--no-think] [--version]
```

### Options

| Flag | Description |
|------|-------------|
| `-p PROMPT` | One-shot mode: run a single prompt and exit |
| `-m MODEL` | Model name or alias (default: `ji`) |
| `-t N` | Max agent turns (default: 50) |
| `-v` | Verbose output (show tool args, token counts) |
| `--plan` | Enable task planning for complex prompts |
| `--resume` | Resume the last session |
| `--ndjson` | NDJSON protocol mode for programmatic use |
| `--cwd DIR` | Set working directory |
| `--db PATH` | Custom session database path |
| `--list-models` | List available Ollama models |
| `--list-sessions` | List recent sessions |
| `--memory` | Show persistent cross-session memory |
| `--allowed-tools` | Comma-separated list of allowed tools |
| `--disallowed-tools` | Comma-separated list of blocked tools |
| `--no-think` | Disable model thinking mode |

## Model Aliases (remember to pull these models in Ollama if you want to use them)

| Alias | Model |
|-------|-------|
| `ji` | `qwen3.5:4b-q4_K_M` |
| `cascade` | `nemotron-cascade-2:30b-a3b-q4_K_M` |
| `30b` | `nemotron-cascade-2:30b-a3b-q4_K_M` |
| `qwen3.5` | `qwen3.5:9b` |

## Interactive REPL Commands

When running in interactive mode (no `-p` flag):

| Command | Description |
|---------|-------------|
| `/help` | Show available commands |
| `/exit` | Exit Just Intelligent CLI |
| `/clear` | Clear current session |
| `/new` | Start a fresh session |
| `/resume` | Resume last session |
| `/sessions` | List recent sessions |
| `/session [id]` | Show or switch to a session |
| `/model [name]` | Show or switch model |
| `/models` | List available models |
| `/tools` | List available tools |
| `/memory` | Show persistent memory |
| `/memory set k=v` | Store a memory fact |
| `/memory get k` | Recall a memory fact |
| `/memory search q` | Search memory |
| `/memory forget k` | Forget a memory fact |
| `/compact` | Prune conversation context |

## Built-in Tools

Just Intelligent CLI comes with 7 tools the AI agent can use:

| Tool | Description |
|------|-------------|
| **Bash** | Execute shell commands with safety checks |
| **Read** | Read file contents with line numbers |
| **Write** | Create or overwrite files (creates parent dirs) |
| **Edit** | Find-and-replace text in existing files |
| **Glob** | Find files by glob pattern |
| **Grep** | Search file contents by regex pattern |
| **LS** | List directory contents with file sizes |

### Safety

The Bash tool blocks dangerous commands before execution (I'm not perfect, so expect edge cases to break through, again, container might be a good idea):
- `rm -rf /`, `rm -rf /*`
- `mkfs`, `dd if=`
- `chmod -R 777 /`
- others

## Plugin System

Add custom tools by creating Python files in a `./plugins/` directory:

```python
# plugins/my_tool.py
from jicli.tools.base import Tool

class MyCustomTool(Tool):
    name = "MyTool"
    description = "Does something useful"
    parameters = {
        "type": "object",
        "properties": {
            "input": {"type": "string", "description": "The input to process"}
        },
        "required": ["input"]
    }

    def execute(self, args):
        result = do_something(args["input"])
        return self._ok(result)
```

Plugins are auto-discovered when Just Intelligent CLI starts. Any class that subclasses `Tool` and has a `name` will be registered.

## Context Management

Just Intelligent CLI uses a hierarchical context management strategy:

1. **Token Tracking** — Estimates token usage to stay within the model's context window (default: 16,384 tokens)
2. **Smart Pruning** — When context fills up, older messages are compressed while recent messages stay intact
3. **Re-Prime** — For severe context overflow, the conversation is thrown away and restarted with structured state extraction
4. **Tool Output Capping** — Large tool outputs are truncated with head+tail preservation to prevent context explosion
5. **Error Pattern Learning** — Common errors are tracked in the database and included in the system prompt to avoid repeating mistakes

### How Pruning Works

Messages are divided into three zones:
- **Head** (first message) — Usually contains important initial context, always preserved
- **Middle** (old messages) — Collapsed into a terse summary when context gets full
- **Tail** (recent messages) — Always preserved for continuation coherence

The pruner iterates: if still over threshold, it keeps fewer tail messages until reaching a minimum of 2.

## Persistent Memory

Just Intelligent CLI maintains cross-session memory in `.jicli_memory/`:

```bash
# Store a fact
python3 jicli.py
ji> /memory set project_lang=python
ji> /memory set db_type=postgresql

# Facts persist across sessions and are included in the system prompt
```

## NDJSON Protocol

For programmatic integration, use NDJSON mode:

```bash
echo '{"type":"message","content":"List files in /tmp"}' | python3 jicli.py --ndjson
```

Output is newline-delimited JSON with event types:
- `{"type": "token", "content": "..."}` — Streaming text token
- `{"type": "tool_call", "name": "...", "args": {...}}` — Tool invocation
- `{"type": "tool_result", "name": "...", "content": "..."}` — Tool result
- `{"type": "done", "session_id": "...", "turns": N, "tokens": N}` — Completion

## Task Planning

For complex tasks, enable planning with `--plan`:

```bash
python3 jicli.py --plan -p "Create a REST API with FastAPI that has CRUD endpoints for a todo list"
```

The planner:
1. Analyzes the task complexity
2. Breaks it into ordered steps
3. Includes the plan in the agent's context
4. The agent follows the plan while executing tools

## Architecture

```
jicli/
├── __init__.py          # Package init (version, app name)
├── __main__.py          # python3 -m jicli entry point
├── config.py            # Configuration (aliases, options, defaults)
├── client.py            # Ollama HTTP client (streaming, retry)
├── agent.py             # Core agent loop (LLM ↔ Tool orchestration)
├── cli.py               # CLI (argparse, REPL, one-shot, NDJSON modes)
├── tools/
│   ├── __init__.py      # ToolRegistry with auto-discovery
│   ├── base.py          # Base Tool class
│   ├── bash.py          # Shell execution with safety
│   └── filesystem.py    # Read, Write, Edit, Glob, Grep, LS
├── memory/
│   ├── __init__.py      # Exports
│   ├── session.py       # SQLite session store (WAL mode)
│   ├── context.py       # Context window manager (pruning, re-prime)
│   └── persistent.py    # File-backed cross-session memory
├── planner/
│   ├── __init__.py      # Exports
│   └── planner.py       # Task decomposition
└── prompts/
    ├── __init__.py      # Exports
    └── builder.py       # Template-based prompt construction

TEMPLATES/
├── system.md            # Main system prompt template
├── reprime.md           # Re-prime template for context reset
└── planner.md           # Planner prompt template

TESTS/
├── run_all.py           # Test runner
├── test_config.py       # Config tests
├── test_tools.py        # Tool registry & tool tests
├── test_memory.py       # Session, context, persistent memory tests
├── test_agent.py        # Agent loop parsing & logic tests
└── test_prompts.py      # Template & prompt builder tests
```

## Running Tests

```bash
python3 TESTS/run_all.py
```

Or individual test files:

```bash
python3 TESTS/test_tools.py
python3 TESTS/test_memory.py
```

## Configuration

Just Intelligent CLI can be configured via:
- **Environment variables**: `OLLAMA_HOST`, `JICLI_CONFIG`, `JICLI_DATA_DIR`
- **Config file**: Set `JICLI_CONFIG` to a JSON file path

```json
{
    "model": "qwen3.5:9b",
    "host": "http://localhost:11434",
    "context_window": 32768,
    "temperature": 0.7,
    "max_turns": 100
}
```


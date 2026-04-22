# MiniClaw — Claude Code Build Prompt
# Paste this entire prompt into Claude Code to scaffold the project.

---

Build me a project called **MiniClaw** — a lightweight local AI assistant
inspired by OpenClaw (openclaw.ai), but designed to run entirely offline on
low-power hardware (Lenovo ThinkCentre M910q, Intel i5 7th Gen, 16 GB RAM).

## Stack

- **Language:** Python 3 (stdlib only — zero pip dependencies)
- **LLM backend:** Ollama HTTP API (`http://localhost:11434`)
- **Default model:** `qwen2.5:1.5b`
- **No cloud, no API keys required**

---

## Project Structure

Create the following files and folders exactly:

```
miniclaw/
├── miniclaw.json          ← main config
├── agent.py               ← core engine + CLI entrypoint
├── server.py              ← web UI (HTTP server, port 8080)
├── setup.sh               ← install script (Ollama + model pull)
├── README.md
├── memory/                ← auto-created at runtime
├── logs/                  ← auto-created at runtime
├── skills/
│   ├── _template/
│   │   ├── SKILL.md
│   │   └── skill.py
│   ├── weather/
│   │   ├── SKILL.md
│   │   └── skill.py
│   ├── shell/
│   │   ├── SKILL.md
│   │   └── skill.py
│   └── notes/
│       ├── SKILL.md
│       └── skill.py
└── plugins/
    ├── _template/
    │   ├── plugin.json
    │   └── plugin.py
    ├── calculator/
    │   ├── plugin.json
    │   └── plugin.py
    └── timer/
        ├── plugin.json
        └── plugin.py
```

---

## miniclaw.json

```json
{
  "version": "1.0.0",
  "agent": {
    "name": "MiniClaw",
    "persona": "You are MiniClaw, a helpful local AI assistant running on Ollama. You are concise, capable, and friendly. You can run skills and plugins to help the user accomplish tasks.",
    "model": "ollama/qwen2.5:1.5b",
    "max_tokens": 1024,
    "temperature": 0.7,
    "context_window": 4096
  },
  "ollama": {
    "host": "http://localhost:11434",
    "model": "qwen2.5:1.5b",
    "timeout": 60,
    "stream": true
  },
  "memory": {
    "enabled": true,
    "path": "./memory/memory.json",
    "max_entries": 500,
    "summarize_after": 50
  },
  "skills": {
    "path": "./skills",
    "enabled": true,
    "auto_discover": true
  },
  "plugins": {
    "path": "./plugins",
    "enabled": true,
    "auto_discover": true
  },
  "gateway": {
    "host": "0.0.0.0",
    "port": 8765,
    "web_ui": true,
    "web_ui_port": 8080
  },
  "logging": {
    "level": "info",
    "path": "./logs/miniclaw.log",
    "max_size_mb": 10
  },
  "performance": {
    "low_memory_mode": true,
    "max_history_messages": 20,
    "response_timeout": 90
  }
}
```

---

## agent.py — Core Engine

Implement a `MiniClawAgent` class with:

### Memory class
- Load/save `memory/memory.json` (JSON with keys: `facts`, `conversation`, `skills_used`)
- `add_fact(fact: str)` — stores a fact with ISO timestamp
- `add_message(role, content)` — appends to conversation history, trimmed to `max_history_messages`
- `get_history()` — returns list of `{role, content}` dicts
- `get_facts_summary()` — returns last 10 facts as a bullet string

### Skill class
- Loads from a folder containing `SKILL.md` and optionally `skill.py`
- Parses `SKILL.md` frontmatter for: `name`, `description`, `triggers` (comma-separated)
- Dynamically imports `skill.py` and calls `run(args, memory, cfg) -> str`
- `matches(text)` — returns True if any trigger word appears in the text

### SkillManager class
- Auto-discovers all valid skill folders in `./skills/`
- `find_skill(text)` — returns first matching Skill or None
- `run_skill(name, args)` — runs a skill by name
- `list_skills()` — formatted string of all skills + descriptions

### Plugin class
- Loads from a folder containing `plugin.json` and `plugin.py`
- `plugin.json` has: `name`, `description`, `commands` (list of strings)
- `plugin.py` exports a `COMMANDS` dict mapping command name → callable

### PluginManager class
- Auto-discovers all plugin folders in `./plugins/`
- `handle_command(cmd, args)` — routes `/cmd args` to the right plugin function
- `list_plugins()` — formatted list of all commands + descriptions

### OllamaClient class
- `is_running()` — GET `/api/tags`, returns bool
- `chat(messages, system)` — POST `/api/chat` with model, messages, options; returns response string
- Uses only `urllib.request` (no external libraries)
- Respects `timeout` and `max_tokens` from config

### MiniClawAgent class
- Builds system prompt from: persona + facts summary + skills list + plugins list
- Appends `[REMEMBER: <fact>]` instruction to system prompt
- `_handle_slash_command(text)` — handles: `/status`, `/help`, `/skills`, `/plugins`, `/memory`, `/clear`, `/skill <name> [args]`, plus delegates to PluginManager
- `_extract_memory(response)` — regex-extracts `[REMEMBER: ...]` tags, stores as facts, strips tags from response
- `respond(user_input)` — full pipeline: slash check → skill auto-trigger → LLM chat → memory extraction → history update

### CLI loop (`run_cli()`)
- Prints startup banner with model name
- Warns if Ollama is not running
- Input loop with `You: ` prompt
- Shows response time in seconds after each reply
- Exits cleanly on Ctrl+C

---

## server.py — Web UI

Implement a pure stdlib `HTTPServer` on port 8080.

### Routes
- `GET /` — serve the full HTML/CSS/JS single-page chat UI (inline in the Python file as a string)
- `GET /status` — JSON: `{ollama: bool, model: str, skills: int, plugins: int}`
- `POST /chat` — JSON body `{message: str}`, returns `{response: str}`

### Web UI Design Requirements

Dark terminal aesthetic. Use these CSS variables:
```css
--bg: #0d0f14
--surface: #13161e
--border: #1f2433
--accent: #e84040
--text: #e8eaf0
--muted: #6b7080
```

Fonts: `JetBrains Mono` (monospace) + `Syne` (headings) from Google Fonts.

UI elements:
- Header: 🦀 logo, "MiniClaw" brand, model tag chip, animated status dot (green=online, red=offline)
- Message area: scrollable, bot messages left-aligned with 🦀 avatar, user messages right-aligned
- Typing indicator: animated 3-dot pulse while waiting
- Quick command buttons: `/status`, `/skills`, `/plugins`, `/memory`, `/clear`, `/help`
- Input: auto-resizing textarea, Enter to send, Shift+Enter for newline
- Send button: red, arrow icon
- Timestamps + response time shown under each message
- Fade-up animation on new messages
- Polls `/status` every 10 seconds to update the status dot

---

## Skills Implementation

### skills/weather/skill.py
- Extract city name from natural language (e.g. "weather in London" → "London")
- Fetch from `https://wttr.in/{city}?format=4` using urllib
- Return formatted string with emoji
- Fallback gracefully if network unavailable

### skills/shell/skill.py
- ALLOWED set: `ls`, `pwd`, `df`, `free`, `ps`, `uname`, `uptime`, `date`, `whoami`, `hostname`, `cat`, `echo`, `which`, `du`, `lscpu`
- Parse args with `shlex.split()`
- Reject commands not in ALLOWED set
- Run with `subprocess.run(..., capture_output=True, timeout=10)`
- Trim output to 40 lines max
- Return output wrapped in triple backticks

### skills/notes/skill.py
- Subcommands: `save <text>`, `list`, `search <term>`, `clear`
- Store in `memory/notes.json` as list of `{text, ts}` objects
- Return formatted strings with emoji

---

## Plugins Implementation

### plugins/calculator/plugin.py
- Safe math evaluator using Python `ast` module only — NO `eval()`
- Walk the AST, only allow: numbers, +, -, *, /, //, %, ** and unary +/-
- `/calc <expression>` → `🧮 expr = result`
- Handle ZeroDivisionError gracefully

### plugins/timer/plugin.py
- `/timer <seconds> [label]`
- Validates 1–86400 seconds
- Runs countdown in a `daemon=True` background thread
- Prints a bell notification to stdout when done
- Returns human-readable confirmation (e.g. "⏱️ Timer set for 5m 0s — 'pasta'")

---

## SKILL.md Format

Each `SKILL.md` must have a YAML-style frontmatter block at the top:

```markdown
---
name: skill_name
version: 1.0.0
description: One-line description shown in /skills list
triggers: keyword1, keyword2, trigger phrase
author: MiniClaw
---

# Skill Title

Long description, usage examples, notes.
```

---

## Plugin JSON Format

```json
{
  "name": "plugin_name",
  "version": "1.0.0",
  "description": "Shown in /plugins list",
  "author": "MiniClaw",
  "commands": ["commandname"]
}
```

---

## setup.sh

Bash script that:
1. Checks for Python 3
2. Checks for / installs Ollama via `curl -fsSL https://ollama.com/install.sh | sh`
3. Starts `ollama serve` in background if not already running
4. Runs `ollama pull qwen2.5:1.5b`
5. Creates `memory/` and `logs/` directories
6. Prints a coloured success summary with usage instructions

---

## Performance Constraints

Design everything with these hardware limits in mind:
- **CPU only** — no GPU acceleration
- **Max ~3 GB RAM** for the whole stack
- `max_tokens: 1024` keeps responses fast on CPU
- `context_window: 4096` — don't exceed this
- `max_history_messages: 20` — trim conversation history aggressively
- All I/O is synchronous (no async/await) — keeps code simple and debuggable

---

## README.md

Write a full README covering:
- What MiniClaw is (1 paragraph)
- Quick start (setup.sh → server.py or agent.py)
- miniclaw.json config reference with model swap examples
- All built-in commands table (/status, /help, /skills, /plugins, /memory, /clear, /calc, /timer)
- Skills documentation (weather, shell, notes)
- How to create a custom skill (step by step)
- How to create a custom plugin (step by step)
- Memory system explanation
- Performance tips for low-power hardware
- Project structure tree
- Comparison table vs OpenClaw

---

## Final Checklist

After generating all files, verify:
- [ ] `python3 agent.py` starts without errors (even if Ollama is offline — it should warn, not crash)
- [ ] `python3 server.py` starts and serves HTML at port 8080
- [ ] `/status` command works in CLI
- [ ] `/calc 2 + 2` returns 4
- [ ] `/skill notes save hello` saves a note
- [ ] Skills auto-discover from `./skills/` folder
- [ ] Plugins auto-discover from `./plugins/` folder
- [ ] Memory persists between runs (written to `memory/memory.json`)
- [ ] No external pip packages used anywhere

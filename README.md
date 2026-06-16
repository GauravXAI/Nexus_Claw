# Nexusclaw

An agentic AI CLI tool for automating codebase tasks — locally via terminal or remotely via Telegram. Built on [Vercel AI SDK](https://sdk.vercel.ai/), [Bun](https://bun.sh/), and [OpenRouter](https://openrouter.ai/).

---

## Architecture Overview

```
nexusclaw/
├── index.ts                  # CLI entrypoint (commander)
├── ai/
│   └── ai.config.ts          # OpenRouter model factory
├── tui/
│   ├── wakeup.ts             # Banner + mode selector
│   └── terminal-md.ts        # marked-terminal renderer
└── modes/
    ├── cli.ts                # CLI sub-mode router (agent/plan/ask)
    ├── agent/                # Core agent engine
    │   ├── type.ts           # ActionLog, AgentConfig, ActionType types
    │   ├── action-tracker.ts # Append-only mutation log with status tracking
    │   ├── tool-executor.ts  # FS operations + overlay staging layer
    │   ├── agent-tools.ts    # AI SDK tool definitions (wraps executor)
    │   ├── approval.ts       # CLI interactive approval flow
    │   ├── diff-view.ts      # Unified diff generation
    │   └── orchestrator.ts   # Agent mode entry point
    ├── ask/
    │   └── orchestrator.ts   # Read-only Q&A agent, optional .md save
    ├── plan/
    │   ├── types.ts          # Plan, PlanStep types
    │   ├── planner.ts        # JSON-schema-constrained plan generator
    │   ├── selection.ts      # multiselect step picker (CLI)
    │   ├── web-tools.ts      # Firecrawl web_search/web_crawl/fetch_url
    │   └── orchestrator.ts   # Plan mode entry point
    └── telegram/
        ├── index.ts          # Telegraf bot launch
        ├── handlers.ts       # Command + callback action registration
        ├── agent-run.ts      # ask/agent/planSteps runners for Telegram
        ├── approval-session.ts # Inline keyboard approval flow
        ├── plan-session.ts   # Interactive plan toggle UI
        ├── auth.ts           # Owner-only guard
        ├── constants.ts      # Welcome message
        └── text.ts           # Telegram text helpers (clip, replyMd)
```

---

## How It Works

### Staging Layer

All file mutations are **staged, never directly written**. `ToolExecutor` maintains an in-memory overlay (`Map<path, content>`) and a deletion set. The `ActionTracker` records every operation as an `ActionLog` with status `pending | approved | rejected | executed`. Nothing hits disk until the user explicitly approves.

```
Agent calls tool → ToolExecutor stages change → ActionTracker logs it
                                                        ↓
                                            User reviews via approval flow
                                                        ↓
                                            applyApprovedFromTracker() writes to disk
```

### Modes

| Mode | Description | Mutations |
|------|-------------|-----------|
| **Agent** | Free-form agentic task on your codebase | Yes — with approval |
| **Ask** | Read-only Q&A with optional `.md` save | Optional file create |
| **Plan** | LLM generates a step plan → you pick steps → agent executes each | Yes — with approval |
| **Telegram** | All three modes via bot commands with inline keyboard approval | Yes — with approval |

---

## Prerequisites

- [Bun](https://bun.sh/) `>= 1.x`
- Node.js is **not** required (Bun runtime only)

---

## Installation

```bash
git clone https://github.com/your-user/nexusclaw.git
cd nexusclaw
bun install
```

---

## Configuration

Create a `.env` file in the project root:

```env
# Required
OPENROUTER_API_KEY=sk-or-...
OPENROUTER_DEFAULT_MODEL=anthropic/claude-3.5-sonnet

# Optional: enables web_search / web_crawl / fetch_url tools in Plan and Ask modes
FIRECRAWL_API_KEY=fc-...

# Required only for Telegram mode
TELEGRAM_BOT_TOKEN=...
TELEGRAM_OWNER_ID=...        # Your Telegram user ID (integer)

# Optional: custom skill directories (semicolon-separated)
SKILLS_DIRS=/path/to/skills;/another/path
```

> `TELEGRAM_OWNER_ID` gates all bot commands — only that user ID can interact with the bot.

---

## Usage

### Start

```bash
bun run index.ts wakeup
# or
bunx nexusclaw wakeup
```

You'll see the nexusclaw ASCII banner and a mode prompt.

### CLI Mode

```
? Choose CLI sub-mode
❯ Agent Mode     — give the agent a task, review & apply changes
  Plan Mode      — generate a plan, pick steps, execute
  Ask Mode       — ask a question, get a markdown answer
  ← Back
```

### Telegram Mode

```
/start           — show help
/ask <question>  — read-only Q&A about your codebase
/agent <task>    — agentic task with inline approval
/plan <goal>     — generate plan, toggle steps, execute
```

After `/agent` or `/plan`, the bot sends an approval message with:
- **Show Diff** — view unified diff of staged changes
- **Accept All** — write all staged changes to disk
- **Reject All** — discard everything

---

## Tool Capabilities

### Workspace Tools (all modes)

| Tool | Description |
|------|-------------|
| `read_file` | Read a file (size-capped at 1MB by default) |
| `list_files` | List directory contents, optionally recursive |
| `search_files` | Glob pattern search with optional content filter |
| `analyze_codebase` | File/dir count summary |
| `create_file` | Stage a new file |
| `modify_file` | Stage a full-file replacement |
| `delete_file` | Stage a file deletion |
| `create_folder` | Stage directory creation |
| `execute_shell` | Queue a shell command (runs post-approval) |
| `list_skills` | Find SKILL.md files in `~/.cursor/skills-cursor` or `~/.claude/skills` |
| `read_skill` | Read a SKILL.md |

### Web Tools (Plan/Ask/Telegram, requires `FIRECRAWL_API_KEY`)

| Tool | Description |
|------|-------------|
| `web_search` | Search the web, returns title/url/snippet list |
| `web_crawl` | Scrape a URL to markdown |
| `fetch_url` | Raw HTTP GET |

---

## Default Exclusions

The following paths are never read or staged (configurable via `AgentConfig.excludePatterns`):

```
node_modules  .git  dist  build  .next  *.log  .env*
```

---

## Agent Step Limits

| Mode | Max Steps |
|------|-----------|
| Agent | 40 |
| Plan (per step) | 30 |
| Ask | 20 |
| Plan generation | 20 |
| Telegram Ask | 20 |

Controlled via `stepCountIs(n)` from the AI SDK. Adjust in the respective orchestrator.

---

## Development

```bash
# Run directly
bun run index.ts wakeup

# Type-check
bunx tsc --noEmit
```

No build step required — Bun executes TypeScript natively.

---

## Known Limitations & Notes

- **No conversation history** — each agent invocation is stateless. Context is limited to a single `generate()` call.
- **Full-file replacement only** — `modify_file` replaces the entire file. There's no patch/hunk-level editing; the agent must rewrite the whole file.
- **Single-user Telegram** — `TELEGRAM_OWNER_ID` enforces one owner. Multi-user sessions are not supported.
- **Shell execution is on by default** — review `defaultAgentConfig()` in `modes/agent/type.ts` and set `allowShellExecution: false` if running in untrusted environments.
- **Overlay is in-memory** — if the process crashes mid-session, staged changes are lost (nothing was written anyway).


## License

Private. See `package.json`.

---

## Output

<img width="1920" height="1080" alt="Screenshot From 2026-06-16 17-34-20" src="https://github.com/user-attachments/assets/5dd7c6f9-64f6-45e7-ba44-8254ad4c463a" />
<img width="1920" height="1080" alt="Screenshot From 2026-06-16 17-34-13" src="https://github.com/user-attachments/assets/8c23a7c6-215e-4aa3-bdac-f6c0e97e50d4" />
<img width="1920" height="1080" alt="Screenshot From 2026-06-16 17-34-04" src="https://github.com/user-attachments/assets/a5339e3c-f7f9-4911-8209-0cbc07dcddcd" />
<img width="1920" height="1080" alt="Screenshot From 2026-06-16 17-33-10" src="https://github.com/user-attachments/assets/4b8749b9-bb4a-4bb2-ac42-9df226dada46" />
<img width="1920" height="1080" alt="Screenshot From 2026-06-16 17-33-04" src="https://github.com/user-attachments/assets/d7b11ac7-2174-4eef-8bf8-f0744567936d" />
<img width="1920" height="1080" alt="Screenshot From 2026-06-16 17-32-57" src="https://github.com/user-attachments/assets/580e23f5-a907-437c-8e43-aa466345e2ca" />


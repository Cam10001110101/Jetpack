# Jetpack Cheatsheet

Quick reference for Jetpack - Multi-agent orchestration for software development

---

# Full Setup (Most Capable)

## Global Installation (Recommended)

```bash
# Clone and build
cd ~/GITHUB  # or your preferred location
git clone https://github.com/spencerthomas/Jetpack.git
cd Jetpack
pnpm install && pnpm build
```

```bash
# Setup pnpm global bin (first time only)
pnpm setup
source ~/.zshrc  # or ~/.bashrc for bash
```

```bash
# Link CLI globally
cd apps/cli
pnpm link --global
```

```bash
# Verify installation
cd ~
jetpack --version
```

```bash
# Use in any project
cd ~/my-project
jetpack init
jetpack start --agents 3
```

### What You Get

| Component | What it does |
| --------- | ------------ |
| Global CLI | Use `jetpack` command anywhere (like Beads `bd` command) |
| Web UI | Kanban dashboard on http://localhost:3002 |
| Agents | Autonomous task execution with swarm intelligence |
| Memory (CASS) | Vector-based persistent memory for learned patterns |
| Task System | Git-backed `.beads/` for task tracking with dependencies |
| MCP Integration | Bidirectional sync with Claude Code |

---

# Installation Options

## Option 1: Global Installation (Recommended)

**When to use:** Working across multiple projects, Beads-like workflow

```bash
cd /path/to/Jetpack
pnpm install && pnpm build
pnpm setup && source ~/.zshrc
cd apps/cli && pnpm link --global

# Then use anywhere
cd ~/any-project
jetpack init
jetpack start
```

**Benefits:**
- Use `jetpack` command in any project
- Web UI automatically available
- Easy updates: `git pull && pnpm build`
- Symlink preserves monorepo structure

## Option 2: Local Development

**When to use:** Contributing to Jetpack, developing new features

```bash
cd /path/to/Jetpack
pnpm install
pnpm build

# Run via pnpm
pnpm jetpack start
pnpm jetpack task -t "My task" -p high
```

## Option 3: MCP Server (Claude Code Integration)

**Method A: Direct Node Invocation**

```bash
# Ensure Jetpack is built
cd /path/to/Jetpack
pnpm build
```

Add to `.claude/settings.local.json`:

```json
{
  "mcpServers": {
    "jetpack": {
      "command": "node",
      "args": ["/path/to/Jetpack/packages/mcp-server/dist/index.js"],
      "env": {
        "JETPACK_WORK_DIR": "/path/to/your/project"
      }
    }
  }
}
```

**Method B: CLI Wrapper**

```json
{
  "mcpServers": {
    "jetpack": {
      "command": "node",
      "args": [
        "/path/to/Jetpack/apps/cli/dist/index.js",
        "mcp",
        "--dir",
        "/path/to/your/project"
      ]
    }
  }
}
```

After configuration:
1. Restart Claude Code
2. Verify with `/mcp` command
3. Tools will appear under "jetpack" server

---

# Core Commands

| Command | Description | Example |
|---------|-------------|---------|
| `init` | Initialize project directories | `jetpack init` |
| `start` | Launch orchestrator + agents + web UI | `jetpack start -a 3` |
| `task` | Create a new task | `jetpack task -t "Title" -p high` |
| `status` | Show system status | `jetpack status` |
| `demo` | Run guided demo workflow | `jetpack demo --agents 5` |
| `supervise` | AI-powered task orchestration | `jetpack supervise "Build API"` |
| `mcp` | Start MCP server | `jetpack mcp --dir /path` |

## Start Command Options

```bash
# Basic
jetpack start                          # 3 agents, port 3002
jetpack start -a 5                     # 5 agents
jetpack start -p 3005                  # Custom port
jetpack start --dir /path/to/project   # Specific directory

# Behavior
jetpack start --no-browser             # Don't auto-open browser
jetpack start --no-ui                  # CLI-only mode

# Runtime Limits
jetpack start --max-cycles 100         # Stop after 100 cycles
jetpack start --max-runtime 8h         # Stop after 8 hours
jetpack start --idle-timeout 5m        # Stop if idle for 5 minutes
jetpack start --max-failures 10        # Stop after 10 consecutive failures
```

## Task Command Options

```bash
jetpack task -t "Title" \
  -d "Description" \
  -p high \                            # low, medium, high, critical
  -s typescript,backend \              # Required skills
  -e 30                                # Estimated minutes
```

---

# Task Creation Methods

## Method 1: CLI Command

```bash
jetpack task \
  --title "Implement feature X" \
  --priority high \
  --skills typescript,react \
  --estimate 60
```

## Method 2: Markdown File

Drop a `.md` file in `.beads/tasks/`:

```markdown
---
title: Implement user authentication
priority: high
skills: [typescript, backend, security]
estimate: 120
---

Build JWT-based authentication with refresh tokens.

Requirements:
- Login/logout endpoints
- Password hashing with bcrypt
- Token refresh mechanism
```

Files are auto-detected within 500ms.

## Method 3: Web UI

1. Open http://localhost:3002
2. Click "New Task"
3. Fill in form
4. Changes sync to `.beads/tasks.jsonl` automatically

---

# MCP Tools Reference

Once configured in Claude Code, these tools become available:

## Plan Management
- `jetpack_list_plans` - List all plans
- `jetpack_get_plan` - Get specific plan details
- `jetpack_create_plan` - Create plan with items
- `jetpack_update_plan` - Update plan status/title

## Task Management
- `jetpack_list_tasks` - List tasks (filter by status)
- `jetpack_get_task` - Get task details
- `jetpack_create_task` - Create new task
- `jetpack_claim_task` - Claim task
- `jetpack_start_task` - Mark in progress
- `jetpack_complete_task` - Mark completed
- `jetpack_fail_task` - Mark failed

## Status & Sync
- `jetpack_status` - System status (agents, tasks, memory)
- `jetpack_sync_todos` - Sync Claude Code todos to Jetpack tasks

---

# Supervisor (LangGraph Orchestration)

AI-powered task breakdown and orchestration using LangGraph.

```bash
# With Claude (requires ANTHROPIC_API_KEY)
jetpack supervise "Build user authentication system"

# With OpenAI (requires OPENAI_API_KEY)
jetpack supervise "Add REST API" --llm openai

# With Ollama (local)
jetpack supervise "Fix login bug" --llm ollama --model llama2

# Custom agent count
jetpack supervise "request" --agents 10
```

The supervisor:
1. **Plans** - Breaks down request into tasks
2. **Assigns** - Matches tasks to agent skills
3. **Monitors** - Tracks progress and dependencies
4. **Coordinates** - Resolves conflicts and blockers

---

# Common Workflows

## Starting Work

```bash
# Global install
cd ~/my-project
jetpack init
jetpack start -a 3

# Web UI opens at http://localhost:3002
```

## Creating Dependent Tasks

```bash
# Create parent task
jetpack task -t "Set up database" -p high -s database

# Create dependent task
jetpack task -t "Create API" -p high -s backend --depends bd-XXXX
```

## Using with Claude Code

1. Configure MCP server (see Option 3 above)
2. Restart Claude Code
3. Create tasks in Claude Code or web UI
4. Both interfaces stay in sync automatically

## Updating Jetpack

```bash
cd /path/to/Jetpack
git pull
pnpm install
pnpm build

# Global link stays active - no need to re-link
```

---

# Environment Variables

| Variable | Required For | Purpose |
|----------|--------------|---------|
| `ANTHROPIC_API_KEY` | Claude agents/supervisor | Anthropic API authentication |
| `OPENAI_API_KEY` | OpenAI supervisor | OpenAI model access |
| `JETPACK_WORK_DIR` | Web UI / MCP server | Points to target project directory |

## Setting JETPACK_WORK_DIR

```bash
# For web UI
JETPACK_WORK_DIR=/path/to/project pnpm --filter @jetpack/web dev

# In MCP server config
{
  "env": {
    "JETPACK_WORK_DIR": "/path/to/your/project"
  }
}
```

---

# Data Storage

| Component | Location | Format | Git? |
|-----------|----------|--------|------|
| Tasks | `.beads/tasks.jsonl` | JSON Lines | ✅ Yes |
| Plans | `.jetpack/plans/*.json` | JSON | ❌ No |
| Memory | `.cass/memory.db` | SQLite | ❌ No |
| Messages | `.jetpack/mail/` | JSON | ❌ No |
| Config | `.jetpack/config.json` | JSON | Optional |

---

# Troubleshooting

| Issue | Solution |
|-------|----------|
| `jetpack` command not found | Run `pnpm link --global` in `apps/cli/` directory |
| Build fails with TypeScript errors | `pnpm clean && pnpm install && pnpm build` |
| Web UI shows wrong/old data | Verify `JETPACK_WORK_DIR` environment variable |
| MCP tools not appearing | Restart Claude Code after config changes |
| Agents not claiming tasks | Check `requiredSkills` matches agent skills |
| Port 3002 already in use | Use `jetpack start -p 3005` for different port |

---

# Quick Reference

```bash
# One-time global setup
cd /path/to/Jetpack && pnpm install && pnpm build
pnpm setup && source ~/.zshrc
cd apps/cli && pnpm link --global

# Use in any project
cd ~/project && jetpack init && jetpack start -a 5

# Create tasks
jetpack task -t "Title" -p high -s typescript,react

# AI orchestration
jetpack supervise "Build user auth" --agents 5

# Check status
jetpack status
```

---

**Documentation:** See `CLAUDE.md` for full architecture details
**Repository:** https://github.com/spencerthomas/Jetpack

# @jetpack/cli

Command-line interface for Jetpack multi-agent orchestration system.

## Installation

### Global Installation (Recommended)

Install Jetpack once, use it in any project:

```bash
# 1. Clone and build Jetpack
cd ~/GITHUB  # or wherever you keep repos
git clone https://github.com/spencerthomas/Jetpack.git
cd Jetpack
pnpm install && pnpm build

# 2. Setup pnpm global bin (first time only)
pnpm setup
source ~/.zshrc  # or ~/.bashrc

# 3. Link CLI globally
cd apps/cli
pnpm link --global

# 4. Verify
jetpack --version
```

### Local Development

```bash
# From Jetpack root
pnpm install
pnpm build
pnpm jetpack --help
```

## Usage

### Initialize a Project

```bash
cd ~/my-project
jetpack init              # Creates .beads/, .cass/, .jetpack/
jetpack init -a 5 -p 3005 # Custom: 5 agents, port 3005
```

### Start the System

```bash
jetpack start              # Default: 3 agents, port 3002
jetpack start -a 5         # With 5 agents
jetpack start --no-ui      # CLI-only (no web UI)
jetpack start --no-browser # Don't auto-open browser
```

### Manage Tasks

```bash
# Create a task
jetpack task -t "Fix login bug" -p high -s typescript,backend

# Check status
jetpack status

# Run demo workflow
jetpack demo --agents 5
```

### AI Supervisor

```bash
# Break down high-level requests into tasks
jetpack supervise "Build user authentication" --llm claude
jetpack supervise "Add dark mode" --llm openai --model gpt-4-turbo
jetpack supervise "Fix bug" --llm ollama --model llama2
```

### MCP Server

```bash
# Start MCP server for Claude Code integration
jetpack mcp --dir /path/to/project
```

## Commands Reference

| Command | Description |
|---------|-------------|
| `init [path]` | Initialize Jetpack in a directory |
| `start` | Start orchestrator + agents + web UI |
| `task` | Create a new task |
| `status` | Show system status |
| `demo` | Run guided demo with sample tasks |
| `supervise <request>` | AI-powered task breakdown |
| `mcp` | Start MCP server for Claude Code |

## Options

### init

```bash
jetpack init [path] [options]

Options:
  -a, --agents <number>    Default number of agents (default: 3)
  -p, --port <number>      Default web UI port (default: 3002)
  --no-gitignore           Skip .gitignore updates
  --no-claude-md           Skip CLAUDE.md updates
```

### start

```bash
jetpack start [options]

Options:
  -a, --agents <number>    Number of agents to spawn (default: from config)
  -p, --port <number>      Web UI port (default: from config)
  -d, --dir <path>         Project directory (default: current directory)
  --no-ui                  CLI-only mode (skip web UI)
  --no-browser             Don't auto-open browser
```

### task

```bash
jetpack task [options]

Options:
  -t, --title <string>         Task title (required)
  -d, --description <string>   Task description
  -p, --priority <level>       Priority: low, medium, high, critical
  -s, --skills <list>          Comma-separated skills (typescript,react,backend)
  -e, --estimate <minutes>     Estimated time in minutes
```

### supervise

```bash
jetpack supervise <request> [options]

Arguments:
  request                  High-level request for the supervisor

Options:
  --llm <provider>        LLM provider: claude, openai, ollama (default: claude)
  --model <name>          Model name (default varies by provider)
  -a, --agents <number>   Number of agents (default: 3)
```

## Environment Variables

| Variable | Description |
|----------|-------------|
| `ANTHROPIC_API_KEY` | Required for Claude agents and supervisor |
| `OPENAI_API_KEY` | Required for OpenAI supervisor or embeddings |
| `JETPACK_WORK_DIR` | Override working directory (for web UI and MCP server) |

## Global Installation Details

### How It Works

`pnpm link --global` creates a **symlink** to the CLI:
- CLI runs from the Jetpack source location
- Preserves relative paths (web app at `../../web` still works)
- No code changes needed
- Ideal for development and multi-project usage

### What Gets Created

When you run `jetpack init`, these folders are created:
- `.beads/` - Task storage (git-tracked)
- `.beads/tasks/` - Drop markdown files here to create tasks
- `.cass/` - Agent memory (gitignored)
- `.jetpack/` - Configuration and agent communication

### .gitignore Patterns

The init command adds to `.gitignore`:
```gitignore
# Jetpack
.cass/
.jetpack/mail/
.jetpack/agents.json
.jetpack/plans/
```

**Note:** `.beads/` is tracked in git to preserve task history.

### Updating

```bash
# Pull latest changes
cd ~/GITHUB/Jetpack
git pull
pnpm install
pnpm build
# Global link stays active
```

### Uninstalling

```bash
pnpm unlink --global @jetpack/cli
```

## Known Limitations

**npm vs pnpm:**
| Method | Works? | Reason |
|--------|--------|--------|
| `pnpm link --global` | ✅ YES | Symlink preserves monorepo structure |
| `npm install -g` | ❌ NO | Web app not included in published package |

The current global linking approach is **recommended for development** and multi-project usage from source.

## Development

### Building

```bash
# From Jetpack root
pnpm build

# Build only CLI
pnpm --filter @jetpack/cli build
```

### Testing

```bash
# From apps/cli
pnpm test
```

### Project Structure

```
apps/cli/
├── src/
│   ├── index.ts           # Main CLI entry point with Commander.js
│   └── commands/          # (Future: individual command files)
├── dist/
│   └── index.js          # Compiled CLI (with shebang)
├── package.json          # Bin configuration
└── tsconfig.json         # TypeScript config
```

### Key Files

- **src/index.ts** - CLI entry point with all commands
  - Uses Commander.js for argument parsing
  - Line 96: Web UI path resolution (`__dirname + '../../web'`)
  - Line 233: .gitignore pattern creation

- **package.json:7-9** - Bin configuration
  ```json
  "bin": {
    "jetpack": "./dist/index.js"
  }
  ```

## Contributing

When adding new commands:
1. Add command definition in `src/index.ts`
2. Follow existing patterns (Commander.js)
3. Update this README with new command docs
4. Rebuild: `pnpm build`

## License

MIT

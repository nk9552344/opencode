# Chapter 1: Getting Started

## Prerequisites

- **Bun** >= 1.3.14 (the package manager and runtime)
- **Node.js** is not required for the compiled binary, but is needed for some Node-specific library paths

Install Bun: https://bun.sh

## Installation (from source)

```bash
# Clone and install dependencies
bun install

# Run in dev mode (no compilation, runs directly from TypeScript)
bun run dev
# equivalent to: bun run src/index.ts
```

## Building the Binary

```bash
# Build for the current platform only (fast iteration)
bun run build -- --single

# Build for all 12 platforms (CI/release)
bun run build

# Build outputs to dist/
# dist/cli-linux-arm64/bin/lildax
# dist/cli-linux-x64/bin/lildax
# dist/cli-darwin-arm64/bin/lildax
# dist/cli-darwin-x64/bin/lildax
# dist/cli-win32-x64/bin/lildax
# etc.
```

The binary is self-contained: it embeds the Bun runtime, all JavaScript, and compile-time constants (version, model catalog).

## Running

```bash
# Open the full TUI (default)
opencode

# Or in dev mode (runs from TypeScript source)
bun run src/index.ts
```

## CLI Commands

```
opencode                        Start the TUI (most users just use this)
opencode api <operation>        Make a raw HTTP request to the running server
opencode service start          Start the background server
opencode service stop           Stop the background server
opencode service restart        Restart the background server
opencode service status         Print server URL or "stopped"
opencode service password       Print the server password
opencode service password <val> Set a new server password (restarts server)
opencode debug agents           List all available agents (JSON)
opencode migrate                Run data migrations (currently a no-op)
opencode serve                  Start the HTTP server in-process (internal)
```

## Configuration

Configuration is loaded from multiple sources in priority order (last wins):

1. `~/.config/opencode/opencode.json` — global user config
2. `opencode.json` in any parent directory up to the project root
3. `.opencode/opencode.json` in any parent directory

### Minimal config to set a provider

```jsonc
// opencode.json
{
  "model": "anthropic/claude-sonnet-4-5",
  "$schema": "https://opencode.ai/config.json"
}
```

### Full config structure

```jsonc
{
  "$schema": "https://opencode.ai/config.json",

  // Default model (providerID/modelID or just modelID)
  "model": "anthropic/claude-sonnet-4-5",

  // Default agent
  "default_agent": "coder",

  // Shell to use for bash tool (default: /bin/sh on POSIX)
  "shell": "/bin/bash",

  // Provider auth overrides (alternative to env vars)
  "providers": {
    "anthropic": {
      "options": { "apiKey": "sk-ant-..." }
    },
    "openai": {
      "options": { "apiKey": "sk-..." }
    }
  },

  // Custom agents
  "agents": [
    {
      "id": "my-agent",
      "model": "anthropic/claude-opus-4-5",
      "system": "You are a specialized expert in...",
      "permissions": {
        "bash": { "allow": ["*"] }
      },
      "steps": 50
    }
  ],

  // Context compaction settings
  "compaction": {
    "auto": true,
    "buffer": 20000,
    "keep": { "tokens": 8000 }
  },

  // File snapshots for revert support
  "snapshots": true,

  // MCP server configuration
  "mcp": {
    "servers": {
      "my-mcp-server": {
        "command": "npx",
        "args": ["-y", "my-mcp-package"],
        "env": {}
      }
    }
  },

  // Skill sources
  "skills": [
    "./my-skills/",
    "https://example.com/skills.zip"
  ],

  // Custom instructions (merged with AGENTS.md)
  "instructions": "./instructions/",

  // File references (named external directories)
  "references": [
    { "name": "docs", "path": "./docs" },
    { "name": "api-repo", "git": "https://github.com/example/api" }
  ],

  // Plugins
  "plugins": [
    "./my-plugin.ts",
    ["npm-plugin-name", { "option": "value" }]
  ],

  // Tool output limits
  "tool_output": {
    "max_lines": 2000,
    "max_bytes": 51200
  },

  // Permission rules
  "permissions": {
    "bash": {
      "allow": ["ls", "cat *", "git *"],
      "deny": ["rm -rf *"]
    }
  },

  // Auto-update
  "autoupdate": true
}
```

## Environment Variables

Providers also accept API keys from environment variables:

| Provider | Environment Variable |
|---|---|
| Anthropic | `ANTHROPIC_API_KEY` |
| OpenAI | `OPENAI_API_KEY` |
| Google | `GOOGLE_GENERATIVE_AI_API_KEY` |
| Azure | `AZURE_OPENAI_API_KEY` |
| xAI | `XAI_API_KEY` |
| OpenRouter | `OPENROUTER_API_KEY` |
| Cloudflare Workers AI | `CLOUDFLARE_WORKERS_AI_TOKEN` |

Other runtime environment variables:

| Variable | Effect |
|---|---|
| `OPENCODE_DB` | Override the database file path |
| `OPENCODE_SERVER_PASSWORD` | Override the server password |
| `OPENCODE_SERVER_USERNAME` | Override the server username |
| `OPENCODE_DISABLE_SHARE` | Disable session sharing |
| `OPENCODE_CHANNEL` | Override the release channel (build-time) |
| `OPENCODE_VERSION` | Override the version (build-time) |

## AGENTS.md

Place an `AGENTS.md` file anywhere in your project hierarchy (or `~/.config/opencode/AGENTS.md` globally) to inject permanent instructions into every LLM turn. opencode discovers all `AGENTS.md` files from the current directory up to the project root and concatenates them in order.

```markdown
# My Project Instructions

Always write tests when creating new files.
Use the existing error handling patterns in src/errors/.
Prefer functional style.
```

## Server Lifecycle

The background server starts automatically when you run `opencode` (or any CLI command that needs the API). It keeps running after you close the TUI, so subsequent `opencode` invocations reuse the same server and pick up where they left off.

```bash
# Check if server is running
opencode service status

# Explicitly stop it
opencode service stop
```

The server state (URL + PID) is stored in `~/.local/share/opencode/server.json`. If the server crashes, the stale registration is detected via the health check and removed automatically.

## Typecheck

```bash
# Typecheck all packages
bun turbo typecheck

# Or individual packages
cd packages/core && bun run typecheck
```

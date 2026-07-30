# Chapter 0: Architecture Overview

## What opencode Is

opencode is an AI-powered developer CLI tool. You run `opencode` in your terminal, it starts a background HTTP server managing your AI sessions, and opens a full-featured terminal UI (TUI) where you can chat with AI models, execute tools on your codebase (bash, file edit/write/read, search), and manage multi-turn conversations backed by a local SQLite event log.

The system is written entirely in TypeScript, runs on Bun (or Node.js for library use), and is built around the **Effect** functional programming library — all services, layers, error channels, concurrency, and resource management flow through Effect's type-safe runtime.

---

## Repository Structure (Post-Cleanup)

```
/                          ← Root = the CLI package (entry point)
├── src/                   ← CLI source (entry, commands, framework, daemon)
│   ├── index.ts           ← main entry point
│   ├── tui.ts             ← TUI launcher
│   ├── framework/         ← command spec DSL + runtime wiring
│   ├── commands/          ← command declarations + all handler modules
│   └── services/          ← Daemon service
├── bin/
│   └── lildax.cjs         ← CJS shim that resolves and launches the binary
├── script/
│   ├── build.ts           ← cross-platform binary build script
│   └── generate.ts        ← fetches model catalog snapshot at build time
├── packages/
│   ├── core/              ← the engine: sessions, tools, agents, config, DB
│   ├── tui/               ← terminal UI (SolidJS + opentui)
│   ├── ui/                ← web/desktop UI component library (not used by TUI)
│   ├── plugin/            ← plugin type contracts (no runtime)
│   ├── llm/               ← LLM provider abstraction (protocols, routes, auth)
│   ├── server/            ← HTTP API server (Effect HttpApi)
│   ├── sdk/               ← JS SDK client (generated from OpenAPI)
│   ├── protocol/          ← HTTP API contract definitions (Effect HttpApi spec)
│   ├── schema/            ← shared domain schemas (Effect Schema)
│   ├── effect-drizzle-sqlite/ ← Effect + Drizzle ORM SQLite adapter
│   ├── effect-sqlite-node/    ← Effect SqlClient for Node.js built-in sqlite
│   └── script/            ← build utilities (version/channel determination)
├── package.json           ← workspace root + CLI package definition
├── bun.lock               ← lockfile
├── bunfig.toml            ← Bun configuration
├── turbo.json             ← Turbo task runner (typecheck only)
└── tsconfig.json          ← root TypeScript config (CLI JSX settings)
```

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         User Terminal                        │
│                                                             │
│   $ opencode                                                │
│         │                                                   │
│   ┌─────▼──────────────────────────────────────────────┐   │
│   │                   CLI Layer (src/)                   │   │
│   │  Daemon Service ─── starts background server        │   │
│   │  TUI Launcher   ─── opens terminal UI               │   │
│   └──────────────────────────┬──────────────────────────┘   │
│                              │ HTTP (localhost)              │
│   ┌──────────────────────────▼──────────────────────────┐   │
│   │                  HTTP Server (packages/server)        │   │
│   │  REST + SSE API  ─── /api/session, /api/event, ...  │   │
│   └──────────────────────────┬──────────────────────────┘   │
│                              │ Effect layers                 │
│   ┌──────────────────────────▼──────────────────────────┐   │
│   │                Core Engine (packages/core)           │   │
│   │  SessionRunner ─── LLM execution loop               │   │
│   │  ToolRegistry  ─── bash, edit, read, grep, ...      │   │
│   │  EventV2       ─── durable event log (SQLite)        │   │
│   │  Catalog       ─── provider + model registry        │   │
│   │  Config        ─── opencode.json loader             │   │
│   └──────────────────────────┬──────────────────────────┘   │
│                              │                              │
│   ┌──────────────────────────▼──────────────────────────┐   │
│   │            LLM Layer (packages/llm)                  │   │
│   │  Routes + Protocols ─── Anthropic, OpenAI, Gemini,  │   │
│   │                          Bedrock, Azure, ...         │   │
│   └──────────────────────────┬──────────────────────────┘   │
│                              │ HTTPS                        │
│                        Provider APIs                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Package Dependency Graph

```
schema              ← leaf: no opencode deps
  ↑
llm                 ← schema (narrow import for ToolContent)
  ↑
protocol            ← schema (endpoint types)
  ↑
core                ← llm, schema, effect-drizzle-sqlite, effect-sqlite-node,
  │                   plugin (tool types), sdk (client generation)
  ↑
server              ← core, protocol
  ↑
sdk                 ← schema (generated types), protocol (API contract)
  ↑
plugin              ← sdk (client types)
  ↑
tui                 ← core, plugin, sdk, ui
  ↑
[root CLI]          ← core, sdk, server, tui
```

**Key design principle:** Every package exposes its `exports` as raw `.ts` files. There are no per-package compilation steps. Bun resolves and bundles everything from TypeScript source in a single `Bun.build()` call when building the final binary.

---

## The Effect Framework

Everything in opencode's runtime is built on the [Effect](https://effect.website) library. Key concepts:

| Concept | What it means in this codebase |
|---|---|
| `Effect<A, E, R>` | A computation yielding `A`, failing with `E`, requiring context `R` |
| `Layer<A, E, R>` | A recipe to construct service `A` from requirements `R` |
| `Context.Service` | A typed service tag used to access a service from the Effect context |
| `Schema` | Runtime type validation + TypeScript inference in one |
| `Stream<A, E>` | An async sequence of `A` values (used for SSE, LLM streaming) |
| `Schedule` | Retry/polling policies (used by Daemon health checks, cleanup jobs) |
| `Scope` | Managed resource lifetime (services are acquired/released via Scope) |
| `Semaphore` | Concurrency control (DB writes, tool settlement, file mutations) |

---

## Runtime Topology

When you run `opencode`:

1. **CLI starts** (`src/index.ts`) — wires the command tree and provides the `Daemon.layer`.
2. **Daemon checks** (`src/services/daemon.ts`) — reads `$STATE_DIR/server.json`, pings `/v2/health`. If a healthy, version-compatible server is already running, uses it. Otherwise:
3. **Daemon spawns server** — forks `opencode serve --register` as a detached child process, polls for it to become healthy (up to 5 seconds).
4. **TUI launches** (`src/tui.ts`) — connects to the server URL via the SDK client, opens an SSE stream for real-time events, renders the full terminal UI.

Meanwhile the **server process** (`packages/server`):
- Boots the Effect runtime with all core service layers (Database, EventV2, SessionV2, Catalog, etc.)
- Serves the HTTP API
- Manages multiple concurrent `SessionRunner` fibers (one per active session turn)

---

## Data Persistence

All persistent state lives in:

| Path | Content |
|---|---|
| `~/.local/share/opencode/opencode.db` | Main SQLite database (sessions, events, messages, credentials, permissions) |
| `~/.local/share/opencode/server.json` | Running server registration (URL + PID) |
| `~/.local/share/opencode/password` | Server authentication password (base64url, 600 permissions) |
| `~/.local/share/opencode/kv.json` | TUI key-value store (theme, recent models, pinned sessions) |
| `~/.local/share/opencode/tool-output/` | Truncated tool output files (for outputs > 50 KB / 2000 lines) |
| `~/.local/share/opencode/snapshot/<projectID>/` | Shadow git repo for file snapshots (revert support) |
| `~/.config/opencode/` | Global config (`opencode.json`), global `AGENTS.md` |
| `<project>/opencode.json` | Project-level config |
| `<project>/.opencode/` | Project-level config directory (alternative) |

---

## Event Sourcing

The session system is **event-sourced**. Every mutation to a session (prompt submitted, LLM text streamed, tool called, agent switched, etc.) is recorded as a durable event in `EventTable` (SQLite). The `SessionMessageTable` and `SessionTable` are **projections** rebuilt by applying events in order.

This means:
- Sessions can be replayed from scratch
- The TUI receives real-time updates via SSE event streaming
- Revert operations work by replaying events up to a boundary point
- Multiple clients can subscribe to the same session's event stream concurrently

---

## Location vs. Global Services

The core engine distinguishes two service scopes:

- **Global services** — singletons for the lifetime of the server process: `Database`, `EventV2`, `SessionV2`, `Credential`, `ProjectV2`, `ApplicationTools`, `PermissionSaved`
- **Location services** — one instance per active working directory: `Config`, `Catalog`, `AgentV2`, `Integration`, `ToolRegistry`, `Snapshot`, `SkillV2`, `Reference`, `PluginV2`, `SessionRunner`

A "Location" is a `{ directory, project }` pair. The server creates one Location layer stack per directory that has an active session, and tears it down when idle.

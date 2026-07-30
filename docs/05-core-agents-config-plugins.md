# Chapter 5: Core — Agents, Configuration, and Plugins

**Source:** `packages/core/src/`

---

## Agent System — `agent.ts`

Agents are named AI personas with specific system prompts, tool permissions, model preferences, and step limits.

### Agent Schema

```ts
interface AgentInfo {
  id: string               // e.g. "coder", "researcher", "my-agent"
  mode: "build" | "subagent"  // subagents are not user-selectable
  hidden: boolean          // if true, not shown in agent picker
  system?: string          // system prompt override (appended after base)
  permissions?: PermissionRuleset  // { action: { allow, deny } } rules
  steps?: number           // max tool-call steps per turn (default: unlimited)
  model?: Model.Ref        // model override for this agent
}
```

### Built-In Agents

- **`coder`** — the default general-purpose coding agent
- Built-in agents are registered via `packages/core/src/plugin/agent.ts` at layer startup

### Custom Agents (via config)

```jsonc
// opencode.json
"agents": [
  {
    "id": "reviewer",
    "system": "You are a code reviewer. Only read files; never modify them.",
    "permissions": {
      "edit": { "deny": ["*"] },
      "write": { "deny": ["*"] },
      "bash": { "deny": ["*"] }
    }
  }
]
```

### `AgentV2.Service` API

| Method | Description |
|---|---|
| `get(id)` | Lookup agent by ID |
| `default()` | Default agent (from config, or "coder", or first non-subagent) |
| `resolve(id?)` | Resolve by ID or fall back to default |
| `all()` | All registered agents |
| `select(id?)` | Returns `{ id, info }` pair; info may be undefined for unknown IDs |

### Agent Selection Per Turn

In `SessionRunner`, agent selection happens per turn:
1. Read `session.agent` from the current session row
2. Call `AgentV2.select(session.agent)` → get agent info
3. Apply agent's system prompt, permissions, and step limit to the LLM request

---

## Configuration System — `config.ts`

`Config.Service` (Location-scoped) loads and merges configuration from multiple sources.

### Loading Priority (lowest to highest)

1. `~/.config/opencode/opencode.json` — global user config
2. Project root `opencode.json` / `opencode.jsonc`
3. Any `opencode.json` in parent directories (up to project root)
4. `.opencode/opencode.json` in any parent directory

All files are parsed and returned as a priority-ordered list of `Entry[]` objects. `Config.latest(entries, key)` returns the last non-undefined value — higher-priority configs override lower ones.

### Full Config Schema (selected fields)

```ts
interface ConfigInfo {
  shell?: string                    // shell path for BashTool
  model?: string                    // default model "providerID/modelID"
  default_agent?: string            // default agent ID
  autoupdate?: boolean

  // Provider API key overrides (alternative to env vars)
  providers?: Record<ProviderID, {
    env?: string                    // env var name
    apiKey?: string                 // literal key
    options?: Record<string, unknown>
  }>

  // Agent definitions
  agents?: AgentConfig[]

  // Per-provider/model permission policies
  permissions?: Record<string, PermissionRuleset>

  // MCP server config
  mcp?: {
    servers?: Record<string, MCPServerConfig>
    disabled?: string[]
  }

  // Skill sources
  skills?: Array<string | SkillSourceConfig>

  // Named file references
  references?: ReferenceConfig[]

  // Instruction files/directories
  instructions?: string | string[]

  // Plugin specs
  plugins?: Array<string | [string, PluginOptions]>

  // Tool output limits
  tool_output?: { max_lines?: number; max_bytes?: number }

  // Compaction settings
  compaction?: {
    auto?: boolean
    buffer?: number       // token buffer before compaction (default: 20000)
    keep?: { tokens?: number }  // tokens to keep after compaction (default: 8000)
  }

  // File snapshot settings
  snapshots?: boolean

  // LSP integration
  lsp?: Record<string, LSPConfig>

  // Formatter config
  formatter?: Record<string, FormatterConfig>

  // Watcher config
  watcher?: WatcherConfig

  // Experimental features
  experimental?: Record<string, unknown>
}
```

### V1 Config Migration

If a V1 `opencode.json` is detected (different schema), `ConfigMigrateV1.migrate` converts it to V2 format automatically.

---

## Plugin System — `plugin.ts`

Plugins extend opencode with custom tools, agents, providers, skills, references, commands, and auth methods.

### Plugin Types

**Server plugins** (`packages/plugin/src/index.ts`): Run on the server side. Receive a `PluginInput` with the SDK client, project info, and shell access.

**TUI plugins** (`packages/plugin/src/tui.ts`): Run in the terminal UI. Receive `TuiPluginApi` with the full TUI API surface.

**v2 Effect plugins** (`packages/plugin/src/v2/effect/`): Use Effect types for async operations. For advanced server-side extension.

### Plugin Loading — `plugin.ts`

`PluginV2.Service` manages plugin lifecycle:

```ts
// Add a plugin (by ID + Effect factory)
yield* PluginV2.add("my-plugin", Effect.gen(function* () {
  const host = yield* PluginHost.make  // access to all extension APIs
  // register tools, agents, skills, etc.
}))

// Remove a plugin
yield* PluginV2.remove("my-plugin")
```

All state transforms from one plugin load are batched into a single reload via `State.batch`.

### Plugin Host — `plugin/host.ts`

`PluginHost.make` creates a `PluginContext` with access to all registries:

```ts
// Available on the host
host.agent.transform(draft => {
  draft.update("my-agent", agent => ({ ...agent, system: "Custom system" }))
})

host.catalog.transform(draft => {
  draft.provider.update("my-provider", p => ({
    ...p,
    models: { ...p.models, "my-model": modelInfo }
  }))
})

host.integration.transform(draft => {
  draft.update("my-provider", integration => ({
    ...integration,
    methods: [...integration.methods, { type: "key", label: "API Key" }]
  }))
})

host.skill.transform(draft => {
  draft.source({ type: "directory", path: "/my/skills" })
})

host.reference.transform(draft => {
  draft.add({ name: "my-ref", path: "/some/directory" })
})

host.command.transform(draft => {
  draft.update("/mycommand", () => ({
    description: "Does something",
    run: async (input) => { /* ... */ }
  }))
})
```

### State System — `state.ts`

`State.create<S, DraftApi>(options)` is the reactive mutable state used by all registries:

- **`transform(callback)`** — registers a scoped transform. Opens a `DraftApi` view, calls `callback(draft)`. Closing the scope removes the transform and triggers reload. This is how plugins register their extensions: the transform runs on every reload, and the scope cleanup removes it.
- **`reload()`** — replays all active transforms from `initial()`, applies the optional `finalize` hook, notifies subscribers.
- **`batch(effect)`** — groups multiple finalizations into a single reload (used during plugin load to prevent intermediate state flickers).

This means plugin state is always consistent: removing a plugin (closing its scope) automatically removes all its transforms from all registries in a single atomic reload.

---

## Catalog — Provider and Model Registry — `catalog.ts`

`Catalog.Service` (Location-scoped) maintains the registry of all providers and their models.

### Catalog State

```ts
type CatalogState = Map<ProviderID, {
  provider: Provider.Info    // id, api, name, etc.
  models: Map<ModelID, Model.Info>  // all models for this provider
}>
```

### Key Queries

| Method | Returns |
|---|---|
| `provider.available()` | Providers with at least one active connection (API key or OAuth) |
| `model.available()` | Models whose provider is available and `enabled = true` |
| `model.default()` | The configured default model, or the most recent available one |
| `model.small(providerID)` | Cheapest "small" model for a provider (nano/flash/mini/haiku/small pattern) |
| `model.get(providerID, modelID)` | Single model lookup |

### Provider Plugins — `plugin/provider/`

Each provider (anthropic, openai, google, azure, amazon-bedrock, openrouter, github-copilot, xai, cloudflare, etc.) is a Location-scoped layer that:
1. Transforms the Catalog state to add its provider info and model list
2. Transforms the Integration state to register auth methods (env var, API key, OAuth)
3. Optionally hooks AISDK to install custom language model factories

The model list for each provider includes all models with their capabilities, costs, context limits, and API type tags.

---

## Integration & Auth — `integration.ts`

`Integration.Service` (Location-scoped) manages provider authentication.

### Connection Types

| Type | Source |
|---|---|
| `env` | Environment variable (e.g. `ANTHROPIC_API_KEY`) |
| `key` | User-stored API key (in `Credential` SQLite table) |
| `oauth` | OAuth flow (stored access + refresh tokens in `Credential`) |

### OAuth Flow

```ts
// Start OAuth attempt
const attempt = yield* Integration.connection.oauth({
  integrationID: "github-copilot",
  method: "device-flow",
  mode: "auto",        // "auto" starts the flow immediately; "manual" returns URL
})
// attempt.url → open in browser
// attempt.instructions → display to user

// Complete after user authorizes
yield* attempt.complete({ code: "..." })  // for code-mode
// or just wait (for auto-mode with polling)
```

OAuth tokens are automatically refreshed when they expire within 5 minutes.

### Credential Storage — `credential.ts`

`Credential.Service` (global, SQLite-backed) stores per-integration credentials:

```ts
// API key
{ type: "key", value: "sk-ant-..." }

// OAuth token
{ type: "oauth", access: "...", refresh: "...", expires: 1234567890, methodID: "device-flow" }
```

Credentials are never exposed to the LLM or plugins; they flow only through the Integration → SessionRunnerModel → LLM Route auth chain.

---

## Project Resolution — `project.ts`

`ProjectV2.Service` (global) resolves a directory to a project identity:

1. Run `git rev-parse --git-dir` to find the git repository
2. If no git repo: project ID is a stable global ID, directory is filesystem root
3. If git repo: project ID determined by priority:
   - Remote URL hash (most stable; same project across clones)
   - Cached ID in `<commonDirectory>/opencode` file
   - Root commit hash (fallback for repos without remotes)

The project ID is stable across machines for the same repository (when using remote URL hashing), enabling future sync features.

---

## Skill System — `skill.ts`

Skills are named markdown documents with associated files that can be loaded into conversation by the AI (via `SkillTool`) or by the user (via slash commands).

### Skill Sources

| Type | Config |
|---|---|
| `directory` | Local directory path, auto-discovers `*.md` and `**/SKILL.md` files |
| `url` | Downloads a zip archive from a URL, caches locally |
| `embedded` | Built-in skill (defined in code) |

### Skill Metadata

YAML frontmatter in skill markdown files:

```yaml
---
name: My Skill
description: What this skill does
slash: /my-skill    # makes it available as a slash command
---
```

### Skill Guidance

`SkillGuidance.Service` injects a `"core/skill-guidance"` system context listing available skills (those with descriptions). Formatted as XML tags for the model. This tells the AI what skills it can use without sending the full skill content in every turn.

---

## Reference System — `reference.ts`

Named external directories that can be referenced in conversations.

```jsonc
// opencode.json
"references": [
  { "name": "docs", "path": "./docs" },
  { "name": "api-types", "git": "https://github.com/example/types.git" }
]
```

Git references are cloned/fetched into `~/.local/share/opencode/repos/` on first use.

`ReferenceGuidance.Service` injects a `"core/reference-guidance"` system context listing references with descriptions, so the AI knows what external codebases are available.

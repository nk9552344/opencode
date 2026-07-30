# Chapter 8: Terminal UI (TUI)

**Source:** `packages/tui/`

The TUI is a reactive terminal application built on SolidJS rendered via `@opentui/core` (a custom terminal rendering engine). It is completely separate from the HTTP server — it communicates with the server via the SDK client over HTTP/SSE.

---

## Rendering Engine — `@opentui/core` + `@opentui/solid`

`@opentui/core` is a retained-mode terminal renderer. It maintains a virtual scene graph of cells, handles ANSI escape sequences, and produces efficient terminal output. `@opentui/solid` is the SolidJS adapter — it bridges SolidJS's fine-grained reactivity to the terminal renderer.

`createCliRenderer()` — creates the renderer instance. The `@opentui/solid/preload` module (loaded via `bunfig.toml`) sets up the SolidJS runtime environment before any code runs.

---

## Entry Point — `src/app.tsx`

```ts
export type TuiInput = {
  url: string           // server URL
  args: CliArgs         // parsed CLI arguments
  config: TuiConfig.Resolved
  onSnapshot?: (session: Session.Info) => void
  directory?: string
  fetch?: typeof fetch
  headers?: Headers
  events?: EventEmitter
  pluginHost: TuiPluginHost
}

export const run: Effect<void>
```

`run` creates the renderer, registers the keymap, renders the provider tree into the terminal, and awaits a shutdown signal.

---

## Provider Hierarchy (outermost → innermost)

```
ExitProvider              → shutdown signal (Deferred)
  EpilogueProvider        → text printed to stdout on exit
    ErrorBoundary         → catches render errors
      TuiPathsProvider    → { cwd, home, state, worktree } paths
        TuiTerminalEnvironmentProvider  → { platform, multiplexer?, displayServer? }
          TuiStartupProvider  → { initialRoute?, skipInitialLoading }
            ClipboardProvider   → platform clipboard read/write
              OpencodeKeymapProvider  → keymap instance (from @opentui/keymap)
                ArgsProvider          → parsed CLI args ({ model, agent, prompt, ... })
                  KVProvider          → persistent JSON KV store
                    ToastProvider     → toast notification queue
                      RouteProvider   → current route (home/session/plugin)
                        TuiConfigProvider      → resolved tui.json config
                          PluginRuntimeProvider  → plugin slot registry
                            SDKProvider          → HTTP client + SSE event emitter
                              PermissionProvider → permission mode (auto/normal)
                                ProjectProvider  → project/workspace state
                                  SyncProvider   → full reactive data store
                                    DataProvider → V2 event-driven live updates
                                      ThemeProvider  → active theme + syntax styles
                                        LocalProvider  → model/agent/session local state
                                          PromptStashProvider
                                            DialogProvider
                                              FrecencyProvider
                                                PromptHistoryProvider
                                                  PromptRefProvider
                                                    EditorContextProvider  → IDE connection
                                                      LocationProvider
                                                        App   ← main component
```

---

## App Component

`App` (innermost component):

1. Creates `TuiAttentionHost` (`createTuiAttention`) — notifications and sound
2. Assembles `TuiPluginApi` via `createTuiApiAdapters` + `createTuiApi`
3. Calls `pluginHost.start({ api, config, runtime, dispose })` — loads all plugins
4. Registers global keymap commands (35+ bindings: session navigation, model/agent switching, command palette, theme, help, etc.)
5. Subscribes to SDK events: `tui.command.execute`, `tui.toast.show`, `tui.session.select`, `session.deleted`, `session.error`, `installation.update-available`
6. Renders `<Home />` or `<Session />` based on route, plus plugin slots `app_bottom` and `app`

---

## Route System — `src/context/route.tsx`

```ts
type HomeRoute   = { type: "home";    prompt?: string }
type SessionRoute = { type: "session"; sessionID: string; prompt?: string }
type PluginRoute  = { type: "plugin";  id: string; data?: unknown }
```

`useRoute()` → `{ data: RouteStore, navigate(route) }`.

`useRouteData<T>(type)` — typed accessor; returns the route data only when the current route matches `type`.

Initial route comes from `TuiStartupProvider` (which reads `OPENCODE_ROUTE` env var or derives from CLI args).

---

## Data Flow — Sync and Data Contexts

### `SyncProvider` — `src/context/sync.tsx`

The primary reactive data store. All server state lives here.

**Fields:** `status`, `provider`, `provider_default`, `capabilities`, `provider_auth`, `agent`, `command`, `permission`, `question`, `config`, `session`, `session_status`, `session_diff`, `todo`, `message`, `part`, `lsp`, `mcp`, `mcp_resource`, `formatter`, `vcs`, `console_state`

**Bootstrap:** Two-phase load:
- *Blocking phase* (before rendering content): providers, agents, config, project, and optionally sessions (for `--continue`)
- *Non-blocking phase* (after render): sessions, commands, lsp, mcp, resources, formatter, session_status, provider_auth, vcs, workspace

`session.sync(sessionID)` — full sync with message hydration. Prevents a race condition where real-time SSE events could arrive before the initial bulk message list.

Auto-replies to permission requests when in `"auto"` mode.

### `DataProvider` — `src/context/data.tsx`

Translates live V2 session events (received over SSE) into reactive state updates.

**Session events handled:** `agent.switched`, `model.switched`, `prompted`, `shell.*`, `step.*`, `text.*`, `tool.*`, `reasoning.*`, `compaction.ended`, `catalog.updated`, `reference.updated`, `integration.updated`

**API surface:**
```ts
{
  session: { get(id), refresh(id), message(id, msgId), permission(id), question(id) }
  project: { permission }
  location: {
    default, agent, command, integration, model, provider, reference, skill
    // each: { list(), refresh() }
  }
}
```

---

## KV Store — `src/context/kv.tsx`

Persists to `{state}/kv.json` with file locking (flock).

```ts
const kv = useKV()
kv.get<T>(key, default?)          // read
kv.set(key, value)                // write
kv.signal<T>(name, default)       // [accessor, setter] reactive signal
kv.ready                          // true after initial load
```

Used throughout for UI state that should persist across sessions: theme choice, tips visibility, pinned sessions, attention sound pack, thinking mode, etc.

---

## Local State — `src/context/local.tsx`

Wraps KV-persisted state with typed reactive subsystems:

### Model Subsystem

```ts
local.model.current()              // selected Model.Ref
local.model.recent()               // recently used models (up to 10)
local.model.set(model, opts?)      // update current
local.model.cycle(dir)             // cycle through available
local.model.cycleFavorite(dir)     // cycle through favorites
local.model.toggleFavorite(model)  // favorite/unfavorite
local.model.variant.{selected, current, list, set, cycle}
```

Persisted to `{state}/model.json`.

### Agent Subsystem

```ts
local.agent.list()       // all available agents
local.agent.current()    // current agent
local.agent.set(name)    // change agent
local.agent.move(dir)    // cycle through agents
local.agent.color(name)  // stable color derived from agent name hash
```

### Session Subsystem

```ts
local.session.pinned()          // [SessionID, ...] ordered pins
local.session.slots()           // up to 9 ordered pin slots
local.session.isPinned(id)
local.session.togglePin(id)
local.session.quickSwitch(slot) // switch to pinned session at slot 1–9
```

Persisted to `{state}/session.json`.

---

## Theme System — `src/context/theme.tsx` + `src/theme/index.ts`

### Theme Schema

56 `RGBA` color fields organized into:

| Group | Fields |
|---|---|
| Base | `primary`, `secondary`, `accent`, `error`, `warning`, `success`, `info` |
| Text | `text`, `textMuted`, `selectedListItemText` |
| Background | `background`, `backgroundPanel`, `backgroundElement`, `backgroundMenu` |
| Border | `border`, `borderActive`, `borderSubtle` |
| Diff | 12 diff colors (added/removed/context backgrounds, highlights, line numbers) |
| Markdown | 13 colors |
| Syntax | 9 colors (comment, keyword, function, variable, string, number, type, operator, punctuation) |

Plus: `thinkingOpacity: number` (default 0.6) and `_hasSelectedListItemText: boolean`.

### Bundled Themes (35)

aura, ayu, carbonfox, catppuccin (mocha/macchiato/latte), cobalt2, cursor, dracula, everforest, flexoki, github, gruvbox, kanagawa, lucent-orng, material, matrix, mercury, monokai, nightowl, nord, one-dark, opencode, orng, osaka-jade, palenight, rosepine, solarized, synthwave84, tokyonight, vercel, vesper, zenburn.

### System Theme

`generateSystem(colors, mode)` — auto-generates a theme from the terminal's 16-color ANSI palette (queried via `renderer.getPalette({ size: 16 })`). This makes opencode match the terminal's color scheme without any configuration.

### Custom Themes

Discovered from:
- `{config}/themes/*.json`
- `{cwd}/.opencode/themes/*.json` (walks up directory tree)

Theme JSON format: object mapping color field names to hex strings, color references (`"@other-theme/color-field"`), or RGBA arrays.

### Refresh Trigger

Theme reloads on `SIGUSR2` or xterm OSC sequences `\x1b[?997;1n` (dark) / `\x1b[?997;2n` (light).

### Syntax Highlighting

`generateSyntax(theme)` → `SyntaxStyle` — 60+ TreeSitter scope rules covering all supported languages.

`generateSubtleSyntax(theme, overrides?)` — same but applies `thinkingOpacity` alpha to all foreground colors (used for reasoning/thinking content).

`createSyntaxStyleMemo(factory)` — memo that defers destruction on idle (prevents flicker during model switches).

### Supported Languages (via Tree-sitter WASM parsers)

python, rust, go, cpp, csharp, bash, c, java, kotlin, ruby, php, scala, html, vue, hcl, json, yaml, haskell, css, julia, lua, ocaml, clojure, swift, toml, nix, diff/patch, elixir, fsharp, r, make, vim, xml, agda. (JS/TS/Markdown use built-in opentui parsers.)

---

## Keymap System — `src/keymap.tsx`

Built on `@opentui/keymap`. The TUI uses a leader-key model:

- **Leader key:** `Ctrl+X` (default, configurable)
- **Leader timeout:** configurable ms
- **LEADER_TOKEN:** `"leader"` — used in binding strings like `"<leader>n"`

### Key Binding Syntax

```
"ctrl+c"
"<leader>n"        → leader then n
"ctrl+p,ctrl+p"   → sequence of two chords
```

### Mode Stack

`createOpencodeModeStack(keymap)` — a stack of active mode names. `push(mode)` returns cleanup. Modes are used to namespace bindings (e.g. `"prompt"`, `"session"`, `"dialog"`).

### Hooks

```ts
useBindings(factory)        // register commands + bindings in current scope
useCommandShortcut(command) // reactive formatted shortcut string for a command
useCommandSlashes()         // all commands with slashName (for slash command list)
useLeaderActive()           // true while leader key is pending
useOpencodeKeymap()         // keymap instance
useOpencodeModeStack()      // mode stack instance
```

### Standard Keybinds (from `tui.json` or defaults)

| Keybind ID | Default | Command |
|---|---|---|
| `leader` | `Ctrl+X` | activates leader |
| `app_exit` | `Ctrl+C`, `Ctrl+D`, `<leader>q` | `app.exit` |
| `session_new` | `<leader>n` | `session.new` |
| `session_list` | `<leader>l` | `session.list` |
| `model_list` | `<leader>m` | `model.list` |
| `agent_list` | `<leader>a` | `agent.list` |
| `theme_list` | `<leader>t` | `theme.switch` |
| `command_palette` | `<leader>p` / `Ctrl+P` | `command.palette.show` |
| `help` | `<leader>?` | `help.show` |
| `quick_switch_1`–`9` | `<leader>1`–`9` | `session.quick_switch.{1-9}` |

35 textarea-specific input commands (undo, redo, cut, copy, select-all, cursor movement, etc.) are registered on the `"textarea"` mode layer.

---

## Plugin System — `src/plugin/`

### Slot System — `src/plugin/slots.tsx`

Plugins render content into named slots:

| Slot Name | Location |
|---|---|
| `home_logo` | Home screen logo area |
| `home_prompt` | Home screen prompt area |
| `home_prompt_right` | Right of home prompt |
| `home_bottom` | Below home prompt |
| `home_footer` | Home screen footer |
| `session_prompt` | Session prompt area |
| `session_prompt_right` | Right of session prompt |
| `sidebar_title` | Sidebar title |
| `sidebar_content` | Sidebar main area (multiple plugins, ordered by `order` number) |
| `sidebar_footer` | Sidebar footer |
| `app_bottom` | Bottom of entire app |
| `app` | Full-app overlay |

### `TuiPluginHost` — `src/plugin/runtime.tsx`

```ts
interface TuiPluginHost {
  start(input: { api, config, runtime, dispose? }): Promise<void>
  dispose(): Promise<void>
}
```

Plugin commands: `{ activate, deactivate, add, install }`.

### `TuiPluginApi`

The full API surface exposed to plugins:

```ts
{
  route: { navigate(name) }
  ui: { DialogAlert, DialogConfirm, ... }   // SolidJS dialog components
  state: {                                   // reactive server state
    ready, config, provider, path, vcs,
    session: { list, get, create, prompt, messages, ... }
    part(), lsp(), mcp()
  }
  kv: KvContext                              // persistent KV store
  client: SDK client                         // raw HTTP client
  theme: ThemeContext
  attention: TuiAttentionHost               // notifications + sound
  keymap: OpencodeKeymap
  mode: ModeStack
  keys: { formatSequence, formatBindings }
  tuiConfig: TuiConfig.Resolved
  plugins: { list, activate, deactivate, add, install }
  app: { version: string }
  event: EventEmitter                        // filtered to current workspace
}
```

### Builtin Plugins — `src/feature-plugins/builtins.ts`

| Plugin | Slot | Description |
|---|---|---|
| `HomeFooter` | `home_footer` | Directory + branch, MCP status, version |
| `HomeTips` | `home_bottom` | Rotating usage tips (89+ tips) |
| `SidebarContext` | `sidebar_content` (order 100) | Token count, context %, cost |
| `SidebarMcp` | `sidebar_content` (order 200) | MCP server list with status |
| `SidebarLsp` | `sidebar_content` (order 300) | LSP server list with status |
| `SidebarTodo` | `sidebar_content` (order 400) | Active todos from session |
| `SidebarFiles` | `sidebar_content` (order 500) | Modified files with diff stats |
| `SidebarFooter` | `sidebar_footer` | Getting-started, branding, path+branch |
| `Notifications` | (system) | OS notifications + sound on events |
| `PluginManager` | (system) | Plugin install/activate/deactivate |
| `WhichKey` | (disabled by default) | Leader key popup |
| `DiffViewer` | (system) | Diff viewing integration |

---

## Attention System — `src/attention.ts`

`createTuiAttention({ renderer, config, kv?, audio? }): TuiAttentionHost`

- Tracks terminal focus state: `"unknown" | "focused" | "blurred"`
- `notify(request)` — fires OS notification (via `node-notifier`) and/or sound if terminal is blurred or focus state is unknown
- Sound triggers: `question.asked` → `"question"`, `permission.asked` → `"permission"`, session idle after busy → `"done"` or `"subagent_done"`, session error → `"error"`

### Sound System — `src/audio.ts`

Uses `@opentui/core` Audio engine. 6 named sounds per pack: `default`, `question`, `permission`, `error`, `done`, `subagent_done`.

```ts
loadSoundFile(file)          // reads + caches
play(sound, options?)        // lazy-starts engine
stopVoice(voice)
dispose()
```

KV key: `"attention_sound_pack"`. Default pack: `"opencode.default"`.

---

## IDE Integration — `src/context/editor.ts`

WebSocket connection to the IDE (VS Code, JetBrains, Zed) using MCP protocol version `"2025-11-25"`.

Port detection: `CLAUDE_CODE_SSE_PORT` or `OPENCODE_EDITOR_SSE_PORT` env vars. Zed detected via `ZED_TERM=true` or `TERM_PROGRAM=zed`.

**`EditorSelection`:** `{ filePath, ranges[]: { text, selection: { start, end }: { line, character } } }`
**`EditorMention`:** `{ filePath, startLine?, endLine? }`

`useEditorContext()` exposes: `selection()`, `labelState()`, `clearSelection()`, `markSelectionSent()`, `preserveSelectionFromNewSession()`.

Reconnects on directory change with exponential backoff (up to 10s).

---

## Clipboard — `src/clipboard.ts`

Multi-platform:

| Platform | Image Read | Write |
|---|---|---|
| macOS | `osascript` | AppleScript |
| Win32/WSL | PowerShell | `Set-Clipboard` |
| Linux Wayland | `wl-paste` | `wl-copy` |
| Linux X11 | `xclip` | `xclip`/`xsel` |
| Fallback | — | `clipboardy` npm |

`writeOsc52(text)` — writes OSC 52 escape sequence (works in tmux/mux terminals).

---

## TUI Configuration — `src/config/`

`tui.json` (in `~/.config/opencode/` or project root `.opencode/`) controls TUI-specific settings:

```jsonc
{
  "$schema": "...",
  "theme": "catppuccin-mocha",
  "leader_timeout": 500,
  "keybinds": {
    "session_new": "<leader>n",
    "model_list": "<leader>m"
  },
  "prompt": {
    "max_height": 20,
    "placeholders": ["What can I help with?"]
  },
  "scroll_speed": 3,
  "scroll_acceleration": { "enabled": true },
  "diff_style": "stacked",
  "mouse": true,
  "attention": {
    "sound_pack": "opencode.default"
  }
}
```

`TuiConfig.resolve(input, options)` fills defaults and returns a `TuiConfig.Resolved` with all fields present.

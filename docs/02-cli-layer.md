# Chapter 2: CLI Layer

**Source:** `src/`

The CLI layer is the thin process that users interact with directly. It wires the command tree together, manages the background server lifecycle, and launches the TUI. It is deliberately minimal — all business logic lives in `packages/core`.

---

## Entry Point — `src/index.ts`

The file is a `#!/usr/bin/env bun` script. Its job is to:

1. **Build the handler table** — calls `Runtime.handlers(Commands, { lazy imports })` to traverse the spec tree and associate each command leaf with a dynamic `import()` factory.
2. **Execute** — calls `Runtime.run(Commands, Handlers, { version: "local" })` to produce the top-level Effect.
3. **Provide services** — pipes through `Daemon.layer` (the background server manager) and `NodeServices.layer` (Effect's Node platform services), then runs via `NodeRuntime.runMain`.

All handler modules are loaded lazily — nothing besides the routing table is imported at startup. This keeps cold-start time near zero.

---

## Command Spec — `src/framework/spec.ts`

`Spec.make` is a thin wrapper around `effect/unstable/cli/Command` that:
- Adds type-level tracking of child commands (for TypeScript inference)
- Keeps handler logic entirely separate from the command declaration

```ts
// Minimal example of how commands are declared
const myCommand = Spec.make("mycommand", {
  description: "Does something",
  options: { flag: Flag.boolean("verbose").pipe(Flag.withAlias("v")) },
  args: Argument.text("input"),
  commands: [childCommand],
})
```

The resulting `Spec.Node<Name, Spec, Children>` type carries the full tree structure at compile time, enabling the runtime to traverse it generically.

---

## Runtime Wiring — `src/framework/runtime.ts`

Three exported functions:

### `Runtime.handler(node, run)`

A type-safe factory used inside handler modules:

```ts
// Inside src/commands/handlers/my-handler.ts
export default Runtime.handler(Commands.commands.myCommand, (input) =>
  Effect.gen(function* () {
    // input is fully typed to the command's parsed values
    const daemon = yield* Daemon.Service
    // ...
  })
)
```

### `Runtime.handlers(root, handlerMap)`

Recursively traverses the spec tree paired with a nested handler map, producing a flat `LazyHandler[]`. Each `LazyHandler` holds the `spec` (a `Command.Command` object for identity matching) and a `load` function (calls `import()` when the command is invoked).

### `Runtime.run(commands, handlers, options)`

Attaches handlers to the command tree via `Command.withSubcommands`, then executes via `Command.run`. Supports `--help` and `--version` automatically.

---

## Command Declarations — `src/commands/commands.ts`

Declares the full command tree using `Spec.make`. The binary name is controlled by the `OPENCODE_CLI_NAME` compile-time constant (default `"opencode"`).

```
opencode                         → default handler (TUI)
  api                            → API client
    --data / -d <string>         
    --header / -H <string>       (repeatable, up to 100)
    --param <key=value>          
    <request: 1–2 args>          operationID or "METHOD /path"
  debug
    agents                       → list agents as JSON
  migrate                        → (placeholder, no-op)
  service
    start
    restart
    status
    stop
    password [value]
  serve                          → internal: start HTTP server in-process
    --hostname <string>          (default: 127.0.0.1)
    --port <integer>
    --register <boolean>
```

---

## Daemon Service — `src/services/daemon.ts`

The Daemon manages the lifecycle of the background `opencode serve` process. All CLI commands that need the API acquire it through `Daemon.Service`.

### Interface

```ts
interface Interface {
  // Get a typed SDK client pointed at the running server
  client():    Effect<OpencodeClient>
  // Get the server URL + auth headers for the TUI
  transport(): Effect<{ url: string; headers: RequestInit["headers"] }>
  // Start the server (idempotent, health-checks first)
  start():     Effect<string>          // returns server URL
  // Current server URL if healthy, undefined if stopped
  status():    Effect<string | undefined>
  // Stop the server gracefully (SIGTERM → wait → SIGKILL)
  stop():      Effect<void>
  // Get or set the server password
  password(value?: string): Effect<string>
  // Used by `serve` handler to write registration and start watchdog
  register(address: HttpServer.Address): Effect<void, _, Scope>
}
```

### Registration File

The server writes `~/.local/share/opencode/server.json` when it starts (via `register(address)`):

```json
{
  "id": "01J...",
  "version": "1.18.9",
  "url": "http://127.0.0.1:4096",
  "pid": 12345
}
```

### Password

The password is stored in `~/.local/share/opencode/password` (mode `0600`). It is 32 random bytes encoded as base64url. The Daemon generates it on first use and reuses it across server restarts.

### Health Checking

`healthy()` GETs `/v2/health` with a 2-second timeout and checks `data.healthy === true`.

`compatible()` additionally checks that `info.version` matches the compiled-in `OPENCODE_VERSION`.

### Server Startup Flow

```
daemon.start()
  1. Read server.json → get existing URL + PID
  2. ping /v2/health → if healthy AND compatible → return existing URL
  3. If incompatible: daemon.stop() (SIGTERM → wait → SIGKILL)
  4. Spawn: opencode serve --register (detached, stdio ignored, unref'd)
  5. Poll compatible() up to 100 × 50ms = 5 seconds
  6. Return URL
```

### Self-Eviction Watchdog

When `serve --register` starts, it calls `daemon.register(address)`. This:

1. Writes `server.json` with its own PID.
2. Starts a scoped fiber that reads `server.json` every 10 seconds. If the registration no longer belongs to this process (newer server replaced it), sends `SIGTERM` to itself.
3. Registers a finalizer that removes `server.json` on scope exit (clean shutdown).

This ensures only one server instance claims the registration at a time.

---

## Command Handlers

### Default (TUI) — `src/commands/handlers/default.ts`

The most-used handler. Runs when you type `opencode` with no subcommand:

```ts
// Simplified
Effect.gen(function* () {
  const daemon = yield* Daemon.Service
  const transport = yield* daemon.transport()   // starts server if needed
  const { runTui } = yield* Effect.promise(() => import("../../tui"))
  yield* runTui(transport)                       // opens the TUI
})
```

### API — `src/commands/handlers/api.ts`

A cURL-style HTTP client for the running server. Resolves the endpoint by either:
- Treating `["METHOD", "/path"]` as a raw request
- Treating `["operationId"]` as an OpenAPI operation (fetches `/openapi.json` and resolves the path/method by `operationId`, interpolating `--param` values into `{placeholder}` path segments)

Supports `--header "Name: Value"` and `--data '{"key":"val"}'`. Output is written to stdout.

### Service Handlers — `src/commands/handlers/service/`

| File | What it does |
|---|---|
| `start.ts` | `daemon.start()` → print URL |
| `stop.ts` | `daemon.stop()` |
| `restart.ts` | `daemon.stop()` → `daemon.start()` → print URL |
| `status.ts` | `daemon.status()` → print `"running <url>"` or `"stopped"` |
| `password.ts` | If new value: `daemon.stop()` first (server must restart to pick it up). Then `daemon.password(value)` → print password |

### Serve — `src/commands/handlers/serve.ts`

Starts the HTTP API server in-process. Binds to the requested hostname/port, or auto-increments from 4096 until a port is free. If `--register` is true, calls `daemon.register(address)` to write `server.json` and start the self-eviction watchdog, then suspends via `Effect.never`.

### Debug Agents — `src/commands/handlers/debug/agents.ts`

Calls `client.v2.agent.list({ location: { directory: process.cwd() } })`, sorts by `id`, pretty-prints JSON.

### Migrate — `src/commands/handlers/migrate.ts`

Currently logs `"No migrations to run."` — a placeholder for future V1 → V2 data migration logic.

---

## TUI Launcher — `src/tui.ts`

`runTui(transport)` boots the terminal UI:

1. Calls `TuiConfig.resolve({}, { terminalSuspend: false })` to build TUI configuration.
2. Provides a `gracefulFetch` wrapper that stubs out legacy V1 API paths (for backward compatibility during server version transitions).
3. Calls `run({ ...transport, args: {}, config, fetch: gracefulFetch, pluginHost })` from `@opencode-ai/tui`.

The `pluginHost` is a no-op stub — plugin loading is handled inside the TUI package itself.

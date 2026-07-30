# Chapter 9: Schema and Protocol Layer

**Sources:** `packages/schema/` and `packages/protocol/`

`packages/schema/` contains all shared data types used across every package. `packages/protocol/` defines the HTTP API contract (endpoints, errors, middleware) using Effect's `HttpApi` framework.

---

## packages/schema — Shared Types

All types are defined as Effect Schemas, which provide:
- Runtime validation
- TypeScript type derivation
- JSON encode/decode
- OpenAPI schema generation

The barrel export (`packages/schema/src/index.ts`) re-exports every module: `Agent`, `Command`, `Connection`, `Credential`, `Event`, `FileSystem`, `Integration`, `LLM`, `Location`, `Model`, `Permission`, `PermissionSaved`, `Project`, `ProjectCopy`, `Provider`, `Prompt`, `Reference`, `Revert`, `Session`, `SessionInput`, `SessionMessage`, `Skill`, `Pty`, `PtyTicket`, `Question`, `Workspace`, and shared primitives.

---

## Primitive Types — `schema.ts`

```ts
PositiveInt       // int > 0
NonNegativeInt    // int >= 0
RelativePath      // branded string
AbsolutePath      // branded string
DateTimeUtcFromMillis  // Effect.DateTimeUtc from Unix ms number
```

---

## Identifier System — `identifier.ts`

Custom ascending/descending ID generator: millisecond timestamp + counter encoded as 6 hex bytes + 14 random base62 characters = 26 characters total.

| ID Type | Prefix | Order | Module |
|---|---|---|---|
| `Session.ID` | `ses` | descending | `session-id.ts` |
| `Workspace.ID` | `wrk` | ascending | `workspace-id.ts` |
| `Event.ID` | `evt_` | — | `event.ts` |
| `Permission.ID` | `per` | — | `permission.ts` |
| `PermissionSaved.ID` | `psv_` | — | `permission-saved.ts` |
| `Question.ID` | `que` | — | `question.ts` |
| `Credential.ID` | `cred_` | — | `credential.ts` |
| `Integration.AttemptID` | `con_` | — | `integration.ts` |
| `Pty.ID` | `pty` | — | `pty.ts` |
| `SessionMessage.ID` | `msg_` | — | `session-message.ts` |

---

## Core Domain Types

### Session — `session.ts`

```ts
Session.Info = {
  id: Session.ID
  parentID?: Session.ID
  projectID: Project.ID
  agent?: string
  model?: Model.Ref
  cost: number
  tokens: { input, output, reasoning, cache: { read, write } }
  time: { created: DateTimeUtc, updated: DateTimeUtc, archived?: DateTimeUtc }
  title: string
  location: Location.Ref
  subpath?: string
  revert?: Revert.State
}
```

`Session.ListAnchor = { id, time, direction: "previous" | "next" }` — pagination anchor for session list cursor.

---

### Session Messages — `session-message.ts`

All messages share `Base = { id: SessionMessage.ID, metadata?, time: { created } }`.

| Type | Extra Fields |
|---|---|
| `user` | `text`, `files?: FileAttachment[]`, `agents?: AgentAttachment[]` |
| `assistant` | `agent`, `model`, `content: AssistantContent[]`, `snapshot?`, `finish?`, `cost?`, `tokens?`, `error?`, `time: { created, completed? }` |
| `agent-switched` | `agent: string` |
| `model-switched` | `model: Model.Ref` |
| `compaction` | `reason: "auto"\|"manual"`, `summary: string`, `recent: string` |
| `synthetic` | `sessionID`, `text` |
| `system` | `text` |
| `shell` | `callID`, `command`, `output`, `time: { created, completed? }` |

**`AssistantContent`** union:
- `AssistantText` — `{ type: "text", id, text }`
- `AssistantReasoning` — `{ type: "reasoning", id, text, providerMetadata?, time? }`
- `AssistantTool` — `{ type: "tool", id, name, provider?, state: ToolState, time }`

**`ToolState`** (discriminated by `status`):
- `pending` — `{}`
- `running` — `{ input: unknown }`
- `completed` — `{ input, output: ToolOutput }`
- `error` — `{ input, output: ToolError }`

---

### Session Events — `session-event.ts`

Session events are the durable audit log. All durable events use `{ durable: { aggregate: "sessionID", version: 1 } }` (Step.Ended/Failed use version 2 for schema migration compatibility).

**Durable events** (written to SQLite, replayed on reconnect):

| Event Type | Key Payload Fields |
|---|---|
| `session.agent.switched` | `agent` |
| `session.model.switched` | `model: Model.Ref` |
| `session.moved` | `sessionID`, `directory` |
| `session.prompted` | `prompt: Prompt`, `delivery`, `messageID` |
| `session.prompt.admitted` | `sessionID`, `input: SessionInput.Admitted` |
| `session.context.updated` | `sessionID`, `text` |
| `session.synthetic` | `sessionID`, `text`, `messageID` |
| `session.shell.started/ended` | `callID`, `command`, `output` |
| `session.step.started` | `sessionID`, `messageID`, `snapshot` |
| `session.step.ended` | `sessionID`, `messageID`, `tokens`, `cost`, `finish`, `snapshot` |
| `session.step.failed` | `sessionID`, `messageID`, `error` |
| `session.text.started/ended` | `sessionID`, `messageID`, `partID`, `text?` |
| `session.reasoning.started/ended` | `sessionID`, `messageID`, `partID` |
| `session.tool.input.started/ended` | `sessionID`, `messageID`, `callID`, `name` |
| `session.tool.called` | `sessionID`, `messageID`, `callID`, `name`, `input` |
| `session.tool.progress` | `sessionID`, `messageID`, `callID`, `output` |
| `session.tool.success/failed` | `sessionID`, `messageID`, `callID`, `output`/`error` |
| `session.retried` | `sessionID`, `attempt`, `error` |
| `session.compaction.started/ended` | `sessionID`, `summary`, `recent`, `seq` |
| `session.revert.staged/cleared/committed` | `sessionID`, `state?: Revert.State` |

**Live-only events** (not written to SQLite, only broadcast via PubSub):

| Event | Description |
|---|---|
| `session.text.delta` | Streaming text chunk |
| `session.reasoning.delta` | Streaming reasoning chunk |
| `session.tool.input.delta` | Streaming tool input JSON chunk |
| `session.compaction.delta` | Streaming compaction summary chunk |

`SessionEvent.Durable` — a `Schema.Union` of all durable definitions (discriminated by `type`).

---

### Model — `model.ts`

```ts
Model.Ref = { id: Model.ID, providerID: Provider.ID, variant?: Model.VariantID }
Model.Capabilities = { tools: boolean, input: string[], output: string[] }
Model.Cost = { tier?: { type: "context", size: number }, input, output, cache: { read, write } }
Model.Info = {
  id, providerID, family?, name, api: Model.Api,
  capabilities: Capabilities,
  request: { headers, body, variant? }
  variants: VariantDef[]
  time: { released: DateTimeUtc }
  cost: Cost[]
  status: "alpha" | "beta" | "deprecated" | "active"
  enabled: boolean
  limit: { context: number, input?: number, output?: number }
}
```

---

### Provider — `provider.ts`

```ts
Provider.ID // branded string, predefined constants:
  // opencode, anthropic, openai, google, googleVertex, githubCopilot,
  // amazonBedrock, azure, openrouter, mistral, gitlab

Provider.AISDK = { type: "aisdk", package, url?, settings? }
Provider.Native = { type: "native", url?, settings }
Provider.Info = { id, integrationID?, name, disabled?, api: Provider.Api, request: Provider.Request }
```

---

### Agent — `agent.ts`

```ts
Agent.Info = {
  id: Agent.ID
  model?: Model.Ref
  request: Provider.Request      // extra headers/body to pass with all requests
  system?: string               // system prompt addition
  description?: string
  mode: "subagent" | "primary" | "all"
  hidden: boolean
  color?: Agent.Color           // "#rrggbb" or named semantic color
  steps?: number
  permissions: Permission.Ruleset
}
```

---

### Permission — `permission.ts`

```ts
Permission.Request = { id, sessionID, action, resources: string[], save?, metadata?, source? }
Permission.Source = { type: "tool", messageID, callID }
Permission.Rule = { action: string, resource: string, effect: "allow" | "deny" | "ask" }
Permission.Ruleset = Permission.Rule[]
Permission.Reply = "once" | "always" | "reject"
Permission.Effect = "allow" | "deny" | "ask"
```

Events: `Permission.Event.Asked` (`"permission.v2.asked"`), `Permission.Event.Replied` (`"permission.v2.replied"`).

---

### Question — `question.ts`

```ts
Question.Option = { label: string, description: string }
Question.Info = { question, header, options: Option[], multiple?: boolean, custom?: boolean }
Question.Request = { id, sessionID, questions: Info[], tool?: { messageID, callID } }
Question.Reply = { answers: string[][] }  // one answer-array per question
```

Events: `Asked`, `Replied`, `Rejected`.

---

### Location — `location.ts`

```ts
Location.Ref = { directory: AbsolutePath, workspaceID?: Workspace.ID }
Location.Info extends Location.Ref = {
  directory, workspaceID?,
  project: { id: Project.ID, directory: AbsolutePath }
}
Location.response<T>(data: T) → { location: Location.Info, data: T }
// Used for all location-scoped API responses
```

---

### Integration — `integration.ts`

```ts
Integration.AttemptStatus =
  | { status: "pending" }
  | { status: "complete" }
  | { status: "failed", message: string }
  | { status: "expired" }

Integration.Attempt = {
  attemptID: AttemptID
  url: string
  instructions: string
  mode: "auto" | "code"
  time: { created, expires }
}
```

Auth method types: `KeyMethod` (API key input), `EnvMethod` (reads from env var), `OAuthMethod` (device flow or browser redirect), `TextPrompt`/`SelectPrompt` (custom input forms).

---

### PTY — `pty.ts`

```ts
Pty.Info = { id, title, command, args, cwd, status: "running"|"exited", pid, exitCode? }
Pty.CreateInput = { command?, args?, cwd?, title?, env? }
Pty.UpdateInput = { title?, size?: { rows, cols } }
```

`PtyTicket.ConnectToken = { ticket: string, expires_in: PositiveInt }` — one-time token for WebSocket upgrade to PTY.

---

### Revert — `revert.ts`

```ts
Revert.FileDiff = { path, status: "added"|"modified"|"deleted", additions, deletions, patch }
Revert.State = { messageID, partID?, snapshot?, diff?, files?: FileDiff[] }
```

---

## Event System — `event.ts` + `event-manifest.ts`

### `Event.define` — Defining Events

```ts
const MyEvent = Event.define({
  type: "my-namespace.my-event",
  durable: { aggregate: "sessionID", version: 1 },  // optional
  schema: Schema.Struct({ sessionID: Session.ID, ... }),
})
// MyEvent.type        → "my-namespace.my-event"
// MyEvent.durable     → durable options
// MyEvent.data        → schema type
```

### `Event.inventory` / `Event.latest` / `Event.durable`

- `Event.inventory(...defs)` — frozen definition array
- `Event.latest(defs)` — deduplicates by type, keeps highest `durable.version`
- `Event.durable(defs)` — builds `ReadonlyMap<"type.version", Definition>`

### Event Manifest Layers

```
SessionV1 events         (legacy V1 session events)
     +
SessionEvent.Definitions (current V2 session events)
     = coreDefinitions

coreDefinitions
+ ModelsDev, Integration, Catalog events
= foundationDefinitions

foundationDefinitions
+ FileSystem, Reference, Permission, Plugin, ProjectDirectories
+ FileSystemWatcher, Pty, Question events
= ServerDefinitions   ← used by the server

ServerDefinitions
+ SessionV1 live events, InstallationEvent, LspEvent, PermissionV1
+ TuiEvent, McpEvent, LegacyEvent, Project, SessionStatusEvent
+ QuestionV1, SessionCompactionEvent, VcsEvent, WorkspaceEvent
+ WorktreeEvent, ServerEvent
= Definitions         ← full set (used by TUI)
```

### All Non-Session Event Types

| Event | Data |
|---|---|
| `"server.connected"` | `{}` |
| `"global.disposed"` | `{}` |
| `"catalog.updated"` | `{}` |
| `"models-dev.refreshed"` | `{}` |
| `"installation.updated"` | `{ version: string }` |
| `"installation.update-available"` | `{ version: string }` |
| `"lsp.updated"` | LSP server state |
| `"mcp.tools.changed"` | `{ server: string }` |
| `"mcp.browser.open.failed"` | `{ mcpName, url }` |
| `"tui.prompt.append"` | text |
| `"tui.command.execute"` | command name |
| `"tui.toast.show"` | toast data |
| `"tui.session.select"` | session ID |
| `"vcs.branch.updated"` | `{ branch?: string }` |
| `"workspace.ready"` | `{ name }` |
| `"workspace.failed"` | `{ message }` |
| `"workspace.status"` | `WorkspaceEvent.ConnectionStatus` |
| `"worktree.ready"` | `{ name, branch? }` |
| `"worktree.failed"` | `{ message }` |
| `"session.compacted"` | `{ sessionID }` |
| `"session.status"` | `{ sessionID, status: "idle"\|"busy"\|"retry" }` |
| `"session.idle"` | `{ sessionID }` (deprecated) |
| `"project.directories.updated"` | `{ projectID }` |
| `"file.watcher.updated"` | `{ file, event: "add"\|"change"\|"unlink" }` |
| `"file.edited"` | `{ file }` |
| `"todo.updated"` | `{ sessionID, todos: Todo[] }` |
| `"plugin.added"` | `{ id: Plugin.ID }` |
| `"permission.asked"` | V1 permission request |
| `"permission.replied"` | V1 permission reply |
| `"question.asked"` | V1 question |
| `"question.replied"` | V1 reply |
| `"question.rejected"` | V1 rejection |
| `"integration.updated"` | integration state |
| `"integration.connection.updated"` | connection state |
| `"reference.updated"` | reference list changed |
| `"command.executed"` | `{ name, sessionID, arguments, messageID }` |
| `"project.updated"` | project info |

---

## packages/protocol — API Contract

### HttpApi Composition — `api.ts`

`makeApi(options)` assembles the full API as an `Effect HttpApi`:

```ts
HttpApi.make("server")
  .add(HealthGroup)
  .add(LocationGroup.middleware(locationMiddleware))
  .add(AgentGroup.middleware(locationMiddleware))
  .add(makeSessionGroup(sessionLocationMiddleware))
  .add(MessageGroup.middleware(sessionLocationMiddleware))
  .add(ModelGroup.middleware(locationMiddleware))
  .add(ProviderGroup.middleware(locationMiddleware))
  .add(IntegrationGroup.middleware(locationMiddleware))
  .add(CredentialGroup.middleware(locationMiddleware))
  .add(makePermissionGroup(...))
  .add(FileSystemGroup.middleware(locationMiddleware))
  .add(CommandGroup.middleware(locationMiddleware))
  .add(SkillGroup.middleware(locationMiddleware))
  .add(eventGroup)
  .add(PtyGroup.middleware(locationMiddleware))
  .add(makeQuestionGroup(...))
  .add(ReferenceGroup.middleware(locationMiddleware))
  .add(ProjectCopyGroup.middleware(locationMiddleware))
  .middleware(Authorization)         // global Basic auth
  .middleware(SchemaErrorMiddleware)  // global schema validation error mapping
```

The `locationMiddleware` and `sessionLocationMiddleware` are injected by the server (`packages/server/`) so the protocol layer doesn't depend on server-side service types.

`makeDefaultApi(options)` — uses the static `EventGroup` (with `EventManifest.ServerDefinitions`).
`makeApi(options)` — uses a dynamic event group (used when the server needs to include custom event types).

### Middleware

**`Authorization`** — `HttpApiMiddleware.Service` that adds `UnauthorizedError` to all endpoint error unions. Concrete implementation in `packages/server/`.

**`SchemaErrorMiddleware`** — `HttpApiMiddleware.Service` that adds `InvalidRequestError` to all endpoint error unions. Concrete implementation catches Effect Schema parse errors.

### Endpoint Identifiers

Every endpoint has a unique string identifier used by the SDK for type-safe calls:

```
v2.health.get
v2.location.get
v2.agent.list
v2.session.list / .create / .active / .get
v2.session.switchAgent / .switchModel
v2.session.prompt / .compact / .wait / .interrupt
v2.session.revert.stage / .clear / .commit
v2.session.context / .history / .events / .message
v2.session.messages
v2.session.permission.create / .list / .get / .reply
v2.session.question.list / .reply / .reject
v2.model.list
v2.provider.list / .get
v2.integration.list / .get
v2.integration.connect.key / .connect.oauth
v2.integration.attempt.status / .complete / .cancel
v2.credential.update / .remove
v2.permission.request.list
v2.permission.saved.list / .remove
v2.question.request.list
v2.fs.read / .list / .find
v2.command.list
v2.skill.list
v2.pty.list / .create / .get / .update / .remove / .connectToken / .connect
v2.reference.list
v2.event.subscribe
v2.projectCopy.create / .remove / .refresh
```

### `Location.response<T>` Wrapper

Location-scoped endpoints return `{ location: Location.Info, data: T }`. The client uses the embedded `Location.Info` to route requests to the correct Location service instance.

### PTY WebSocket

`v2.pty.connect` — the PTY connection endpoint uses a WebSocket upgrade (not SSE). Protected by one-time ticket (`x-opencode-ticket` header or `ticket` query param). `hasPtyConnectTicketURL(url)` bypasses the Basic auth check for WebSocket upgrades.

---

## V1 Schema Compatibility — `schema/src/v1/`

V1 schemas are kept for reading legacy event data from SQLite. The V1 event system used `MessagePart` (fine-grained part updates) instead of V2's projection-based message table.

V1 part types: `TextPart`, `ReasoningPart`, `FilePart`, `ToolPart`, `StepStartPart`, `StepFinishPart`, `SnapshotPart`, `PatchPart`, `AgentPart`, `RetryPart`, `CompactionPart`, `SubtaskPart`.

V1 events: `session.created/updated/deleted`, `message.updated/removed`, `message.part.updated/removed/delta`, `session.diff`, `session.error`.

These events are still in the SQLite database for sessions created before the V2 migration and are handled by the backward-compatibility code in the session projector.

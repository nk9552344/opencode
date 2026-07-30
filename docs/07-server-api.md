# Chapter 7: HTTP Server and API

**Source:** `packages/server/` and `packages/protocol/`

The HTTP server exposes all opencode functionality as a typed REST + SSE API. It is built on Effect's `HttpApiBuilder` framework, which derives routing, validation, and OpenAPI generation from the protocol schema. The `packages/protocol/` package defines the API contract; `packages/server/` implements it.

---

## Architecture

```
packages/protocol/   ← API contract (HttpApi spec, endpoint types, error types)
         ↓ implements
packages/server/     ← Route handlers, auth, middleware, service wiring
```

The split means the SDK and TUI import only `packages/protocol/` to get the contract types, without pulling in server-side dependencies.

---

## Starting the Server

The server is started by the `serve` command handler (`src/commands/handlers/serve.ts`):

```ts
// Bind to a port (auto-increments from 4096 if no port specified)
const address = yield* listen(hostname, port, password)

// If --register: write server.json and start the watchdog
yield* daemon.register(address)
yield* Effect.never  // keep running
```

The server also supports embedded mode (in-process for testing/SDK use):

```ts
const handler = createEmbeddedRoutes()   // no auth
// or
const handler = createRoutes(password)   // with password auth
```

---

## Authentication — `src/auth.ts`

HTTP Basic authentication:

- **Username:** `OPENCODE_SERVER_USERNAME` env var (default: `"opencode"`)
- **Password:** `OPENCODE_SERVER_PASSWORD` env var, or the daemon-managed password file

Every request must include `Authorization: Basic base64(username:password)` (except health check).

The `ServerAuth.headers(credentials?)` utility generates the correct auth header for SDK clients.

---

## Middleware

### Authorization Middleware

Checks `Authorization: Basic` header on every request. Returns `401 UnauthorizedError` if invalid when a password is configured.

### Schema Error Middleware

Converts Effect Schema validation errors (malformed request bodies) into `400 InvalidRequestError` with a human-readable message.

### Session Location Middleware

For session-scoped routes: extracts `sessionID` from the path, looks up the session's directory in the database, and injects the correct `Location` context so the handler runs in the right Location service instance.

---

## API Groups and Endpoints

The full API is defined in `packages/protocol/src/api.ts` as an Effect `HttpApi`. Here are all endpoint groups:

### Health — `GET /v2/health`

```json
{ "healthy": true, "version": "1.18.9" }
```

### Session — `/api/session`

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/session` | List sessions (paginated) |
| `POST` | `/api/session` | Create session |
| `GET` | `/api/session/active` | Active sessions map |
| `GET` | `/api/session/:id` | Get session info |
| `POST` | `/api/session/:id/agent` | Switch agent |
| `POST` | `/api/session/:id/model` | Switch model |
| `POST` | `/api/session/:id/prompt` | Submit prompt |
| `POST` | `/api/session/:id/compact` | Manual compaction |
| `POST` | `/api/session/:id/wait` | Wait for session idle |
| `POST` | `/api/session/:id/interrupt` | Interrupt current turn |
| `POST` | `/api/session/:id/revert/stage` | Stage a revert |
| `POST` | `/api/session/:id/revert/clear` | Clear staged revert |
| `POST` | `/api/session/:id/revert/commit` | Commit revert |
| `GET` | `/api/session/:id/context` | Current context messages |
| `GET` | `/api/session/:id/history` | Paginated durable events |
| `GET` | `/api/session/:id/event` | SSE event stream |
| `GET` | `/api/session/:id/message/:msgId` | Single message |

#### Session List Query Parameters

| Param | Type | Meaning |
|---|---|---|
| `directory` | string | Filter by working directory |
| `project` | string | Filter by project ID |
| `workspace` | string | Filter by workspace ID |
| `subpath` | string | Filter by path prefix |
| `cursor` | string | Pagination cursor (opaque base64url JSON) |
| `order` | `asc\|desc` | Time order |
| `search` | string | Full-text search in title/messages |
| `limit` | number | Page size (max 100) |

#### Session Create Body

```ts
{
  id?: string        // optional: specify for idempotency
  agent?: string     // agent ID
  model?: {          // model reference
    id: string
    providerID: string
    variant?: string
  }
  location: {        // working directory or project
    directory: string
  }
}
```

#### Prompt Body

```ts
{
  prompt: {
    text: string
    files?: Array<{ url: string; name?: string }>  // attachments
    agents?: string[]  // agent IDs for subagent delegation
  }
  delivery?: "queue" | "steer"  // default: "queue"
  resume?: string               // resume from session ID
}
```

#### Session History Response

```ts
{
  data: SessionEvent.DurableEvent[]   // raw durable events
  hasMore: boolean
}
```

Paginate with `?after=<lastSeq>`.

### Message — `/api/session/:id/message`

`GET /api/session/:id/message` — paginated message list.

Query params: `cursor` (base64url), `limit` (1–200), `order` (`asc|desc`).

Returns: `{ data: SessionMessage.Message[], hasMore: boolean }`

### Event Stream — `GET /api/event`

Global SSE stream of all server events. Receives:
- `server.connected` immediately on connect
- All `OpenCodeEvent` types (session events, model changes, catalog updates, etc.)
- Heartbeat comments every 15 seconds

Connect to this to build external UIs or automation that reacts to opencode activity.

### Agent — `GET /api/agent`

Returns list of available agents for the request's Location.

Query: `?directory=<path>` (Location context).

### Model — `GET /api/model`

Returns list of available models (filtered by provider availability and `enabled` flag).

### Provider — `/api/provider`

- `GET /api/provider` — list all providers
- `GET /api/provider/:providerID` — single provider info

### Permission — `/api/permission`

- `GET /api/permission` — list pending permission requests
- `GET /api/permission/:id` — get specific request
- `POST /api/permission/:id/respond` — respond to permission request

#### Permission Response

```ts
{
  effect: "allow" | "deny" | "always"
  // "always" saves the rule to persistent storage
}
```

### Credential — `/api/credential`

- `GET /api/credential` — list stored credentials
- `POST /api/credential` — create/update credential
- `DELETE /api/credential/:id` — remove credential

### Integration — `/api/integration`

- `GET /api/integration` — list integrations with their connection status
- `POST /api/integration/:id/connect` — start OAuth flow
- `POST /api/integration/:id/disconnect` — remove credential

### File System — `/api/fs`

- `GET /api/fs` — list directory contents
- `GET /api/fs/read` — read file content

### PTY — `/api/pty`

- `POST /api/pty` — create PTY session
- `POST /api/pty/:id/resize` — resize terminal
- `POST /api/pty/:id/write` — write input to terminal
- `DELETE /api/pty/:id` — close PTY
- `GET /api/pty/:id/event` — SSE stream of terminal output

### Question — `/api/question`

- `GET /api/question` — list pending questions
- `POST /api/question/:id/respond` — answer a question

### Reference — `/api/reference`

- `GET /api/reference` — list named references

### Skill — `/api/skill`

- `GET /api/skill` — list available skills

### Command — `/api/command`

- `GET /api/command` — list registered slash commands

### Project Copy — `/api/project`

- `POST /api/project/copy` — copy project files

---

## Error Responses

All errors use consistent HTTP status codes:

| Error Type | Status |
|---|---|
| `InvalidRequestError` | 400 |
| `UnauthorizedError` | 401 |
| `ForbiddenError` | 403 |
| `SessionNotFoundError` | 404 |
| `MessageNotFoundError` | 404 |
| `PermissionNotFoundError` | 404 |
| `ProviderNotFoundError` | 404 |
| `PtyNotFoundError` | 404 |
| `ConflictError` | 409 |
| `ServiceUnavailableError` | 503 |
| `UnknownError` | 500 |

Error response shape:
```json
{
  "_tag": "SessionNotFoundError",
  "message": "Session abc123 not found",
  "sessionID": "abc123"
}
```

---

## Service Layer Wiring — `src/routes.ts`

`createRoutes(password?)` assembles the full application:

```
applicationServices = [
  Database, EventV2, httpClient, ToolOutputStore,
  SessionV2, PermissionSaved, PtyTicket, Credential,
  PtyEnvironment, LocationServiceMap
]

serviceLayer = SessionExecutionLocal (runs session turns in-process)

Assembly:
  HttpApiBuilder.layer(Api)
    → all handler groups
    → sessionLocationLayer
    → locationLayer
    → authorizationLayer
    → schemaErrorLayer
    → auth
    → serviceLayer
    → applicationServices
```

---

## OpenAPI Spec

The server generates an OpenAPI 3.1 spec at `GET /openapi.json`. This spec is used by:
- The `opencode api` CLI command (for operation ID resolution)
- The SDK code generation (the generated client in `packages/sdk/js/src/v2/gen/` was generated from this spec)

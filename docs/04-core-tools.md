# Chapter 4: Core — Tool System

**Source:** `packages/core/src/tool/`

The tool system connects the LLM's requests to run code with actual execution on the user's machine. It handles tool definition, input parsing, execution, permission gating, and output bounding.

---

## Tool Definition

### `Tool.make<Input, Output, Structured>` — `tool/tool.ts`

```ts
const MyTool = Tool.make({
  description: "Does something useful",

  // Effect Schema for the model's JSON input
  input: Schema.Struct({
    path: Schema.String,
    content: Schema.String,
  }),

  // Effect Schema for the full output (stored/returned)
  output: Schema.Struct({
    success: Schema.Boolean,
    linesWritten: Schema.Number,
  }),

  // Optional: a subset of output for the model to see (token-efficient)
  structured: Schema.Struct({
    success: Schema.Boolean,
  }),

  // The actual execution
  execute: (input, context) =>
    Effect.gen(function* () {
      // input is fully decoded from JSON
      // context provides session/tool metadata
      yield* context.ask({ action: "write", resource: input.path })  // permission
      // ...do work...
      return { success: true, linesWritten: 42 }
    }),

  // Optional: custom model-facing content (text/file parts)
  toModelOutput: (output) => [{ type: "text", text: `Wrote ${output.linesWritten} lines` }],
})
```

### `Tool.withPermission(tool, permission)`

Decorates a tool with a named permission string. The registry uses this to apply permission rules and to check if a tool is wholly disabled before sending it to the model.

### Tool Name Validation

Tool names must match `/^[A-Za-z][A-Za-z0-9_-]{0,63}$/`. This is enforced at registration time.

---

## Tool Registry — `tool/registry.ts`

`ToolRegistry.Service` (Location-scoped) is the runtime materialization layer.

### Registration

```ts
// Register tools scoped to an Effect Scope
ToolRegistry.register({ myTool: MyTool, otherTool: OtherTool })
```

Registration is scope-managed: closing the scope removes the tools. This enables plugin tools to appear and disappear cleanly.

### Materialization

```ts
const { definitions, settle } = yield* ToolRegistry.materialize(permissions?)
```

`materialize()` produces:
- **`definitions`** — `ToolDefinition[]` filtered by permission rules (tools with `{ action: "*", resource: "*", effect: "deny" }` are omitted entirely from the model's view)
- **`settle(call, context)`** — executes one tool call end-to-end

### Settlement

`settle(call, context)`:
1. Look up the tool registration by `call.name`
2. Decode `call.input` from JSON using the tool's input schema
3. Call `tool.execute(decodedInput, context)`
4. Call `ToolOutputStore.bound(output)` to truncate/store large output
5. Return `ToolOutput { structured, content: ToolContent[] }`

---

## Permission System — `permission.ts`

All tool execution goes through the permission gate.

### Permission Rules

Rules are matched by `action` (tool name) and `resource` (the argument to the tool, like a file path or bash command):

```jsonc
// In opencode.json
"permissions": {
  "bash": {
    "allow": ["git *", "ls", "cat *"],
    "deny": ["rm -rf *"]
  },
  "edit": {
    "allow": ["src/**"]
  }
}
```

Wildcards use `fnmatch`-style matching (`*` matches anything). The **last matching rule wins**.

### Permission Evaluation

`PermissionV2.assert({ sessionID, action, resource })`:

1. Evaluate agent-level rules (from agent config)
2. Evaluate saved rules (from `PermissionSaved` SQLite table)
3. Result:
   - **`"allow"`** → proceed immediately
   - **`"deny"`** → fail with `BlockedError`
   - **`"ask"`** → pause the tool fiber, publish `Permission.Event.Asked`, wait for user response

### User Response Flow

When a tool triggers `"ask"`:
1. The tool fiber blocks on a `Deferred`
2. The server publishes a permission event — TUI shows a permission dialog
3. User clicks Allow / Deny / Always Allow
4. `PermissionV2.reply(requestID, effect)` resolves the `Deferred`
5. If `"always"`, the rule is saved to `PermissionSaved` (so future same-pattern requests auto-approve)

### Declined Error

If the user declines all pending requests for a session, the runner detects `DeclinedError` (a defect) and halts the current turn. This matches the user's expectation that declining a permission stops the AI.

---

## Built-In Tools

### BashTool — `tool/bash.ts`

Runs shell commands using the configured shell (default `/bin/sh`, or `COMSPEC` on Windows).

**Key behaviors:**
- Default timeout: 2 minutes. Maximum: 10 minutes (configurable per call).
- Output capped at 1 MB (`MAX_CAPTURE_BYTES`)
- Exit code non-zero → `ToolFailure` with stderr
- `external_directory` permission required for paths outside the Location
- Commands are scanned for absolute paths that reference external directories (advisory detection for safety)
- Permission: `{ action: "bash", resource: <full command string> }`

### EditTool — `tool/edit.ts`

Exact-string replacement in files.

```
Input:
  filePath: string         // absolute or relative
  oldString: string        // exact string to replace (must be unique in file)
  newString: string        // replacement
  replaceAll?: boolean     // replace all occurrences (default: false)
```

**Key behaviors:**
- Validates `oldString !== newString` and `oldString !== ""`
- CRLF-aware: normalizes line endings for matching, restores original line endings in output
- Uses CAS (Compare-And-Swap) write: `FileMutation.writeIfUnchanged` — rejects if the file was modified since it was last read, preventing data loss from concurrent edits
- Returns a diff with additions/deletions count
- Permission: `{ action: "edit", resource: <filePath> }`

### WriteTool — `tool/write.ts`

Writes full file content (creates or overwrites).

- Preserves BOM if the file already has one (`FileMutation.writeTextPreservingBom`)
- Permission: `{ action: "edit", resource: <filePath> }`

### ReadTool — `tool/read.ts`

Reads files or lists directories.

- Supports `offset` + `limit` for paginated reading of large files
- Image files (jpeg, png, gif, webp) are normalized via `Image.Service`: resized if oversized, returned as base64 data URI
- Directory listing returns file names with types
- Permission: `{ action: "read", resource: <filePath> }`

### GlobTool — `tool/glob.ts`

Glob pattern file search within the Location directory.

- Matches against the configured glob pattern
- Returns matching file paths
- Permission: `{ action: "glob", resource: <pattern> }`

### GrepTool — `tool/grep.ts`

Regex search in files using ripgrep.

- Uses the ripgrep binary (auto-downloaded if not present)
- Returns matching lines with file paths and line numbers
- Permission: `{ action: "grep", resource: <pattern> }`

### SkillTool — `tool/skill.ts`

Loads a named skill's markdown content and up to 10 associated files.

- Looks up skill by name from `SkillV2.Service`
- Returns the skill's SKILL.md content and any files in the skill's directory
- Permission: `{ action: "skill", resource: <skillName> }`

### TodoWriteTool — `tool/todowrite.ts`

Updates the session's todo list (shown in the TUI sidebar).

- Stores todo items in SQLite via `TodoTable`
- Each todo has: `id, content, status (pending|in_progress|completed), priority`
- Permission: `{ action: "todowrite", resource: "todos" }`

### WebFetchTool — `tool/webfetch.ts`

Fetches HTTP URLs and extracts readable content.

- Converts HTML to markdown for readability
- Permission: `{ action: "webfetch", resource: <url> }`

### WebSearchTool — `tool/websearch.ts`

Web search via configured search provider.

- Permission: `{ action: "websearch", resource: <query> }`

### ApplyPatchTool — `tool/apply-patch.ts`

Applies unified diff patches to files.

- Parses unified diff format
- Permission: `{ action: "edit", resource: <filePath> }`

### QuestionTool — `tool/question.ts`

Asks the user a structured question from within the AI turn.

- Publishes `Question.Event.Asked`, blocks until `QuestionV2.reply` is called
- Used by agents that need user input mid-execution
- Permission: `{ action: "question", resource: <questionType> }`

---

## Tool Output Store — `tool-output-store.ts`

Prevents tool output from overwhelming the model's context window.

### Limits (configurable in `tool_output` config)

| Limit | Default |
|---|---|
| `max_lines` | 2000 lines |
| `max_bytes` | 50 KB |

### `bound(input)`

If output fits within both limits: return as-is.

If output exceeds either limit:
1. Write full content to `~/.local/share/opencode/tool-output/tool_<ascending>/`
2. Return a preview: `ceil(max_lines/2)` head lines + `floor(max_lines/2)` tail lines
3. The preview includes an ellipsis with the file path: `... (full output at /path/to/file)`

The model sees the preview; the TUI can show a link to the full file.

### Cleanup

Files older than 7 days are deleted on a 1-hour schedule.

---

## Application Tools — `tool/application-tools.ts`

`ApplicationTools.Service` (global) is the registry of tools available to all Locations. Built-in tools are registered here at startup:

- `BashTool`
- `EditTool`
- `WriteTool`
- `ReadTool`
- `GlobTool`
- `GrepTool`
- `SkillTool`
- `TodoWriteTool`
- `WebFetchTool`
- `WebSearchTool`
- `ApplyPatchTool`
- `QuestionTool`

Plugins can also register tools here to make them available globally:

```ts
// In a plugin
host.plugin.add("my-plugin", Effect.gen(function* () {
  // Register a custom tool
  const tools = yield* ApplicationTools.Service
  yield* tools.register({ myCustomTool: MyCustomTool })
}))
```

---

## File Mutation Safety — `file-mutation.ts`

All file write operations go through `FileMutation.Service` which provides two safe primitives:

### `writeIfUnchanged({ target, expected, content })`

Writes `content` to `target` only if the current bytes match `expected`. Throws `StaleContentError` if the file was modified since it was read. This is the **Compare-And-Swap (CAS)** pattern used by `EditTool` to prevent data loss when files change between read and write.

### `writeTextPreservingBom({ target, content })`

Writes text content, preserving the file's existing BOM (Byte Order Mark) if present. Used by `WriteTool` for correctness with Windows-style UTF-8 files.

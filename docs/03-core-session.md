# Chapter 3: Core — Session System

**Source:** `packages/core/src/session/`

The session system is the heart of opencode. It is **event-sourced**: every change to a session (prompt submitted, AI text streamed, tool called, agent switched) is first recorded as a durable event in SQLite, then projected into readable table rows. This chapter covers the entire lifecycle from prompt submission to LLM response.

---

## Data Model

### Session (`session/sql.ts`)

A session row in `SessionTable` contains:

| Column | Meaning |
|---|---|
| `id` | ULID, primary key |
| `project_id` | Which git project this belongs to |
| `workspace_id` | Optional workspace |
| `directory` | Working directory when session was created |
| `title` | Auto-generated title |
| `agent` | Current agent ID |
| `model` | JSON: `{ id, providerID, variant? }` |
| `cost` | Running total cost (USD) |
| `tokens_input/output/reasoning` | Running token totals |
| `tokens_cache_read/write` | Cache token counts |
| `revert` | JSON: staged revert state |
| `time_created/updated` | Timestamps |
| `time_archived` | Set when session is archived |

### Session Messages (`session/sql.ts`)

`SessionMessageTable` is the **projection** of session events into readable message rows:

| Column | Meaning |
|---|---|
| `id` | Message ID (same as event ID for the primary event) |
| `session_id` | Parent session |
| `type` | `user`, `assistant`, `system`, `synthetic`, `shell`, `compaction`, `agent-switched`, `model-switched` |
| `seq` | Event sequence number (monotonic per session) |
| `data` | Full JSON of the `SessionMessage.Message` union type |

### Event Tables

`EventTable` stores all durable events:
- `id` — event ID
- `aggregate_id` — session ID (the aggregate)
- `seq` — sequence number within this aggregate
- `type` — event type string (e.g. `session.v1.created`, `session.text.delta`)
- `data` — JSON payload

`EventSequenceTable` tracks the current sequence number per aggregate.

---

## Event Sourcing Architecture

### EventV2 Service — `event.ts`

The central event bus and durable log. Key operations:

**`publish(definition, data, options?)`** — For durable events, runs inside a SQLite transaction:
1. Read current sequence from `EventSequenceTable`
2. Write to `EventTable` (seq + 1)
3. Run all registered **projectors** atomically in the same transaction
4. Update sequence in `EventSequenceTable`
5. Notify in-memory PubSub subscribers

**`durable(aggregateID, after?)`** — Returns a `Stream` that:
1. Replays historical events from `EventTable` (from `after` sequence)
2. Tails live events via a PubSub subscription

**`project(definition, projector)`** — Registers a function that runs inside the publish transaction. Projectors must be registered at layer startup (before any events are published).

This transactional projection means **the message table is always consistent with the event log** — if a projector fails, the event is not recorded either.

### SessionProjector — `session/projector.ts`

Registers projectors for every session event type. Selected examples:

| Event | Projection action |
|---|---|
| `SessionV1.Event.Created` | INSERT into `SessionTable` |
| `SessionV1.Event.Updated` | UPDATE `SessionTable` fields |
| `SessionEvent.AgentSwitched` | UPDATE `SessionTable.agent` + run `SessionMessageUpdater` |
| `SessionEvent.Prompted` | `SessionInput.projectPrompted` + `SessionMessageUpdater` (creates User message row) |
| `SessionEvent.PromptAdmitted` | INSERT into `SessionInputTable` |
| `SessionEvent.Text.Delta` | `SessionMessageUpdater` (appends text to assistant message JSON) |
| `SessionEvent.Tool.Called` | `SessionMessageUpdater` (updates tool state to running) |
| `SessionEvent.Step.Ended` | `SessionMessageUpdater` + update session cost/tokens |
| `SessionEvent.Compaction.Ended` | `SessionMessageUpdater` (inserts compaction message row) |
| `SessionEvent.RevertEvent.Committed` | DELETE `SessionMessageTable` rows after boundary, clean inputs |

### SessionMessageUpdater — `session/message-updater.ts`

A pure stateful projection that applies a `SessionEvent.Event` to a message state. Uses immer `produce` to build the next `SessionMessage.Message` from the current one. The DB-backed adapter reads the current row, applies the update, writes it back.

---

## Session Lifecycle

### Creating a Session

```ts
SessionV2.create({
  id?,              // optional: specify ID for idempotency
  agent?,           // agent ID
  model?,           // model ref
  location: { directory: "/my/project" }
})
```

Internally:
1. Resolve project (`ProjectV2.resolve(directory)`) → git root + project ID
2. Publish `SessionV1.Event.Created` (durable) → projector inserts `SessionTable` row
3. Duplicate creation is handled: `SessionAlreadyProjected` defect is caught and converted to a read of the existing session

### Submitting a Prompt

```ts
SessionV2.prompt({
  sessionID,
  prompt: { text: "Please explain this code" },
  delivery: "queue",  // or "steer" to inject into current turn
})
```

Internally:
1. **Admit** — `SessionInput.admit(db, events, input)` publishes `SessionEvent.PromptAdmitted` (durable). Returns an `Admitted` record. Idempotent: if the same prompt was already admitted (same ID or same content), returns the existing record.
2. **Wake** — `SessionExecution.wake(sessionID)` signals the `SessionRunCoordinator` to start or continue processing.

**Delivery types:**
- `"queue"` — processed as an independent turn after the current one finishes
- `"steer"` — injected into the current turn (model sees it mid-response)

### Session Input Inbox — `session/input.ts`

`SessionInputTable` is the "inbox" for pending prompts:

| State | Meaning |
|---|---|
| `admitted_seq` set, `promoted_seq` null | Pending (not yet promoted into the event log) |
| Both set | Promoted (the `Prompted` event has been published) |

`promoteSteers(db, events, sessionID, cutoff)` — promotes all steer inputs whose `admitted_seq ≤ cutoff` by publishing `SessionEvent.Prompted` events (which the projector turns into User message rows).

`promoteNextQueued(db, events, sessionID)` — promotes exactly one queued input.

---

## Session Runner

### Coordination — `session/run-coordinator.ts`

`SessionRunCoordinator.make({ drain })` creates a keyed coordinator that:
- **`run(key)`** — if no active entry, starts `drain(key, force=true)`. If there is an active entry, joins it (returns when it completes).
- **`wake(key)`** — if active, sets `pendingWake = true` (so a follow-up drain runs after the current one). If idle, starts `drain(key, force=false)`.
- **`interrupt(key)`** — fiber-interrupts the active drain, clears pending wakes.

This ensures at most one runner fiber per session, with coalesced wakes.

### The LLM Execution Loop — `session/runner/llm.ts`

The core loop (`run({ sessionID, force })`):

```
OUTER LOOP: while there are pending inputs
  PROMOTION: promote steers + next queued input
  INNER LOOP: while continuation needed (tool calls were made)
    1. Verify session location matches current Location
    2. Select agent
    3. Initialize/reconcile system context (AGENTS.md + builtins)
    4. Resolve model (via SessionRunnerModel)
    5. Load conversation history (projected messages)
    6. Determine if this is the final allowed step
    7. Materialize tool definitions (filtered by permissions)
    8. Build LLMRequest (model, system, messages, tools)
    9. Check compaction (may abort + compact if token budget exceeded)
   10. Capture "before" snapshot
   11. Stream provider response, publishing session events for each LLM event
   12. For each tool-call event: start settle fiber in parallel
   13. Await all settle fibers
   14. If all tools returned: loop (needsContinuation = true)
   15. If no tools or max steps: exit inner loop
  CHECK: any new inputs admitted during this turn? → outer loop again
```

### Model Resolution — `session/runner/model.ts`

`SessionRunnerModel.resolve(session)`:

1. Look up model in `Catalog` by `session.model.providerID + session.model.id`
2. Check that the provider has an active connection (API key or OAuth token)
3. Map the model's `api` type to an LLM protocol:
   - `@ai-sdk/anthropic` → Anthropic Messages protocol
   - `@ai-sdk/openai` → OpenAI Responses protocol
   - `@ai-sdk/openai-compatible` → OpenAI Compatible Chat protocol
4. Apply variant overrides (extra request headers/body patches)
5. Return an `@opencode-ai/llm` `Model` object

### Message → LLM Translation — `session/runner/to-llm-message.ts`

Converts projected `SessionMessage.Message[]` to `@opencode-ai/llm` `Message[]`:

- `user` messages → `LLM.Message.user([{type:"text",text}])`
- `assistant` messages → `LLM.Message.assistant([...content parts])` with tool call/result pairs
- `compaction` messages → wrapped as `<conversation-checkpoint>` user message
- Reasoning parts stripped (converted to text) when the model differs from original
- Provider-executed tool results (web_search, etc.) carried through as native content

### LLM Event Publishing — `session/runner/publish-llm-event.ts`

`createLLMEventPublisher` translates `LLMEvent` stream events into session durable events:

| LLM Event | Session Event Published |
|---|---|
| First text/reasoning/tool event | `SessionEvent.Step.Started` |
| `text-start/delta/end` | `SessionEvent.Text.Started/Delta/Ended` |
| `reasoning-start/delta/end` | `SessionEvent.Reasoning.Started/Delta/Ended` |
| `tool-input-start/delta/end` | `SessionEvent.Tool.Input.Started/Delta/Ended` |
| `tool-call` | `SessionEvent.Tool.Called` |
| (after tool runs) | `SessionEvent.Tool.Success/Failed` |
| `step-finish` | `SessionEvent.Step.Ended` (with usage + snapshot) |
| `provider-error` | `SessionEvent.Step.Failed` |

All publications go through a `Semaphore(1)` to prevent races with concurrent tool fibers.

---

## Context Window Management

### System Context — `system-context/`

The system prompt is built from composable `SystemContext.Source<A>` objects. Each source has:
- `key` — unique namespaced string
- `load()` — Effect that returns the current value or `Unavailable`
- `baseline(current)` — formats the initial prompt string
- `update(previous, current)` — formats the delta when value changes

Built-in sources:
- `"core/environment"` — working directory, workspace root, git status, platform
- `"core/date"` — today's date
- `"core/skill-guidance"` — available skills list
- `"core/reference-guidance"` — named references with descriptions
- `"core/instructions"` — content of all `AGENTS.md` files

**Initialization:** On the first turn for a session, `SystemContextEpoch.initialize` calls `SystemContext.initialize` which loads all sources, generates the full baseline string, and stores a snapshot of each source's value in `SessionContextEpochTable`.

**Reconciliation:** On subsequent turns, `SystemContext.reconcile` compares current values against the stored snapshot. If anything changed, it generates delta text and publishes a `SessionEvent.ContextUpdated` event (which injects a system message into the conversation). If the schema changed (source added/removed), it forces a full rebuild.

### Compaction — `session/compaction.ts`

When the context exceeds the model's limit, compaction summarizes old conversation:

1. Estimate tokens for the pending LLM request
2. If `tokens > context - max(output, buffer)`: trigger compaction
3. Build a summarization prompt from the full conversation history
4. Stream an LLM completion (uses the same model) to generate a summary
5. Find a split point: most recent messages that fit within `keep.tokens`
6. Publish `Compaction.Started` + `Compaction.Ended` events
7. The projector replaces old messages with a single `Compaction` message row

The compaction message carries both the summary and the recent messages, so the model always has the last N tokens of conversation regardless of how long it ran.

---

## Revert System — `session/revert.ts`

The revert system lets users undo changes made by the AI.

### Stage a Revert

```ts
SessionRevert.stage({ sessionID, messageID })
```

1. Finds the assistant message at `messageID`
2. Collects all files changed in messages after the boundary (from their `snapshot.start` values)
3. Restores those files to their pre-change state using `Snapshot.restore`
4. Publishes `SessionEvent.RevertEvent.Staged`

### Snapshot Service — `snapshot.ts`

Uses a shadow git repository at `~/.local/share/opencode/snapshot/<projectID>/<hash>/`:

- **`capture()`** — creates a git tree object from the current working directory (respecting `.gitignore`, skipping large untracked files). Returns a content-addressed `Snapshot.ID`.
- **`files(from, to)`** — lists changed paths between two tree IDs
- **`restore(input)`** — checks out selected files from their tree IDs into the working directory
- **`diff(input)`** — unified diff between two trees

The runner captures a snapshot before (`Step.Started`) and after (`Step.Ended`) each turn, storing the IDs in the step events. The revert system uses these stored IDs to restore files to their pre-change state.

---

## Reading Session Data

### SessionStore — `session/store.ts`

- `get(sessionID)` → reads `SessionTable` row → `Session.Info`
- `context(sessionID)` → reads projected messages after last compaction → `SessionMessage.Message[]`
- `message(messageID)` → reads one `SessionMessageTable` row

### SessionHistory — `session/history.ts`

`load(db, sessionID)` — determines the display window:
1. Find the latest compaction `seq`
2. Find the system context epoch baseline `seq`
3. Query `SessionMessageTable` for rows >= max(compaction_seq, baseline_seq)
4. Exclude system messages before the baseline (they're part of the context epoch)

This means the TUI always shows: the compaction summary + all messages since the last compaction.

---

## SSE Event Streaming

Clients subscribe to session events via:

```
GET /api/session/:sessionID/event
GET /api/event  (all sessions)
```

Both are SSE streams. The server runs:
```ts
EventV2.durable(sessionID, afterSeq)
  → Stream<SessionEvent.DurableEvent>
  → SSE: "data: {JSON}\n\n"
```

New clients immediately receive historical events from `after` sequence, then tail live events. This enables TUI reconnection without missing events.

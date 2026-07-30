# Chapter 6: LLM Layer

**Source:** `packages/llm/`

The LLM layer is a protocol-neutral, Effect-based abstraction over every AI provider's HTTP API. It completely separates the semantic request/response contract from provider-specific wire formats, authentication, and transport.

---

## Core Concept: Four Orthogonal Axes

Every LLM request flows through four independently composable pieces:

```
Route = Protocol + Endpoint + Auth + Framing

Protocol  →  what the request body looks like and how to parse the response stream
Endpoint  →  where to send the request (URL construction)
Auth      →  how to authenticate (bearer, header, SigV4, OAuth, etc.)
Framing   →  how to decode the byte stream (SSE, AWS binary event-stream)
```

This means, for example, the `openai-chat` Protocol can be reused unchanged by OpenAI, Azure, Together, Groq, DeepSeek, Cerebras, Fireworks, and any other OpenAI-compatible provider — they just need a different Endpoint and Auth.

---

## Domain Schema — `src/schema/`

### LLMRequest — the canonical input

```ts
class LLMRequest {
  id?: string
  model: Model            // { id, provider, route, defaults?, compatibility? }
  system: SystemPart[]    // system prompt parts
  messages: Message[]     // conversation history
  tools: ToolDefinition[] // available tools
  toolChoice?: ToolChoice
  generation?: GenerationOptions   // temperature, maxTokens, etc.
  providerOptions?: ProviderOptions  // { anthropic: { thinking: {...} } }
  http?: HttpOptions               // per-request header/body/query overrides
  responseFormat?: ResponseFormat
  cache?: CachePolicy
  metadata?: unknown
}
```

### LLMResponse — the canonical output

```ts
class LLMResponse {
  message: Message          // the assistant message
  events: LLMEvent[]        // the raw event stream
  usage?: Usage             // token counts
  finishReason: FinishReason

  // Computed accessors
  get text(): string                    // concatenated text content
  get reasoning(): string              // concatenated reasoning content
  get toolCalls(): ToolCallPart[]      // all tool calls
}
```

`LLMResponse.reduce(state, event)` — folds a single `LLMEvent` into the response state.
`LLMResponse.complete(state)` — finalizes the response.
`LLMResponse.fromEvents(events)` — builds a complete response by folding all events.

### LLMEvent — the streaming event union

16 event types forming a state machine:

```
step-start                          ← turn begins
  text-start → text-delta* → text-end    ← text output
  reasoning-start → reasoning-delta* → reasoning-end  ← thinking output
  tool-input-start → tool-input-delta* → tool-input-end → tool-call  ← tool
  tool-result / tool-error          ← (after tool executes)
  provider-error                    ← in-stream error
step-finish                         ← turn ends (carries usage + finish reason)
finish                              ← stream terminates
```

### Message Content Parts

```ts
type ContentPart =
  | { type: "text"; text: string; cache?: CacheHint; metadata?: unknown }
  | { type: "media"; mediaType: string; data: string }  // base64 image
  | { type: "reasoning"; text: string; providerMetadata?: ProviderMetadata }
  | { type: "tool-call"; id: string; name: string; input: unknown }
  | { type: "tool-result"; toolCallId: string; result: ToolResultValue }
```

### LLMError

A typed error wrapping an `LLMErrorReason` discriminated union:

| Reason | Retryable | Meaning |
|---|---|---|
| `InvalidRequest` | false | Bad request; sub-classification `"context-overflow"` |
| `Authentication` | false | Missing/invalid/expired credentials |
| `RateLimit` | true | 429; carries `retryAfterMs` |
| `QuotaExceeded` | false | Hard quota limit |
| `ContentPolicy` | false | Content filter refusal |
| `ProviderInternal` | true | 5xx server errors |
| `Transport` | false | Network-level failures |
| `InvalidProviderOutput` | false | Unparseable provider response |
| `UnknownProvider` | false | Catch-all for unexpected HTTP errors |

---

## Protocol Layer — `src/route/protocol.ts`

A `Protocol<Body, Frame, Event, State>` defines the wire format:

```ts
interface Protocol<Body, Frame, Event, State> {
  id: ProtocolID
  body: {
    schema: Codec<Body, unknown>           // validates the request body
    from(request: LLMRequest): Effect<Body>  // builds the body from a canonical request
  }
  stream: {
    event: Codec<Event, Frame>             // decodes one frame → provider event
    initial(request): State               // initial streaming state
    step(state, event): Effect<[State, LLMEvent[]]>  // state machine step
    terminal?(event): boolean              // is this the last event?
    onHalt?(state): LLMEvent[]            // events to emit when stream ends naturally
  }
}
```

### Available Protocols

| Protocol ID | Used By |
|---|---|
| `anthropic-messages` | Anthropic |
| `openai-chat` | OpenAI, Azure, Together, Groq, DeepSeek, Cerebras, etc. |
| `openai-responses` | OpenAI (Responses API), xAI |
| `gemini` | Google |
| `bedrock-converse` | AWS Bedrock |
| `openai-compatible-chat` | Generic OpenAI-compatible providers |

---

## Provider Protocols in Detail

### Anthropic Messages Protocol

- **Base URL:** `https://api.anthropic.com/v1`
- **Path:** `/messages`
- **Auth:** `x-api-key` header; env `ANTHROPIC_API_KEY`
- **Body:** `model, system, messages, tools, tool_choice, stream:true, max_tokens, thinking`
- **Caching:** ephemeral cache hints with 4 breakpoints (tools → system → messages)
- **Extended Thinking:** `thinking: { type: "enabled", budget_tokens: N }` enables `reasoning` parts
- **Server Tools:** web_search, code_execution, web_fetch results round-tripped as opaque JSON
- **Usage:** native breakdown — `inputTokens = nonCached + cacheRead`, `cacheWriteInputTokens` separate

### OpenAI Chat Protocol

- **Base URL:** `https://api.openai.com/v1`
- **Path:** `/chat/completions`
- **Body:** `model, messages, tools, tool_choice, stream:true, stream_options:{include_usage:true}, reasoning_effort`
- **Reasoning:** `reasoning_content` field for o-series models
- **Tool Images:** base64 images in tool results buffered and appended as a user message
- **Usage:** from the final `[DONE]` chunk via `stream_options.include_usage`

### OpenAI Responses Protocol

- **Path:** `/responses`
- **Body:** Responses API format with `input` array, `instructions`, `reasoning`, `text.verbosity`
- **Hosted Tools:** `web_search_call`, `file_search_call`, `code_interpreter_call`, `computer_use_call` emitted as tool-call + tool-result pairs with `providerExecuted: true`
- **Reasoning:** complex streaming via `reasoning_summary_part` events; `encrypted_content` round-tripped for stateless requests
- **WebSocket variant:** for Realtime API

### Gemini Protocol

- **Base URL:** `https://generativelanguage.googleapis.com/v1beta`
- **Path:** dynamic: `/models/${model.id}:streamGenerateContent?alt=sse`
- **Auth:** `x-goog-api-key` header; env `GOOGLE_GENERATIVE_AI_API_KEY`
- **Tool Schema:** sanitized and projected to Gemini's JSON Schema subset (fixes integer enums, missing `items`, orphan `required`)
- **Thinking:** `thought: true` parts preserved via `thoughtSignature` for round-tripping
- **Tool Call IDs:** synthetic (`tool_0`, `tool_1`, ...)

### Bedrock Converse Protocol

- **Auth:** SigV4 (AWS credential providers)
- **Framing:** AWS binary event-stream (not SSE)
- **Path:** `/model/${modelId}/converse-stream`
- **Supports:** cache points, tool use, streaming

---

## Route Layer — `src/route/client.ts`

`Route.make(input)` assembles a Protocol + Endpoint + Auth + Framing into an executable route:

```ts
const myRoute = Route.make({
  id: "my-provider",
  protocol: OpenAIChat.protocol,
  endpoint: Endpoint.path("https://api.example.com/v1/chat/completions"),
  auth: Auth.bearer(Credential.orElse(Auth.config("MY_PROVIDER_API_KEY"))),
  framing: Framing.sse,
})
```

### Route.compile(request)

The critical boundary: resolves all defaults (route → model → request), applies cache policy, builds the provider body, validates it, prepares the transport. No network I/O.

### LLMClient Service

```ts
// All three use the same Route under the hood
LLMClient.prepare(request)   // Effect<PreparedRequest, LLMError>
LLMClient.stream(request)    // Stream<LLMEvent, LLMError>
LLMClient.generate(request)  // Effect<LLMResponse, LLMError>
```

`generate()` is `stream().runFold(LLMResponse.empty, LLMResponse.reduce)`.

---

## Authentication — `src/route/auth.ts`

Composable auth pipeline:

```ts
// Common patterns
Auth.bearer(Credential.orElse(Auth.config("OPENAI_API_KEY")))
Auth.header("x-api-key", Auth.config("ANTHROPIC_API_KEY"))
Auth.bearerHeader("github-token", Auth.value(githubToken))
Auth.none                          // no auth
Auth.passthrough                   // forward existing headers

// Composition
auth1.andThen(auth2)               // apply both
auth1.orElse(auth2)               // try auth1, fall back to auth2
Auth.remove("Authorization")       // remove a header (used by Azure)
```

### Credential Sources

```ts
Credential.from(effect)            // arbitrary Effect<RedactedString>
Credential.orElse(fallback)        // try credential, fall back if unavailable
credential.bearer()                // wrap as Bearer header
credential.header("x-api-key")    // wrap as named header
```

---

## HTTP Transport & Retry — `src/route/executor.ts`

`RequestExecutor.Service` executes HTTP requests with:

- **Automatic retry** (up to 2 attempts) for retryable errors (429, 503, 504, 529)
- **Backoff:** exponential + `Retry-After` header respect
- **Error classification:** HTTP status → `LLMErrorReason` mapping
- **Rate limit parsing:** OpenAI `x-ratelimit-*` and Anthropic `anthropic-ratelimit-*` formats
- **Redaction:** sensitive values (API keys, tokens) are masked in logs and error messages

---

## High-Level API — `src/llm.ts`

The `LLM` namespace provides convenience wrappers:

```ts
// Simple generation
const response = yield* LLM.generate({
  model: AnthropicProvider.model("claude-sonnet-4-5"),
  system: [{ type: "text", text: "You are helpful." }],
  messages: [Message.user("What is 2+2?")],
  tools: [],
})

// Streaming
yield* LLM.stream(request).pipe(
  Stream.runForEach(event => handleEvent(event))
)

// Structured output (via forced tool call - works on ALL providers)
const result = yield* LLM.generateObject({
  schema: Schema.Struct({ name: Schema.String, age: Schema.Number }),
  model: ..., system: ..., messages: [...],
})
// result.value is { name: string, age: number }
```

`generateObject` works by injecting a synthetic `generate_object` tool definition and forcing `toolChoice: { type: "tool", name: "generate_object" }`. This gives uniform behavior across all providers without relying on native JSON mode.

---

## Cache Policy — `src/cache-policy.ts`

`applyCachePolicy(request)` marks message parts with cache hints before the body builder runs. Effective only for Anthropic and Bedrock (which support inline cache points).

### Default Policy: `"auto"`

Applied when `request.cache` is undefined. Marks:
1. Last tool definition (if any)
2. Last system part
3. Last user message

This is net-positive: cache write cost is 1.25× the input token price, but cache read is 0.1×. Any single reuse within 5 minutes breaks even.

### Explicit Policies

```jsonc
// Turn caching off entirely
"cache": "none"

// Cache specific breakpoints
"cache": {
  "tools": true,          // cache last tool definition
  "system": true,         // cache last system part
  "messages": "latest-user-message"  // or "latest-assistant" or { "tail": 3 }
}
```

---

## Provider Registry — `src/providers/`

All providers expose a consistent interface:

```ts
const provider = Anthropic.configure({
  apiKey: "sk-ant-...",
  baseURL: "https://proxy.example.com",  // optional override
})

const model = provider.model("claude-opus-4-5")
// model: Model { id, provider: "anthropic", route: AnthropicMessagesRoute, ... }
```

### Available Providers

| Provider | ID | Protocol | Notes |
|---|---|---|---|
| Anthropic | `anthropic` | `anthropic-messages` | Full thinking + cache support |
| OpenAI | `openai` | `openai-responses` (default), `openai-chat` | Selectable via `responses/chat` accessors |
| Google | `google` | `gemini` | |
| Amazon Bedrock | `amazon-bedrock` | `bedrock-converse` | SigV4 or Bearer auth |
| Azure | `azure` | `openai-responses` (default), `openai-chat` | Requires `resourceName` or `baseURL` |
| xAI | `xai` | `openai-responses` (default) | env `XAI_API_KEY` |
| OpenRouter | `openrouter` | `openrouter-chat` | Extends openai-chat with usage/reasoning |
| GitHub Copilot | `github-copilot` | `openai-responses` or `chat` | Auto-selects by model ID |
| Cloudflare AI Gateway | `cloudflare` | `openai-compatible-chat` | `accountId` + `gatewayId` |
| Cloudflare Workers AI | `cloudflare` | `openai-compatible-chat` | env `CLOUDFLARE_WORKERS_AI_TOKEN` |
| Generic OpenAI-compatible | `openai-compatible` | `openai-compatible-chat` | `baseURL` required |
| Groq | `groq` | `openai-compatible-chat` | Profile of `openai-compatible` |
| Together AI | `togetherai` | `openai-compatible-chat` | Profile |
| DeepSeek | `deepseek` | `openai-compatible-chat` | Profile |
| Cerebras | `cerebras` | `openai-compatible-chat` | Profile |
| Fireworks | `fireworks` | `openai-compatible-chat` | Profile |
| DeepInfra | `deepinfra` | `openai-compatible-chat` | Profile |
| Baseten | `baseten` | `openai-compatible-chat` | Profile |

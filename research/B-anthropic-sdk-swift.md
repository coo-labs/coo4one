# Lane B: Anthropic SDK + Agent-Loop Patterns in Native Swift

*Research date: 2026-05-11. Investigator: research-investigator agent, sub-commissioned by COO.*

---

## 1. Anthropic SDK for Swift — Does an official one exist?

**No official Swift SDK exists as of May 2026.** Anthropic's official client SDK portfolio covers Python, TypeScript, Java, Go, Ruby, C#, and PHP — Swift is absent from the list. Source: [platform.claude.com/docs/en/api/client-sdks](https://platform.claude.com/docs/en/api/client-sdks).

Three community options exist:

| SDK | Stars | Last release | Notes |
|-----|-------|-------------|-------|
| [SwiftClaude](https://github.com/GeorgeLyon/SwiftClaude) | 73 | May 2026 (3 days ago at research time) | Most active; async/await, streaming, tool-use macros (`@Tool`, `@ToolInput`), prompt caching (beta), UIImage/NSImage, Swift 6, macOS 15 / iOS 18 |
| [AnthropicSwiftSDK](https://github.com/fumito-ito/AnthropicSwiftSDK) | 18 | Jul 2025 | Messages API, streaming, batch, admin, prompt caching, Bedrock + VertexAI extensions; 22 releases; 100% Swift |
| [AnthropicKit](https://github.com/guitaripod/AnthropicKit) | 3 | Jul 2025 | Thin; low adoption |

**SwiftClaude is the best-maintained option** but carries an explicit pre-1.0 warning: "This package is very new and under-tested. The API can change based on feedback." It requires Swift 6 and platform targets that exclude older devices (macOS 15+, iOS 18+).

**The correct path for coo4one:** Use `URLSession` with `async`/`await` as the ground-truth layer, wrapping the Anthropic HTTP API directly. Layer SwiftClaude (or AnthropicSwiftSDK) on top for convenience, but own the HTTP transport so you can absorb community SDK churn without app-breaking fallout. The API surface is straightforward — one endpoint (`POST /v1/messages`), SSE streaming, JSON everywhere — and Anthropic's own documentation gives the complete wire format. The investment in a thin Swift wrapper over URLSession is ~2–3 days and earns total control.

Pseudo-code for the minimal direct path:

```swift
// ClaudeClient.swift — thin URLSession wrapper
struct ClaudeClient {
    let apiKey: String
    let baseURL = URL(string: "https://api.anthropic.com/v1/messages")!
    
    func stream(request: MessageRequest) -> AsyncThrowingStream<StreamEvent, Error> {
        AsyncThrowingStream { continuation in
            var urlRequest = URLRequest(url: baseURL)
            urlRequest.httpMethod = "POST"
            urlRequest.setValue(apiKey, forHTTPHeaderField: "x-api-key")
            urlRequest.setValue("2023-06-01", forHTTPHeaderField: "anthropic-version")
            urlRequest.setValue("application/json", forHTTPHeaderField: "content-type")
            urlRequest.httpBody = try? JSONEncoder().encode(request)
            
            Task {
                let (bytes, _) = try await URLSession.shared.bytes(for: urlRequest)
                for try await line in bytes.lines {
                    if line.hasPrefix("data: "),
                       let data = line.dropFirst(6).data(using: .utf8),
                       let event = try? JSONDecoder().decode(StreamEvent.self, from: data) {
                        continuation.yield(event)
                        if case .messageDelta(let d) = event, d.stopReason != nil {
                            continuation.finish()
                        }
                    }
                }
            }
        }
    }
}
```

**Maturity gap vs TypeScript/Python:** Significant. The Python SDK (v0.100.0, 3.4k stars, 184 releases) and TypeScript SDK are Anthropic-maintained with full feature parity — prompt caching, compaction, tool use, streaming, batch API, Files API, all betas — and receive same-day support for new API features. A Swift app using community SDKs will lag new Anthropic features by weeks to months. Owning the HTTP layer mitigates this.

---

## 2. Claude Agent SDK — What Ports to Swift?

The "Claude Agent SDK" is Anthropic's [Claude Managed Agents](https://platform.claude.com/docs/en/managed-agents) stack: Agents API + Sessions API + Environments API + Skills API (all in beta as of 2026-04-01). It spins up Anthropic-hosted containers that run the agent loop server-side with tools, MCP servers, and conversation history managed by Anthropic infrastructure.

**What it provides:**
- Versioned agent definitions (system prompt, model, tools, MCP servers bundled into an `agent` resource)
- Stateful sessions with `idle`/`running`/`terminated` lifecycle
- Server-side tool execution (bash, web_search, file ops via `agent_toolset_20260401`)
- MCP vault authentication (Anthropic refreshes OAuth tokens)
- Session history persistence across turns

**What this means for a Swift native app:** The Managed Agents stack is a hosted runtime that abstracts the agentic loop *away from the client*. For coo4one, this is the wrong architecture axis. The loop should run in the Swift process so the app can dispatch OS API calls (EventKit, Contacts, Files) as tool results — which requires your code to be in the loop. Managed Agents are designed for developers who want Anthropic to host the infrastructure; coo4one is building the infrastructure natively.

**Concepts that port directly:**

| Agent SDK concept | Swift equivalent |
|---|---|
| Tool-use loop (`stop_reason: "tool_use"` → execute → `tool_result`) | Implement the same while-loop in Swift; API wire format is identical |
| System prompt + memory prefix | Construct the same JSON payload structure in Swift |
| Versioned agent config | Store as a struct in UserDefaults / CloudKit; manually version |
| Context management / compaction | Call the beta `compact-2026-01-12` header via URLSession; handle `compaction` blocks in history |
| Skills (progressive disclosure) | Implement as Swift functions; inject descriptions into system prompt on demand |
| MCP connector | Either run a local MCP server process or implement the JSON-RPC wire protocol in Swift |

**What has to be reimplemented:** The environment lifecycle (container spin-up, port allocation, process management), the session state machine, and the vault OAuth machinery. For a single-user iOS/macOS app none of this is needed — the "session" is the app run, the "environment" is the device, and OAuth is handled via the iOS keychain and system credential providers.

**The agent loop in Swift is 40 lines:**

```swift
func runAgentLoop(userMessage: String) async throws -> String {
    var messages = conversationHistory
    messages.append(Message(role: .user, content: userMessage))
    
    while true {
        let response = try await client.createMessage(
            model: "claude-opus-4-7",
            system: buildSystemPrompt(),
            messages: messages,
            tools: registeredTools,
            maxTokens: 4096
        )
        
        messages.append(Message(role: .assistant, content: response.content))
        
        if response.stopReason == "end_turn" {
            return response.textContent
        }
        
        if response.stopReason == "tool_use" {
            var toolResults: [ToolResult] = []
            for toolUse in response.toolUseBlocks {
                let result = try await dispatch(toolUse)  // OS API call
                toolResults.append(ToolResult(id: toolUse.id, content: result))
            }
            messages.append(Message(role: .user, content: toolResults))
        }
    }
}
```

---

## 3. Long-Running Session Patterns

Context grows linearly — every turn adds input + output tokens to the next request. For a personal assistant with weeks of history, three strategies apply:

**Strategy A — Server-side compaction (Anthropic beta).** Enable with the `compact-2026-01-12` beta header. When input tokens approach a configured threshold, the API auto-generates a `compaction` block that summarizes older history. On subsequent turns, the API drops everything before the compaction block. Your Swift app only needs to preserve `response.content` and append it to `messages`; the compaction block travels with the history. This is the lowest-integration-cost strategy and is available for `claude-opus-4-7`, `claude-opus-4-6`, and `claude-sonnet-4-6`. Source: [platform.claude.com/docs/en/build-with-claude/compaction](https://platform.claude.com/docs/en/build-with-claude/compaction).

**Strategy B — Client-side rolling summary.** Keep a rolling "session summary" struct updated by a lightweight model (Haiku) after every N turns. On new sessions, prepend the summary as a user-visible context block after the system prompt. This is what the COO does with `coo/episodic_memory.md` — the equivalent for coo4one is a `session_summary.json` in CloudKit that gets prepended on app launch.

**Strategy C — Explicit retrieval (memory-as-retrieval, not memory-as-context).** Store memos, decisions, and facts in a local SQLite database with vector embeddings. On each turn, retrieve the top-K relevant chunks and inject them into the system prompt as a `<memory>` block. This is architecturally cleanest for long-lived personal assistants but requires maintaining an embedding pipeline (see Lane C for details).

**Recommended pattern for coo4one:** Combine all three:
1. Enable server-side compaction as a safety net within each conversation.
2. After each "session" (app become-active → backgrounded), run a Haiku summarization pass and write the session summary to CloudKit.
3. Maintain a SQLite memo/decision store with vector search; prepend top-3 relevant memos on each turn.

The 1M-token window means "full context load" is technically feasible, but Anthropic's documentation explicitly warns of "context rot" as recall degrades with length: *"As conversations get longer, models struggle to maintain focus across the full history."* Source: [platform.claude.com/docs/en/build-with-claude/context-windows](https://platform.claude.com/docs/en/build-with-claude/context-windows). Keep the active context window under ~200K tokens; use retrieval for older material.

---

## 4. Prompt Caching at Scale

Cache mechanics (from [platform.claude.com/docs/en/docs/build-with-claude/prompt-caching](https://platform.claude.com/docs/en/docs/build-with-claude/prompt-caching)):

- **Cache reads cost 0.1x base input price** — 90% discount.
- **5-minute TTL cache writes cost 1.25x; 1-hour TTL costs 2x.**
- Minimum cacheable block: 4096 tokens for Opus 4.7.
- Up to 4 cache breakpoints per request.
- The cache is keyed on exact prefix content — any change invalidates everything downstream.

**Recommended cache block structure for coo4one:**

```
[Block 1 — CACHE, 1h TTL] Identity + system prompt
  ~2K tokens; changes only when user updates preferences.
  cache_control: {type: "ephemeral", ttl: "1h"}

[Block 2 — CACHE, 5m TTL] Active memory prefix
  Top-K retrieved memos + today's context
  ~5-20K tokens; changes per session start, stable within a session.
  cache_control: {type: "ephemeral"}

[Block 3 — CACHE, 5m TTL] Tool definitions
  EventKit, Contacts, Reminders, etc. schemas
  ~1-2K tokens; stable per app version.
  cache_control: {type: "ephemeral"}

[Block 4 — no cache] Conversation history for current session
  Grows turn-by-turn; never static enough to cache reliably.
  Use automatic compaction instead.
```

With this structure, blocks 1–3 are cache hits for every request within a session after the first. For a session with 20 turns and a 15K-token stable prefix, cache hits save ~285K input tokens vs uncached — at Opus 4.7 rates ($5/MTok input vs $0.50/MTok cached), that's $1.425 saved per session at full price, or roughly $1.35 net saving after the write cost.

**Key gotcha:** Place `cache_control` on the *last block that stays identical across requests*, not on the changing block. A timestamp, user name, or dynamic greeting in the system prompt will bust the cache on every turn. Keep the system prompt 100% static; inject per-turn context as a separate user-turn message block.

**For automatic conversational caching** (simplest path): add `cache_control: {type: "ephemeral"}` at the top-level request; the API manages the breakpoint as the conversation grows. Source: compaction docs above.

---

## 5. 1M Context Window Economics

**Model pricing (Opus 4.7, source: [platform.claude.com/docs/en/about-claude/pricing](https://platform.claude.com/docs/en/about-claude/pricing)):**
- Input: $5.00 / MTok
- Output: $25.00 / MTok
- 5-min cache write: $6.25 / MTok
- 1-hr cache write: $10.00 / MTok
- Cache read: $0.50 / MTok

**Note:** Opus 4.7 uses a new tokenizer that uses up to 35% more tokens for the same text than previous models.

**Baseline scenario: 20 turns/day, mostly chat-mode**

Assumptions per turn:
- Average input: 8K tokens (2K system+memory prefix + 6K conversation history)
- Average output: 500 tokens
- Cache hit rate on prefix: 80% (first turn writes, 19 subsequent turns read)

Per-turn cost breakdown (steady state, 80% cache hit):
- 1.6K uncached input tokens: 1,600 × $5/1M = $0.008
- 6.4K cached input tokens: 6,400 × $0.50/1M = $0.0032
- 500 output tokens: 500 × $25/1M = $0.0125
- Per-turn total: ~$0.024

Daily cost (20 turns): ~$0.48
Monthly cost (30 days): **~$14.40/month**

**Sensitivity table:**

| Scenario | Avg input tokens | Cache hit rate | Monthly cost |
|---|---|---|---|
| Light use, high cache | 5K | 90% | ~$8/mo |
| Baseline (above) | 8K | 80% | ~$14/mo |
| Heavy use, tool-augmented | 20K | 70% | ~$45/mo |
| Full 1M context load every turn | 1M | 0% | ~$3,000/mo |

**The 1M context window as daily driver is not economically viable without aggressive caching.** The window exists for occasional deep-research turns, not as a default context size. At 80% cache hit on a 10K stable prefix + 4K variable context, cost is ~$15-20/month — totally acceptable as a personal tool.

**6-month trajectory as memory accumulates:** If the retrieved memory prefix grows from 5K to 30K tokens over 6 months (as memos accumulate), the uncached portion of that prefix grows at 1.25x write cost once, then reads at 0.10x forever. The marginal cost per additional 10K of stable memory prefix is approximately +$0.005/turn uncached vs +$0.0005/turn cached — essentially free after the first cache write per session. Memory growth is not a cost driver if properly cached.

**Sonnet 4.6 alternative:** At $3/MTok input and $15/MTok output (cache reads $0.30/MTok), the baseline scenario drops to ~$8/month. For routine turns, consider tiering: Haiku for background tasks (classification, quick answers at $1/MTok), Sonnet for standard turns, Opus for deep work. This tiering could cut costs by 60-70%.

---

## 6. Tool Use in Native Apps

**Wire format for a client tool call:**

When Claude decides to use a tool, the API response contains a block with `type: "tool_use"`:

```json
{
  "type": "tool_use",
  "id": "toolu_01A09q90qw90lq917835lq9",
  "name": "add_calendar_event",
  "input": {
    "title": "Team sync",
    "start": "2026-05-12T14:00:00",
    "duration_minutes": 60,
    "calendar_id": "work"
  }
}
```

Your Swift code executes the EventKit call, then returns a `tool_result` block in the next user-role message:

```json
{
  "role": "user",
  "content": [{
    "type": "tool_result",
    "tool_use_id": "toolu_01A09q90qw90lq917835lq9",
    "content": "Event created. Event ID: EV-2026-05-12-001"
  }]
}
```

**Defining tools in Swift:**

```swift
let calendarTool = Tool(
    name: "add_calendar_event",
    description: "Creates a new event in the user's calendar. Use for scheduling meetings, appointments, and reminders with specific times.",
    inputSchema: JSONSchema(
        type: .object,
        properties: [
            "title": .init(type: .string, description: "Event title"),
            "start": .init(type: .string, description: "ISO 8601 datetime"),
            "duration_minutes": .init(type: .integer, description: "Duration in minutes"),
            "calendar_id": .init(type: .string, description: "Calendar identifier: 'personal', 'work', or specific calendar name")
        ],
        required: ["title", "start"]
    )
)
```

**Permissions flow for OS APIs:**

EventKit, Contacts, and HealthKit require user authorization before any access. The recommended pattern for coo4one:

1. **Install-time disclosure:** On first launch, present a permissions manifest — "this assistant can read/write your calendar, contacts, and reminders when you ask it to." One authorization per OS framework (not per-action).
2. **At tool invocation, before executing:** For write operations (create event, send message, delete reminder), surface a confirmation step in the UI before calling the OS API. This is the governance gradient from the genesis doc — "with-confirmation as defaults."
3. **Autonomous for reads:** Reading calendar events, contacts, or reminders to answer a question requires no per-action confirmation once the user has granted framework-level access.
4. **Ratchet to autonomous:** After N successful confirmed write operations on a tool, offer to "trust this action" — skip confirmation for that specific tool pattern going forward.

The `strict: true` flag on tool definitions (Anthropic docs) ensures Claude's tool call input always matches your schema exactly, reducing the validation burden before passing arguments to EventKit. Source: [platform.claude.com/docs/en/docs/build-with-claude/tool-use/overview](https://platform.claude.com/docs/en/docs/build-with-claude/tool-use/overview).

---

## 7. Streaming UX Patterns

The Anthropic API streams Server-Sent Events over HTTP. Event types relevant to a native UI:

| SSE event | Swift action |
|---|---|
| `content_block_start` with `type: "thinking"` | Show "thinking..." indicator |
| `content_block_delta` with `type: "text_delta"` | Append text to current message bubble |
| `content_block_start` with `type: "tool_use"` | Show tool-invocation chip ("Checking calendar...") |
| `content_block_stop` | End current bubble / chip |
| `message_delta` with `stop_reason: "tool_use"` | Execute tool, show result chip, continue loop |
| `message_delta` with `stop_reason: "end_turn"` | Finalize; enable user input |

**Pattern: token-by-token with structured stages.** Native Apple apps should avoid "raw token stream to text view" — it produces jittery rendering and no compositional structure. Instead:

```swift
// MessageViewModel.swift
@Observable class MessageViewModel {
    var stages: [MessageStage] = []  // each stage is "thinking" | "text" | "toolCall" | "toolResult"
    var currentStageBuffer = ""
    
    func handleStreamEvent(_ event: StreamEvent) {
        switch event {
        case .contentBlockStart(let block):
            if block.type == "thinking" { stages.append(.thinking("")) }
            else if block.type == "text" { stages.append(.text("")) }
            else if block.type == "tool_use" { stages.append(.toolCall(block.name, [:], pending: true)) }
            
        case .contentDelta(let delta):
            updateLastStage(appending: delta.text ?? delta.thinkingDelta ?? "")
            
        case .messageStop(let stop) where stop.reason == "tool_use":
            Task { await executeTools() }
        }
    }
}
```

**SwiftUI rendering:** Bind each `MessageStage` to a distinct view type — `ThinkingBubble`, `TextBubble`, `ToolCallChip`, `ToolResultChip`. Animate each stage in with a spring transition as it appears. This gives the "agent thinking + acting" feel without raw token flicker.

**Thinking indicator:** For chat-mode turns (no tool use), a simple animated three-dot indicator is sufficient. For tool-augmented turns, show the tool name and a spinner: "Reading calendar..." → "Found 3 events." This makes the agent's behavior legible without overwhelming the UI.

---

## 8. Local-Model Fallback

Three on-device options:

**Apple Foundation Models framework (WWDC 2025):**
- Built into macOS, iOS, iPadOS — no app size increase.
- 3-billion-parameter model, on-device, fully private.
- Capabilities: text summarization, classification, entity extraction, content generation, multi-turn conversation, tool calling (via Swift API).
- Limitations: "not designed for world knowledge or advanced reasoning." No current events, no complex multi-step logic, no math. Requires Apple Intelligence-enabled devices (A17 Pro / M-series) in supported regions.
- Swift API: `LanguageModelSession`, `@Generable` macro for structured outputs, `Tool` protocol for function calling.

**MLX-Swift:**
- Apple Machine Learning Research framework; Metal GPU acceleration on Apple Silicon.
- Supports LLM inference via `mlx-swift-lm`; models loaded from Hugging Face (Mistral, Llama, Phi families).
- 1.8K stars, actively maintained (v0.31.3, Apr 2026).
- Requires model download (multi-GB) and local storage — not suitable for the main app bundle.
- Latency: 20-60 tokens/second on M-series chips for 7B models.

**Ollama:**
- Local model server; runs as a macOS daemon; REST API on localhost.
- macOS only — not available on iOS.
- Good for development/testing; not a viable mobile path.

**When local fallback makes sense for coo4one:**

| Task | Recommended model |
|---|---|
| Quick classification ("is this a reminder or a question?") | Apple Foundation Models |
| Session summarization (background, offline) | Apple Foundation Models or MLX 7B |
| Embedding generation for retrieval | CoreML sentence-transformers or Voyage API |
| Routine turn with simple question | Claude Haiku 4.5 (cloud, cheap at $1/MTok) |
| Complex reasoning, tool use, memory synthesis | Claude Opus 4.7 or Sonnet 4.6 |

The Apple Foundation Models framework is the cleanest on-device path for summarization and classification — zero download, zero latency for app startup, full privacy. Use it for background tasks (session summary, notification classification, offline-mode simple answers) and route complex turns to Anthropic. MLX adds value only if the user explicitly wants full on-device reasoning and is willing to download a model; treat it as a power-user option, not the default.

---

## 9. Cost Projections — Sustained Daily Use

**Monthly cost estimates by tier:**

| Profile | Model | Turns/day | Avg tokens/turn (in/out) | Cache hit % | Monthly |
|---|---|---|---|---|---|
| Light personal use | Haiku 4.5 | 10 | 5K / 300 | 85% | ~$3 |
| Baseline Sonnet | Sonnet 4.6 | 20 | 8K / 500 | 80% | ~$8 |
| Baseline Opus | Opus 4.7 | 20 | 8K / 500 | 80% | ~$14 |
| Heavy Opus + tools | Opus 4.7 | 40 | 20K / 800 | 70% | ~$65 |
| Research-mode (PDFs, long docs) | Opus 4.7 | 10 deep | 100K / 2K | 60% | ~$35 |

**Tiered model recommendation for coo4one:**
- Background tasks (summarization, notification triage): Apple Foundation Models (free)
- Routine chat turns: Haiku 4.5 at $1/MTok — ~$3-5/month baseline
- Standard assistant turns: Sonnet 4.6 — route here by default
- Deep research / multi-tool / memory synthesis: Opus 4.7 — route when explicitly needed

A tiered architecture (auto-route by complexity) puts the typical monthly bill at **$10-20/month** for a single power user, well below what most SaaS tools charge. A flat Opus 4.7 deployment for every turn is viable at ~$14/month at baseline but climbs steeply with tool-augmented turns.

**6-month cost trajectory:** As memory accumulates, the stable prefix grows from ~5K to ~30K tokens. With 1-hour TTL cache writes, this adds approximately $0.30 of write cost per day (one write per session on the growing prefix), negligible against the read savings. The dominant cost driver over 6 months is turns-per-day and output length — memory accumulation itself is nearly cost-neutral with proper caching.

**Key risk:** Tool-augmented agentic turns are expensive. A turn that requires 5 tool calls (read calendar, search contacts, check reminders, fetch email, write memo) generates 5 round-trips with growing context. Each of those intermediate turns costs input tokens on the full accumulated history. For heavy agentic use (40 turns/day with tools), monthly cost can exceed $100 on Opus. The mitigant: use a smaller model for tool-dispatch orchestration and reserve Opus for synthesis turns.

---

## Verdict: Viability of Agent-Loop-in-Native-Swift

**Verdict: viable, with one structural risk.** The Anthropic API is a clean HTTP interface that requires no official Swift SDK — URLSession + async/await handles it. The agent loop (tool-use while loop, compaction, context management) is fully implementable in ~500 lines of Swift. Prompt caching makes the cost profile reasonable for single-user daily use. The Apple Foundation Models framework provides a credible on-device fallback for light tasks. MLX-Swift handles heavier local inference on macOS. Streaming is well-supported via `URLSession.bytes`. The wire format for tools is language-agnostic JSON.

**The path is:**
1. Own a thin URLSession wrapper (2-3 days).
2. Implement the tool-use loop natively (1-2 days).
3. Enable server-side compaction for context management.
4. Structure the system prompt into 3 stable cache blocks + variable conversation tail.
5. Use Apple Foundation Models for offline/background operations.
6. Tier model selection by turn complexity.

**Biggest risk: community SDK fragility.** SwiftClaude (73 stars, pre-1.0) is the only live community SDK. If Anthropic ships a breaking API change (new required field, revised streaming events, new beta header mechanics), the community SDK may not update for weeks or months. If coo4one relies on SwiftClaude as the HTTP layer rather than owning it, that's an uncontrolled breakage vector for the app's core function. The mitigation is owning the HTTP transport and using community SDKs only for convenience helpers (not as the trust anchor).

**Secondary risk: no official SDK means no Anthropic-supported path for beta features.** Features like compaction (`compact-2026-01-12`), the Files API, and Managed Agent Sessions are documented in Python and TypeScript first; Swift developers must translate the beta header patterns manually. This is a 1-2 hour exercise per new feature but requires tracking Anthropic release notes proactively.

**What can be deferred:** The Managed Agents / Sessions API is irrelevant for v0 — it solves multi-user hosted infra, not single-user native app. MCP server integration (for Zotero, Gmail, Outlook) can be bridged initially via REST API adapters in Swift rather than the full MCP JSON-RPC protocol. Implement MCP natively in a v1 iteration once the core loop is validated.

---

*Sources: Anthropic platform docs (platform.claude.com), SwiftClaude GitHub, AnthropicSwiftSDK GitHub, MLX-Swift GitHub, Apple Developer WWDC 2025 session 286, Anthropic engineering blog (effective-context-engineering). All pricing verified against platform.claude.com/docs/en/about-claude/pricing as of 2026-05-11.*

# ADK Analysis: Benefits and Constraints

Pi-go is built on Google's Agent Development Kit (ADK) for Go. This document analyses what the ADK provides, where it falls short, and what pi-go has to supply itself.

---

## How the ADK is used

`Agent.RunStreaming` (`internal/agent/agent.go:321`) delegates directly to the ADK runner:

```
User message → runner.Run (ADK) → LLM (Gemini via genai)
                     ↑                        ↓
              FunctionResponse         FunctionCall part
                     ↑                        ↓
              tool.Run() executed ←── lookup by name in req.Tools
```

The ADK runner owns the **LLM ↔ tool execution** cycle. It sends the conversation to the LLM, receives streaming SSE events, executes any `FunctionCall` parts by looking up the tool in `req.Tools`, injects the result as a `FunctionResponse`, and loops until the LLM emits a text-only response.

Pi-go's `runAgentLoop` (`internal/tui/agent_loop.go:284`) is a **passive observer** of the event stream — it translates ADK events into TUI messages but does not drive the loop itself.

---

## Benefits

### 1. The inner loop is free

The hardest part of an agentic system — the LLM→FunctionCall→tool.Run()→FunctionResponse→LLM cycle — is handled entirely by the ADK runner. Without ADK, you would build that state machine yourself: accumulating partial SSE chunks, detecting `stop_reason: tool_use`, dispatching, serialising results back into the conversation history, and looping.

### 2. Session / conversation history management

The ADK `session.Service` (backed by `NewFileService`, a file store at `internal/session/store.go`) maintains the full multi-turn history. The agent calls `runner.Run(ctx, userID, sessionID, msg)` and the runner stitches new turns onto the existing history automatically. Pi-go wrote no conversation accumulation code.

### 3. Clean Go iterator API

`RunStreaming` returns `iter.Seq2[*session.Event, error]`, a standard Go 1.23 range-over-func. The TUI goroutine iterates with a plain `for ev, err := range ...` — no callback hell, no channels on the LLM side, no custom SSE parser.

### 4. Tool declaration plumbing

`coercingTool.ProcessRequest` (`internal/tools/registry.go:226`) registers each tool's `genai.FunctionDeclaration` into the LLM request automatically. Adding a tool is `newTool(name, description, handler)` — the JSON schema and wiring are handled by the ADK.

### 5. Multi-agent / subagent primitives

The `subagent` tool (`internal/tools/subagent.go`) and the A2A tool (`internal/tools/a2a.go`) build on ADK's agent composition model. Spawning child agents that report back is supported structurally rather than ad-hoc.

---

## Constraints

### 1. Locked to Gemini / Google

`buildRunner` (`internal/agent/agent.go:204`) passes `cfg.Model` to `llmagent.New`, but the underlying `runner.Runner` and `genai` types are Google ADK / Vertex AI primitives. Supporting Ollama, OpenAI, Anthropic, or any other provider requires either a shim that speaks `genai` semantics or abandoning ADK's runner entirely. This is a hard constraint for any multi-provider roadmap.

### 2. No mid-loop interception

Pi-go watches the event stream but has no way to inject logic between a tool result and the LLM seeing it. The compactor (`internal/tools/compactor.go`) exists because large tool outputs blow up the context window, but it must be applied *inside* each tool handler — not as a post-processing step — because ADK feeds results back to the LLM without a caller hook between them.

### 3. Retry lives outside the ADK loop

`isTransient` and the retry wrapper (`internal/agent/retry.go`) are pi-go's own code. ADK does not surface transient 429/5xx retries. The detection is pure string-matching on error messages, which is fragile. This gap exists because the ADK runner exposes no retry callback.

### 4. Stuck-loop detection is bolted on

`stuckDetector` (`internal/tui/agent_loop.go:91`) is entirely pi-go code because the ADK runner has no infinite-loop guard. If the LLM calls the same tool repeatedly, ADK will loop forever. Pi-go intercepts `FunctionCall` events and aborts when it detects either a consecutive-call streak or a repeating cycle (AB AB AB pattern) in a sliding window.

### 5. ADK is an early-stage, Google-controlled dependency

The Go ADK is newer and less mature than the Python version. API stability, multi-provider support, and feature parity are all risks. If Google changes the runner interface, session schema, or event model, pi-go must follow.

### 6. Tool argument coercion is pi-go's problem

The `coercingTool` wrapper (`internal/tools/registry.go:198`) with `aliasArgs`, `intProps`, `boolProps`, and `jsonProps` exists because LLMs regularly call tools with wrong argument types or slightly wrong field names. ADK passes raw LLM JSON straight to the handler with no normalisation, so pi-go built an entire layer to coerce types and resolve aliases. A more opinionated framework would absorb this.

---

## Summary

| Concern | Provided by ADK | Provided by pi-go |
|---|---|---|
| LLM ↔ tool inner loop | Yes | — |
| Conversation history | Yes | — |
| Streaming iterator API | Yes | — |
| Tool declaration wiring | Yes | — |
| Multi-agent composition | Yes | — |
| Retry on transient errors | No | `internal/agent/retry.go` |
| Stuck-loop detection | No | `stuckDetector` in `agent_loop.go` |
| Tool argument coercion | No | `coercingTool` in `registry.go` |
| Context window compaction | No | `compactor.go` (inside handlers) |
| Multi-provider support | No | Not yet possible |

ADK is a good fit for the happy path: it eliminates the LLM↔tool state machine, conversation history, and streaming plumbing. The cost is lock-in to Gemini, loss of mid-loop control, and gaps that pi-go fills with bespoke code. As long as Gemini is the target provider and ADK remains stable, it is a net win. If multi-provider support or tighter control over the loop become requirements, ADK becomes the primary constraint.

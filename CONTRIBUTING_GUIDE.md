# Shelley Architecture Guide for Contributors

## What Shelley Is

Shelley is a web-based AI coding agent. A user types a message in a browser,
an LLM responds (possibly calling tools like bash or file editing), and the
results stream back in real time. That's the whole idea.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Browser (React)                       │
│                                                         │
│  ChatInterface.tsx ◄──── SSE ────► handlers.go          │
│  (renders messages,      │        (REST + SSE endpoints)│
│   tool results,          │                              │
│   streaming)             │                              │
└──────────────────────────┼──────────────────────────────┘
                           │
┌──────────────────────────┼──────────────────────────────┐
│                   Go Server                             │
│                          │                              │
│  ┌───────────────────────▼────────────────────────┐     │
│  │           ConversationManager                  │     │
│  │  (one per active conversation)                 │     │
│  │                                                │     │
│  │  • Hydrates from DB on first message           │     │
│  │  • Owns the Loop instance                      │     │
│  │  • Manages message queue                       │     │
│  │  • Publishes updates via SubPub → SSE          │     │
│  └────────────────────┬───────────────────────────┘     │
│                       │                                 │
│  ┌────────────────────▼───────────────────────────┐     │
│  │              Loop (loop.go)                    │     │
│  │                                                │     │
│  │  repeat:                                       │     │
│  │    1. Send history + tools to LLM              │     │
│  │    2. If response has tool calls → execute     │     │
│  │    3. Record results → notify subscribers      │     │
│  │    4. Go to 1 (until LLM says stop)            │     │
│  └───────┬────────────────────┬───────────────────┘     │
│          │                    │                         │
│  ┌───────▼──────┐    ┌───────▼──────────┐               │
│  │ LLM Service  │    │   Tool Execution │               │
│  │              │    │                  │               │
│  │ • Anthropic  │    │ • bash           │               │
│  │ • OpenAI     │    │ • patch (files)  │               │
│  │ • Gemini     │    │ • keyword_search │               │
│  │ • Fireworks  │    │ • subagent       │               │
│  │ • etc.       │    │ • browse         │               │
│  └──────────────┘    └──────────────────┘               │
│                                                         │
│  ┌──────────────────────────────────────────────┐       │
│  │              SQLite (db/)                     │       │
│  │  conversations → messages → llm_requests      │       │
│  └──────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────┘
```

## Message Lifecycle

This is the most important flow to understand. Everything else is plumbing.

```
User types message in browser
        │
        ▼
POST /conversation/{id}/chat          ← handlers.go:handleChatConversation()
        │
        ▼
getOrCreateConversationManager()      ← server.go:671
        │
        ▼
ConversationManager.AcceptUserMessage()  ← convo.go:225
  │  Records message to DB immediately
  │  Calls ensureLoop() if first message
  │
  ▼
Loop.Go() running in background       ← loop.go:122
  │  Picks up queued message
  │
  ▼
Loop.processLLMRequest()               ← loop.go:198
  │  Sends full history + tools to LLM
  │  Calls llmService.Do(ctx, req)
  │
  ▼
LLM responds ──── stop_reason?
  │                    │
  │ "end_turn"         │ "tool_use"
  │                    │
  ▼                    ▼
  Done          Loop.executeToolCalls()  ← loop.go:422
                  │  Runs tools in parallel
                  │  Records results to DB
                  │
                  ▼
                Loop back to processLLMRequest()
```

**Every message recorded triggers:**
```
recordMessage()                        ← server.go:771
  │  Saves to SQLite
  │
  ▼
notifySubscribersNewMessage()          ← server.go:909
  │
  ▼
manager.subpub.Publish()               → SSE to all connected browsers
```

## SSE Streaming

```
Browser                              Server
  │                                    │
  │  GET /conversation/{id}/stream     │
  │ ──────────────────────────────►    │
  │                                    │  handleStreamConversation()
  │                                    │  ← handlers.go:883
  │    initial: all messages + state   │
  │ ◄──────────────────────────────    │
  │                                    │
  │    SubPub.Subscribe(lastSeqID)     │
  │         ┌──────────────────────    │
  │         │  (waits for new msgs)    │
  │         │                          │
  │         │   heartbeat every 30s    │
  │ ◄───────┤                          │
  │         │                          │
  │         │   new message published  │
  │ ◄───────┤                          │
  │         │                          │
  │         │   another message        │
  │ ◄───────┤                          │
  │         │        ...               │
```

## Subagent Flow

A conversation can spawn child conversations to do subtasks.

```
Parent Loop executing tools
        │
        ▼
SubagentTool.Run()                     ← claudetool/subagent.go:142
        │
        ▼
SubagentRunner.RunSubagent()           ← server/subagent.go:27
  │  Creates child conversation in DB
  │  Creates child ConversationManager
  │  Child gets its own Loop instance
  │
  ├── wait=true:  polls until child finishes or timeout
  │               returns child's last response
  │
  └── wait=false: returns immediately
                  child runs in background
```

## Files You Need to Know

### Tier 1: The Core (read these first)

| File | Lines | What It Does |
|------|-------|-------------|
| `loop/loop.go` | ~700 | **The agentic loop.** Send to LLM → execute tools → repeat. Start here. |
| `server/convo.go` | ~1,070 | **Conversation manager.** Owns the loop, manages message queue, hydration. |
| `server/handlers.go` | ~1,460 | **HTTP endpoints.** Chat, stream, conversations list, models. |
| `server/server.go` | ~1,440 | **Server setup.** Routing, manager creation, message recording. |
| `llm/llm.go` | ~150 | **LLM interface.** The `Service` interface all providers implement. |

### Tier 2: Tools and Providers (read when modifying specific features)

| File | Lines | What It Does |
|------|-------|-------------|
| `claudetool/toolset.go` | ~330 | Tool registration. Which tools the LLM gets. |
| `claudetool/bash.go` | — | Shell command execution. |
| `claudetool/patch.go` | — | File editing (replace, append, overwrite). |
| `claudetool/subagent.go` | — | Spawning child conversations. |
| `llm/ant/ant.go` | — | Anthropic (Claude) provider. |
| `llm/oai/oai.go` | — | OpenAI-compatible provider. |
| `llm/gem/gem.go` | — | Google Gemini provider. |

### Tier 3: Data and UI

| File | Lines | What It Does |
|------|-------|-------------|
| `db/db.go` | — | Database operations. Conversations, messages, queries. |
| `db/schema/` | — | SQLite migrations (numbered 001-017). |
| `ui/src/components/ChatInterface.tsx` | ~2,590 | Main UI component. Message rendering, tool display, streaming. |
| `ui/src/services/api.ts` | — | Frontend REST/SSE client. |
| `models/models.go` | ~600 | Model registry. All supported models and their factories. |

### Tier 4: Supporting Packages

| Package | Purpose |
|---------|---------|
| `subpub/` | Generic pub/sub for SSE streaming. |
| `slug/` | LLM-powered human-readable conversation IDs. |
| `gitstate/` | Tracks git repo state (branch, commit). |
| `skills/` | Plugin system following agentskills.io spec. |
| `templates/` | Embedded project templates. |
| `server/notifications/` | Discord, email, ntfy notification channels. |

## How to Add a New Tool

```
1. Create claudetool/yourtool.go
   - Define a struct with a Tool() method returning *llm.Tool
   - The Tool has Name, Description, InputSchema, and Run function

2. Register it in claudetool/toolset.go → NewToolSet()

3. Add UI rendering in ui/src/components/ChatInterface.tsx
   - Add to TOOL_COMPONENTS map

4. Add to predictable model demo in loop/predictable.go

5. Build and test:
   make ui
   go test ./claudetool/ ./server/ ./loop/
```

## How to Add a New LLM Provider

```
1. Create llm/yourprovider/provider.go
   - Implement the llm.Service interface:
     Do(ctx, *Request) (*Response, error)
     TokenContextWindow() int
     MaxImageDimension() int

2. Register model(s) in models/models.go → All()

3. Build and test:
   go test ./llm/yourprovider/ ./models/
```

## Key Interfaces

```go
// The only interface an LLM provider must implement (llm/llm.go)
type Service interface {
    Do(context.Context, *Request) (*Response, error)
    TokenContextWindow() int
    MaxImageDimension() int
}

// A tool given to the LLM (llm/llm.go)
type Tool struct {
    Name        string
    Description string
    InputSchema json.RawMessage
    Run         func(ctx context.Context, input json.RawMessage) ToolOut
}
```

## Build and Test

```bash
make ui                    # Build the frontend (required before Go tests)
go test ./...              # Run all Go tests
cd ui && pnpm run lint     # Lint frontend
cd ui && pnpm run test:e2e # E2E tests (needs built binary)
make serve-test            # Run with predictable model (no API keys needed)
```

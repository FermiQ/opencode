# agent.go

## Overview

The `agent.go` file in the `internal/llm/agent` package defines the core AI agent (`Service`) functionality. This includes managing interactions with an LLM provider, handling tool use, processing user prompts within sessions, managing agent lifecycle (cancellation, busy status), generating titles for conversations, and summarizing conversations. It uses a publish-subscribe mechanism to broadcast agent events.

## Key Components

### Enums and Types
- `AgentEventType`: String enum (`error`, `response`, `summarize`) for types of events published by the agent.
- `AgentEvent`: Struct representing an event from the agent.
    - `Type (AgentEventType)`
    - `Message (message.Message)`: The AI's response message (for `AgentEventTypeResponse`).
    - `Error (error)`: Any error that occurred (for `AgentEventTypeError`).
    - `SessionID (string)`: Relevant session ID (for `AgentEventTypeSummarize`).
    - `Progress (string)`: Progress message during summarization.
    - `Done (bool)`: Indicates if the operation (e.g., summarization) is complete.

### Interfaces
- `Service`: Defines the contract for an AI agent.
    - `pubsub.Suscriber[AgentEvent]`: Embeds pub/sub, allowing subscription to agent events.
    - `Model() models.Model`: Returns the underlying LLM model information.
    - `Run(ctx context.Context, sessionID string, content string, attachments ...message.Attachment) (<-chan AgentEvent, error)`: Main method to process a user's prompt (`content` and `attachments`) within a `sessionID`. Returns a channel for `AgentEvent`s.
    - `Cancel(sessionID string)`: Cancels an active request for the given `sessionID`.
    - `IsSessionBusy(sessionID string) bool`: Checks if a specific session has an active request.
    - `IsBusy() bool`: Checks if any session has an active request for this agent.
    - `Update(agentName config.AgentName, modelID models.ModelID) (models.Model, error)`: Updates the agent's underlying LLM model.
    - `Summarize(ctx context.Context, sessionID string) error`: Initiates a summarization of the conversation in `sessionID`.

### Structs
- `agent`: Concrete implementation of the `Service` interface.
    - `Broker (*pubsub.Broker[AgentEvent])`: For publishing agent events.
    - `sessions (session.Service)`: Session management service.
    - `messages (message.Service)`: Message management service.
    - `tools ([]tools.BaseTool)`: List of tools available to this agent.
    - `provider (provider.Provider)`: The LLM provider for primary interactions.
    - `titleProvider (provider.Provider)`: Separate LLM provider for generating conversation titles (used by Coder agent).
    - `summarizeProvider (provider.Provider)`: Separate LLM provider for summarizing conversations (used by Coder agent).
    - `activeRequests (sync.Map)`: Stores cancel functions for active requests, keyed by session ID.

### Constants and Variables (Error types)
- `ErrRequestCancelled`: Error indicating the user cancelled the request.
- `ErrSessionBusy`: Error indicating the session is already processing a request.

### Core Functions
- `NewAgent(agentName config.AgentName, sessions session.Service, messages message.Service, agentTools []tools.BaseTool) (Service, error)`: Constructor for `agent`. Initializes providers based on `agentName` and configuration.
- `(a *agent) Model()`: Returns the main provider's model.
- `(a *agent) Cancel(sessionID string)`: Implements cancellation logic using `activeRequests`.
- `(a *agent) IsBusy()`, `(a *agent) IsSessionBusy(sessionID string)`: Implement busy status checks.
- `(a *agent) generateTitle(ctx context.Context, sessionID string, content string) error`: Asynchronously generates a title for a new conversation using `titleProvider`.
- `(a *agent) Run(...)`: The main entry point for processing a user prompt.
    - Checks for session busy status.
    - Sets up a cancellable context for the request.
    - Spawns a goroutine for `processGeneration`.
    - Returns a channel for agent events.
- `(a *agent) processGeneration(ctx context.Context, sessionID, content string, attachmentParts []message.ContentPart) AgentEvent`: Orchestrates the generation flow.
    - Generates a title if it's a new conversation.
    - Handles summarized sessions by adjusting message history.
    - Enters a loop:
        - Calls `streamAndHandleEvents` to interact with the LLM provider.
        - If the LLM requests tool use, appends tool results to history and continues the loop.
        - Otherwise, returns the final agent response.
- `(a *agent) createUserMessage(...)`: Creates and stores the user's message.
- `(a *agent) streamAndHandleEvents(ctx context.Context, sessionID string, msgHistory []message.Message) (message.Message, *message.Message, error)`:
    - Creates an initial assistant message in the database.
    - Streams events from `a.provider.StreamResponse()`.
    - Processes each event (`processEvent`): appends content/tool calls to the assistant message, updates it in DB.
    - If tool calls are present, executes them, creates a tool response message, and returns it.
- `(a *agent) finishMessage(...)`: Helper to update a message with a finish reason.
- `(a *agent) processEvent(...)`: Handles different `provider.ProviderEvent` types (content delta, tool use, error, completion), updating the `assistantMsg`.
- `(a *agent) TrackUsage(...)`: Updates session token counts and cost based on provider usage.
- `(a *agent) Update(...)`: Updates the agent's model by creating a new provider.
- `(a *agent) Summarize(...)`: Initiates conversation summarization in a goroutine using `summarizeProvider`. Publishes progress and final result/error via `AgentEvent`. Stores the summary as a special message linked to the session.
- `createAgentProvider(agentName config.AgentName) (provider.Provider, error)`: Helper to create an LLM provider instance based on configuration for a given `agentName`. Sets up model, API key, system prompt, max tokens, and provider-specific options (e.g., reasoning effort for OpenAI, thinking function for Anthropic).

## Usage Examples

This service is typically instantiated and used by higher-level application logic (e.g., in `internal/app/app.go` or TUI handlers).

```go
// Initialization (conceptual, in app setup)
// import "github.com/opencode-ai/opencode/internal/llm/agent"
// import "github.com/opencode-ai/opencode/internal/config"

// agentService, err := agent.NewAgent(
//     config.AgentCoder,
//     sessionService, // instance of session.Service
//     messageService, // instance of message.Service
//     coderAgentTools, // []tools.BaseTool
// )

// Running a prompt
// eventChan, err := agentService.Run(context.Background(), "session-123", "Explain this Go code: ...")
// if err != nil { /* handle */ }
// for event := range eventChan {
//     switch event.Type {
//     case agent.AgentEventTypeResponse:
//         // process event.Message
//     case agent.AgentEventTypeError:
//         // process event.Error
//     }
// }

// Cancelling a request
// agentService.Cancel("session-123")
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `config`: For agent configurations (model, max tokens, etc.) and updating model choices.
    - `llm/models`: For `Model` struct and model IDs.
    - `llm/prompt`: For getting agent-specific system prompts.
    - `llm/provider`: For `Provider` interface and events.
    - `llm/tools`: For `BaseTool` interface and tool call/response structures.
    - `logging`: For application-wide logging.
    - `message`: For `Message` struct, `message.Service`, and message types/roles.
    - `permission`: For `ErrorPermissionDenied` (though handled by tools, not directly here).
    - `pubsub`: For `Broker` to publish `AgentEvent`s.
    - `session`: For `Session` struct and `session.Service`.
- **External Libraries:** `sync` (for `sync.Map`), `errors`, `fmt`, `strings`, `time`, `context`.
- **Interactions:**
    - Central orchestrator for LLM interactions.
    - Manages the state of active LLM requests and supports cancellation.
    - Iteratively communicates with an `llm/provider.Provider`, handling content streaming and tool execution cycles.
    - Persists conversation messages and session metadata using `message.Service` and `session.Service`.
    - Publishes events about its progress and results.

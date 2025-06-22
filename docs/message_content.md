# content.go (in internal/message)

## Overview

The `content.go` file, part of the `internal/message` package, defines the core structures for representing messages in a conversation with an LLM. This includes the `Message` struct itself, different types of `ContentPart` (like text, images, tool calls, tool results, reasoning steps), message roles, and finish reasons. The design supports multimodal messages and complex interactions involving LLM tool use.

## Key Components

### Enums and Types
- `MessageRole (string)`: Defines the originator of a message.
    - `Assistant`: Message from the AI.
    - `User`: Message from the human user.
    - `System`: Initial instructions or context for the AI.
    - `Tool`: Response from a tool execution.
- `FinishReason (string)`: Indicates why an assistant's turn ended.
    - `FinishReasonEndTurn`: Normal completion.
    - `FinishReasonMaxTokens`: Stopped due to token limit.
    - `FinishReasonToolUse`: Stopped because it needs to use a tool.
    - `FinishReasonCanceled`: User cancelled the request.
    - `FinishReasonError`: An error occurred.
    - `FinishReasonPermissionDenied`: Tool use was denied.
    - `FinishReasonUnknown`: Unknown reason.
- `ContentPart (interface)`: An interface with a marker method `isPart()`. All specific content types (text, image, tool call, etc.) implement this.

### Content Part Structs
These structs implement `ContentPart`:
- `ReasoningContent`: For LLM "thinking" steps or internal monologue.
    - `Thinking (string)`
- `TextContent`: Plain text content.
    - `Text (string)`
- `ImageURLContent`: Represents an image provided via a URL.
    - `URL (string)`
    - `Detail (string, omitempty)`: Optional detail level (e.g., "low", "high" for some APIs).
- `BinaryContent`: Represents binary data, typically for images uploaded directly.
    - `Path (string)`: Original file path (for reference).
    - `MIMEType (string)`
    - `Data ([]byte)`: Raw binary data.
    - `String(provider models.ModelProvider) string`: Method to get a string representation, base64 encoding the data. For OpenAI, it prepends the data URI scheme.
- `ToolCall`: Represents a tool invocation requested by the LLM.
    - `ID (string)`: Unique ID for the tool call.
    - `Name (string)`: Name of the tool to call.
    - `Input (string)`: JSON string of arguments for the tool.
    - `Type (string)`: Type of the tool (e.g., "function").
    - `Finished (bool)`: Flag indicating if the tool call part is complete (e.g., input streaming finished).
- `ToolResult`: Represents the output/result from a tool execution.
    - `ToolCallID (string)`: ID of the `ToolCall` this result corresponds to.
    - `Name (string)`: Name of the tool that was called.
    - `Content (string)`: Output from the tool.
    - `Metadata (string)`: Additional metadata from the tool.
    - `IsError (bool)`: True if the tool execution resulted in an error.
- `Finish`: Marks the end of an assistant's turn, including the reason.
    - `Reason (FinishReason)`
    - `Time (int64)`: Unix timestamp of when the finish occurred.

### Message Struct
- `Message`: The main struct representing a single message in a conversation.
    - `ID (string)`: Unique ID for the message.
    - `Role (MessageRole)`: Who sent the message.
    - `SessionID (string)`: ID of the session this message belongs to.
    - `Parts ([]ContentPart)`: A slice holding various parts of the message content (text, images, tool calls, etc.). This allows for multimodal and multi-part messages.
    - `Model (models.ModelID)`: The ID of the LLM model that generated this message (if applicable, usually for assistant messages).
    - `CreatedAt (int64)`: Unix timestamp of creation.
    - `UpdatedAt (int64)`: Unix timestamp of last update.

### Methods on `Message`
- `Content() TextContent`: Returns the first `TextContent` part, or an empty one.
- `ReasoningContent() ReasoningContent`: Returns the first `ReasoningContent` part.
- `ImageURLContent() []ImageURLContent`: Returns all `ImageURLContent` parts.
- `BinaryContent() []BinaryContent`: Returns all `BinaryContent` parts.
- `ToolCalls() []ToolCall`: Returns all `ToolCall` parts.
- `ToolResults() []ToolResult`: Returns all `ToolResult` parts.
- `IsFinished() bool`: Checks if the message contains a `Finish` part.
- `FinishPart() *Finish`: Returns the `Finish` part, if any.
- `FinishReason() FinishReason`: Returns the reason from the `Finish` part.
- `IsThinking() bool`: Heuristic to check if the assistant is in a "thinking" state (has reasoning content but no main text content and isn't finished).
- `AppendContent(delta string)`: Appends `delta` to the existing `TextContent` part, or creates one if none exists.
- `AppendReasoningContent(delta string)`: Appends `delta` to `ReasoningContent`.
- `FinishToolCall(toolCallID string)`: Marks a specific `ToolCall` part as finished.
- `AppendToolCallInput(toolCallID string, inputDelta string)`: Appends `inputDelta` to a specific `ToolCall`'s input (for streaming tool inputs).
- `AddToolCall(tc ToolCall)`: Adds or updates a `ToolCall` part.
- `SetToolCalls(tc []ToolCall)`: Replaces all existing `ToolCall` parts with the provided slice.
- `AddToolResult(tr ToolResult)`: Appends a `ToolResult` part.
- `SetToolResults(tr []ToolResult)`: Appends multiple `ToolResult` parts.
- `AddFinish(reason FinishReason)`: Adds or updates the `Finish` part.
- `AddImageURL(url, detail string)`: Appends an `ImageURLContent` part.
- `AddBinary(mimeType string, data []byte)`: Appends a `BinaryContent` part.

## Important Variables/Constants
- `MessageRole` constants (Assistant, User, System, Tool).
- `FinishReason` constants.

## Usage Examples

Creating a user message with text and an image:
```go
// import "github.com/opencode-ai/opencode/internal/message"
// import "github.com/opencode-ai/opencode/internal/llm/models" // For models.ModelID

userMsg := message.Message{
    ID:        "msg-1",
    Role:      message.User,
    SessionID: "session-abc",
    Parts: []message.ContentPart{
        message.TextContent{Text: "What is in this image?"},
        message.BinaryContent{MIMEType: "image/png", Data: imageDataBytes},
    },
    CreatedAt: time.Now().Unix(),
    UpdatedAt: time.Now().Unix(),
}
```

An assistant responding with text and a tool call:
```go
assistantMsg := message.Message{
    ID:        "msg-2",
    Role:      message.Assistant,
    SessionID: "session-abc",
    Model:     models.ModelID("gpt-4o"), // Example model
    Parts: []message.ContentPart{
        message.TextContent{Text: "I need to search for that."},
        message.ToolCall{ID: "toolcall-123", Name: "glob", Input: `{"pattern": "*.txt"}`},
    },
    // ...
}
// Later, after tool execution, a Finish part would be added:
// assistantMsg.AddFinish(message.FinishReasonToolUse)
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/llm/models`: For `models.ModelID` and `models.ModelProvider` (used in `BinaryContent.String()`).
- **External Libraries:**
    - `encoding/base64`, `slices` (Go 1.21+), `time`: Standard Go libraries.
- **Interactions:**
    - This file defines the fundamental data structures for representing conversational messages exchanged between the user, the AI agent, and tools.
    - The `Parts []ContentPart` slice is key to supporting rich, multimodal messages and complex agent interactions involving reasoning steps and tool use.
    - These structures are used by:
        - `message.Service` (`message.go`) for creating, storing, and retrieving messages.
        - LLM provider implementations (`internal/llm/provider/`) to convert these internal message formats to the specific formats required by different LLM APIs.
        - The AI agent (`internal/llm/agent/agent.go`) to manage conversation history and process LLM responses.
        - The TUI (`internal/tui/`) for displaying messages.

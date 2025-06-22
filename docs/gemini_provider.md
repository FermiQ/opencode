# gemini.go (in internal/llm/provider)

## Overview

The `gemini.go` file, part of the `internal/llm/provider` package, implements the `Provider` interface (defined in `provider.go`) to enable interaction with Google's Gemini family of models. It handles the specifics of the Gemini API, including message and tool format conversions, streaming responses, and error handling with retries.

## Key Components

### Structs
- `geminiOptions`: Holds Gemini-specific options. Currently, only `disableCache` is defined (though Gemini's caching mechanism might differ from others).
- `geminiClient`: The concrete implementation of the `Provider` interface for Gemini.
    - `providerOptions (providerClientOptions)`: Common provider options (API key, model, system message, max tokens).
    - `options (geminiOptions)`: Gemini-specific options.
    - `client (*genai.Client)`: The client from the `google.golang.org/genai` library.

### Types
- `GeminiOption`: A functional option type for configuring `geminiOptions`.
- `GeminiClient`: An alias for `ProviderClient` (alias for `Provider` interface).

### Core Functions
- `newGeminiClient(opts providerClientOptions) GeminiClient`: Constructor for `geminiClient`. Initializes the `genai.Client` using the provided API key.
- `(g *geminiClient) convertMessages(messages []message.Message) []*genai.Content`: Converts OpenCode's internal `message.Message` slice into the `[]*genai.Content` format required by the Gemini SDK. It handles:
    - User messages: Converts text and binary content (images) into `genai.Part`s.
    - Assistant messages: Converts text content and tool calls (`message.ToolCall`) into `genai.Part`s with `FunctionCall`.
    - Tool messages: Converts `message.ToolResult` into `genai.Part`s with `FunctionResponse`.
- `(g *geminiClient) convertTools(tools []tools.BaseTool) []*genai.Tool`: Converts OpenCode's `tools.BaseTool` definitions into the `[]*genai.Tool` format for the Gemini API. This involves mapping parameter schemas.
- `(g *geminiClient) finishReason(reason genai.FinishReason) message.FinishReason`: Maps Gemini's API finish reasons (e.g., `genai.FinishReasonStop`) to OpenCode's `message.FinishReason`.
- `(g *geminiClient) send(ctx context.Context, messages []message.Message, tools []tools.BaseTool) (*ProviderResponse, error)`: Implements non-streaming message sending.
    - Converts messages and tools.
    - Creates a chat session with `g.client.Chats.Create()`, providing history and configuration.
    - Sends the last message using `chat.SendMessage()`.
    - Handles retries for rate limits with exponential backoff.
    - Converts the Gemini response (text and tool calls) into a `ProviderResponse`.
- `(g *geminiClient) stream(ctx context.Context, messages []message.Message, tools []tools.BaseTool) <-chan ProviderEvent`: Implements streaming message sending.
    - Similar setup to `send` for creating a chat session.
    - Uses `chat.SendMessageStream()` to get a stream of responses.
    - Iterates through the stream, converting Gemini response parts (text deltas, function calls) into OpenCode `ProviderEvent`s (e.g., `EventContentDelta`, `EventToolUseStart`, `EventComplete`) and sends them on the returned channel.
    - Accumulates the full message content and tool calls.
    - Handles retries for rate limit errors encountered during streaming.
- `(g *geminiClient) shouldRetry(attempts int, err error) (bool, int64, error)`: Determines if an API call should be retried. For Gemini, it checks the error message string for common rate limit indicators as there isn't a standard typed error for this from the SDK. Calculates backoff time.
- `(g *geminiClient) toolCalls(resp *genai.GenerateContentResponse) []message.ToolCall`: Extracts tool call information from a Gemini response.
- `(g *geminiClient) usage(resp *genai.GenerateContentResponse) TokenUsage`: Extracts token usage from a Gemini response.

### Functional Options
- `WithGeminiDisableCache() GeminiOption`: Option to set `disableCache` (currently informational as Gemini caching details might differ).

### Helper Functions
- `parseJsonToMap(jsonStr string) (map[string]interface{}, error)`: Parses a JSON string into a map.
- `convertSchemaProperties(parameters map[string]interface{}) map[string]*genai.Schema`: Converts OpenCode tool parameter schema to Gemini's `genai.Schema` format.
- `convertToSchema(param interface{}) *genai.Schema`: Converts a single parameter definition.
- `processArrayItems(paramMap map[string]interface{}) *genai.Schema`: Processes items for array-type parameters.
- `mapJSONTypeToGenAI(jsonType string) genai.Type`: Maps JSON schema types to `genai.Type` enum values.
- `contains(s string, substrs ...string) bool`: Helper to check if a string contains any of a list of substrings (case-insensitive), used for error message parsing in `shouldRetry`.

## Important Variables/Constants
This file does not define exported package-level constants or variables beyond the option types and constructor.

## Usage Examples

This client is instantiated via the generic `provider.NewProvider` function when `models.ProviderGemini` is specified.

```go
// Conceptual instantiation via generic provider factory:
// import "github.com/opencode-ai/opencode/internal/llm/provider"
// import "github.com/opencode-ai/opencode/internal/llm/models"

// opts := []provider.ProviderClientOption{
//     provider.WithAPIKey("your_gemini_api_key"),
//     provider.WithModel(models.SupportedModels[models.Gemini25Flash]),
//     provider.WithSystemMessage("You are a helpful assistant using Gemini."),
//     provider.WithMaxTokens(1024),
//     // provider.WithGeminiOptions(provider.WithGeminiDisableCache()), // If needed
// }
// geminiProvider, err := provider.NewProvider(models.ProviderGemini, opts...)

// Streaming a response:
// eventChannel := geminiProvider.StreamResponse(context.Background(), messages, tools)
// for event := range eventChannel {
//     // process event
// }
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For debug logging configuration.
    - `github.com/opencode-ai/opencode/internal/llm/models`: For `ModelProvider` and `Model` definitions.
    - `github.com/opencode-ai/opencode/internal/llm/tools`: For `tools.BaseTool` interface.
    - `github.com/opencode-ai/opencode/internal/logging`: For application logging.
    - `github.com/opencode-ai/opencode/internal/message`: For `message.Message`, `message.ToolCall`, etc.
    - Depends on types and interfaces from `provider.go` in the same package.
- **External Libraries:**
    - `github.com/google/uuid`: For generating unique IDs for tool calls.
    - `google.golang.org/genai`: The official Google AI Go SDK for Gemini.
- **Interactions:**
    - Bridges OpenCode's generic LLM interaction patterns with the Gemini API.
    - Translates message formats, tool definitions (including complex schemas), and event types.
    - Manages API call complexities like streaming, error handling, and retries (with string matching for rate limit errors).
    - Handles the chat session model of the Gemini API (`client.Chats`).

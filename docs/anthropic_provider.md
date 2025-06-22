# anthropic.go (in internal/llm/provider)

## Overview

The `anthropic.go` file, part of the `internal/llm/provider` package, implements the `Provider` interface (defined in `provider.go`) for interacting with Anthropic's Claude models. It handles the specifics of the Anthropic API, including message formatting, tool use conversion, streaming responses, and error handling with retries. It also supports using Anthropic models via AWS Bedrock.

## Key Components

### Structs
- `anthropicOptions`: A struct to hold specific options for the Anthropic client, such as whether to use AWS Bedrock, disable caching, or a custom function to decide when the model should "think" (use reasoning tags).
- `anthropicClient`: The concrete implementation of the `Provider` interface for Anthropic.
    - `providerOptions (providerClientOptions)`: Common provider options (API key, model, system message, max tokens) inherited from the generic provider setup.
    - `options (anthropicOptions)`: Anthropic-specific options.
    - `client (anthropic.Client)`: The actual client from the `anthropics/anthropic-sdk-go` library.

### Types
- `AnthropicOption`: A functional option type for configuring `anthropicOptions`.
- `AnthropicClient`: An alias for `ProviderClient` (which is an alias for the `Provider` interface), used for clarity.

### Core Functions
- `newAnthropicClient(opts providerClientOptions) AnthropicClient`: Constructor for `anthropicClient`. It initializes the `anthropic.Client` from the SDK, applying API keys and Bedrock options if configured.
- `(a *anthropicClient) convertMessages(messages []message.Message) []anthropic.MessageParam`: Converts OpenCode's internal `message.Message` format to the `anthropic.MessageParam` format required by the Anthropic SDK. It handles user, assistant, and tool messages, including binary content (images) and tool calls/results. It also sets cache control flags for recent messages if caching is not disabled.
- `(a *anthropicClient) convertTools(tools []tools.BaseTool) []anthropic.ToolUnionParam`: Converts OpenCode's `tools.BaseTool` definition into the `anthropic.ToolUnionParam` format. It also sets cache control for tools.
- `(a *anthropicClient) finishReason(reason string) message.FinishReason`: Maps Anthropic's API finish reasons (e.g., "end_turn", "max_tokens") to OpenCode's internal `message.FinishReason` enum.
- `(a *anthropicClient) preparedMessages(messages []anthropic.MessageParam, tools []anthropic.ToolUnionParam) anthropic.MessageNewParams`: Constructs the final `anthropic.MessageNewParams` struct for an API call. This includes the model, max tokens, temperature (adjusted if "thinking" is enabled), converted messages, tools, and system prompt. It also implements logic to enable Anthropic's "thinking" feature based on the `shouldThink` option and message content.
- `(a *anthropicClient) send(ctx context.Context, messages []message.Message, tools []tools.BaseTool) (*ProviderResponse, error)`: Implements the non-streaming message sending. It prepares messages, makes the API call using `a.client.Messages.New()`, and handles retries for rate limits (429, 529 status codes) with exponential backoff. It then converts the Anthropic response into a `ProviderResponse`. (This function seems to be unused in favor of `stream` based on the `Provider` interface, but the logic is present).
- `(a *anthropicClient) stream(ctx context.Context, messages []message.Message, tools []tools.BaseTool) <-chan ProviderEvent`: Implements the streaming message sending.
    - Prepares messages similarly to `send`.
    - Calls `a.client.Messages.NewStreaming()` to get a stream.
    - Iterates through events from the stream (`anthropicStream.Next()`).
    - For each event type from Anthropic (e.g., `ContentBlockStartEvent`, `ContentBlockDeltaEvent`, `MessageStopEvent`), it converts it into an OpenCode `ProviderEvent` (e.g., `EventContentStart`, `EventContentDelta`, `EventToolUseStart`, `EventComplete`) and sends it on the returned channel.
    - It accumulates the full message from deltas.
    - Handles retries for rate limits during streaming as well.
- `(a *anthropicClient) shouldRetry(attempts int, err error) (bool, int64, error)`: Determines if an API call should be retried based on the error (specifically 429 or 529 status codes) and the number of attempts. Calculates backoff time.
- `(a *anthropicClient) toolCalls(msg anthropic.Message) []message.ToolCall`: Extracts tool call information from an Anthropic response message and converts it to OpenCode's `message.ToolCall` format.
- `(a *anthropicClient) usage(msg anthropic.Message) TokenUsage`: Extracts token usage information from an Anthropic response and converts it to OpenCode's `TokenUsage` format, including cache-related token counts.

### Functional Options
- `WithAnthropicBedrock(useBedrock bool) AnthropicOption`: Option to configure the client to use AWS Bedrock for accessing Anthropic models.
- `WithAnthropicDisableCache() AnthropicOption`: Option to disable Anthropic's caching features.
- `DefaultShouldThinkFn(s string) bool`: Default logic for deciding if the "thinking" feature should be enabled (if user message contains "think").
- `WithAnthropicShouldThinkFn(fn func(string) bool) AnthropicOption`: Option to provide a custom function to determine if the model should use its "thinking" mode.

## Important Variables/Constants
This file does not define exported package-level constants or variables beyond the option types and constructor.

## Usage Examples

This client is not instantiated directly by typical user code but by the generic `provider.NewProvider` function when `models.ProviderAnthropic` is specified.

```go
// Conceptual instantiation via generic provider factory:
// import "github.com/opencode-ai/opencode/internal/llm/provider"
// import "github.com/opencode-ai/opencode/internal/llm/models"

// opts := []provider.ProviderClientOption{
//     provider.WithAPIKey("sk-..."),
//     provider.WithModel(models.SupportedModels[models.Claude35Sonnet]),
//     provider.WithSystemMessage("You are a helpful assistant."),
//     provider.WithMaxTokens(1024),
//     provider.WithAnthropicOptions( // Anthropic-specific options
//         provider.WithAnthropicBedrock(false),
//         provider.WithAnthropicShouldThinkFn(provider.DefaultShouldThinkFn),
//     ),
// }
// anthropicProvider, err := provider.NewProvider(models.ProviderAnthropic, opts...)

// Streaming a response:
// eventChannel := anthropicProvider.StreamResponse(context.Background(), messages, tools)
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
    - `github.com/opencode-ai/opencode/internal/message`: For `message.Message`, `message.ToolCall`, `message.FinishReason` etc.
    - Depends on the `Provider` interface and `ProviderEvent`, `ProviderResponse`, `TokenUsage`, `providerClientOptions` types from `provider.go` in the same package.
- **External Libraries:**
    - `github.com/anthropics/anthropic-sdk-go`: The official Anthropic Go SDK for all API interactions. This includes sub-packages for `bedrock` and `option`.
- **Interactions:**
    - Acts as a bridge between OpenCode's generic LLM interaction patterns and the specific requirements of the Anthropic API.
    - Translates message formats, tool definitions, and event types.
    - Manages API call complexities like streaming, error handling, and retries.
    - Supports optional configuration for using Anthropic models via AWS Bedrock.
    - Implements Anthropic-specific features like "thinking" prompts and cache control.

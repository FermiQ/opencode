# bedrock.go (in internal/llm/provider)

## Overview

The `bedrock.go` file, part of the `internal/llm/provider` package, implements the `Provider` interface for AWS Bedrock. AWS Bedrock is a service that provides access to foundation models from various AI companies. This Bedrock provider in OpenCode acts as a delegator or wrapper. Based on the specific model being used (e.g., an Anthropic Claude model hosted on Bedrock), it instantiates and uses the corresponding underlying provider client (like the `anthropicClient`) but configured for Bedrock.

## Key Components

### Structs
- `bedrockOptions`: A struct intended to hold Bedrock-specific options. Currently, it's empty, but it's structured to allow future Bedrock-specific configurations.
- `bedrockClient`: The concrete implementation of the `Provider` interface for AWS Bedrock.
    - `providerOptions (providerClientOptions)`: Common provider options (API key, model, system message, max tokens) inherited from the generic provider setup.
    - `options (bedrockOptions)`: Bedrock-specific options.
    - `childProvider (ProviderClient)`: This is the crucial field. It holds an instance of another provider client (e.g., `*anthropicClient`) that will actually handle the API communication with the model on Bedrock.

### Types
- `BedrockOption`: A functional option type for configuring `bedrockOptions` (currently unused as `bedrockOptions` is empty).
- `BedrockClient`: An alias for `ProviderClient` (which is an alias for the `Provider` interface), for clarity when referring to a Bedrock provider instance.

### Functions
- `newBedrockClient(opts providerClientOptions) BedrockClient`: Constructor for `bedrockClient`.
    1.  Initializes `bedrockOptions`.
    2.  Determines the AWS region from environment variables (`AWS_REGION` or `AWS_DEFAULT_REGION`), defaulting to "us-east-1" if neither is set.
    3.  **Region Prefixing (Potentially Problematic)**: It prefixes the `opts.model.APIModel` with the first two characters of the AWS region (e.g., "us.anthropic.claude-v2"). This behavior might be specific to an internal convention or an older version of an SDK, as Bedrock model IDs are typically structured like `anthropic.claude-3-sonnet-20240229-v1:0` and don't usually include a region prefix in the model ID itself for API calls. The region is part of the Bedrock client/endpoint configuration.
    4.  **Child Provider Delegation**:
        - If the (potentially region-prefixed) `opts.model.APIModel` string contains "anthropic", it creates a new `anthropicClient`. Importantly, it passes `WithAnthropicBedrock(true)` and `WithAnthropicDisableCache()` as options to this `anthropicClient`, configuring it to interact with Anthropic models via Bedrock and disable Anthropic's native caching (as Bedrock might have its own or different caching behavior).
        - The `bedrockClient` then stores this configured `anthropicClient` in its `childProvider` field.
    5.  If the model is not recognized (e.g., not an Anthropic model currently supported by this logic), `childProvider` is set to `nil`.
- `(b *bedrockClient) send(ctx context.Context, messages []message.Message, tools []tools.BaseTool) (*ProviderResponse, error)`: Implements the non-streaming send method.
    - If `b.childProvider` is `nil` (meaning an unsupported model was configured for Bedrock), it returns an error.
    - Otherwise, it delegates the call directly to `b.childProvider.send()`.
- `(b *bedrockClient) stream(ctx context.Context, messages []message.Message, tools []tools.BaseTool) <-chan ProviderEvent`: Implements the streaming method.
    - If `b.childProvider` is `nil`, it returns a channel that immediately sends an error event and closes.
    - Otherwise, it delegates the call directly to `b.childProvider.stream()`.

## Important Variables/Constants
This file does not define exported package-level constants or variables beyond the type alias, constructor, and option type.

## Usage Examples

This client is instantiated by the generic `provider.NewProvider` function when `models.ProviderBedrock` is specified.

```go
// Conceptual instantiation via generic provider factory:
// import "github.com/opencode-ai/opencode/internal/llm/provider"
// import "github.com/opencode-ai/opencode/internal/llm/models"

// // Ensure AWS environment variables for authentication and region are set, e.g.:
// // export AWS_ACCESS_KEY_ID="your_access_key"
// // export AWS_SECRET_ACCESS_KEY="your_secret_key"
// // export AWS_REGION="us-west-2"

// bedrockModelID := models.ModelID("bedrock.claude-3.7-sonnet") // Example
// bedrockModelMeta := models.SupportedModels[bedrockModelID]

// opts := []provider.ProviderClientOption{
//     // APIKey for Bedrock is usually handled by AWS SDK credentials chain, not passed directly.
//     provider.WithModel(bedrockModelMeta),
//     provider.WithSystemMessage("You are a helpful assistant via Bedrock."),
//     provider.WithMaxTokens(2048),
// }
// bedrockProvider, err := provider.NewProvider(models.ProviderBedrock, opts...)

// // Interaction with bedrockProvider then delegates to the child provider (e.g., anthropicClient for Claude):
// // eventChannel := bedrockProvider.StreamResponse(context.Background(), messages, tools)
// // for event := range eventChannel {
// //     // process event
// // }
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/llm/models`: For provider type constants like `models.ProviderBedrock` and model metadata.
    - `github.com/opencode-ai/opencode/internal/llm/tools`: For `tools.BaseTool` interface.
    - `github.com/opencode-ai/opencode/internal/message`: For `message.Message` type.
    - Depends on `Provider`, `ProviderClient`, `ProviderEvent`, `ProviderResponse`, `providerClientOptions` types from `provider.go` in the same package.
    - Depends on other provider implementations within the same package, specifically `newAnthropicClient` and its options like `WithAnthropicBedrock`.
- **External Libraries:**
    - `os`: For reading AWS region environment variables.
    - `errors`, `fmt`, `strings`, `context`: Standard Go libraries.
- **Interactions:**
    - Acts as a facade or router for models available on AWS Bedrock.
    - It determines the actual underlying model provider (e.g., Anthropic) based on the model ID.
    - It then instantiates and configures the client for that underlying provider with Bedrock-specific settings.
    - All core LLM interaction logic (message formatting, API calls, streaming) is delegated to this `childProvider`.
    - The AWS SDK's default credential chain is implicitly relied upon for authentication with Bedrock when the child provider (e.g., `anthropicClient` with `WithAnthropicBedrock(true)`) makes its calls.
    - The region prefixing logic for `APIModel` is a notable detail that might need careful handling or updates if Bedrock model ID conventions change.

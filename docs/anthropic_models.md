# anthropic.go

## Overview

The `anthropic.go` file, located in the `internal/llm/models` package, defines constants and metadata for various AI models provided by Anthropic. This information is used throughout the OpenCode application to identify, configure, and calculate costs associated with using these Anthropic models (e.g., Claude series).

## Key Components

### Constants
- `ProviderAnthropic (ModelProvider)`: A constant of type `ModelProvider` (defined in `models.go`) set to "anthropic". This identifies Anthropic as the provider for the models defined in this file.
- Model ID Constants: A series of constants of type `ModelID` (defined in `models.go`) representing specific Anthropic models. Examples:
    - `Claude35Sonnet`
    - `Claude3Haiku`
    - `Claude37Sonnet`
    - `Claude35Haiku`
    - `Claude3Opus`
    - `Claude4Sonnet`
    - `Claude4Opus`

### Variables
- `AnthropicModels (map[ModelID]Model)`: A package-level map where:
    - Keys are `ModelID` constants (e.g., `Claude35Sonnet`).
    - Values are `Model` structs (defined in `models.go`) containing detailed metadata for each Anthropic model.
    - The metadata for each `Model` includes:
        - `ID (ModelID)`: The unique OpenCode identifier for the model.
        - `Name (string)`: A human-readable name for the model (e.g., "Claude 3.5 Sonnet").
        - `Provider (ModelProvider)`: Set to `ProviderAnthropic`.
        - `APIModel (string)`: The specific model name to be used when making API calls to Anthropic (e.g., "claude-3-5-sonnet-latest").
        - `CostPer1MIn (float64)`: Cost in USD per 1 million input tokens.
        - `CostPer1MInCached (float64)`: Cost in USD per 1 million input tokens when using caching (if applicable).
        - `CostPer1MOutCached (float64)`: Cost in USD per 1 million output tokens when using caching (if applicable).
        - `CostPer1MOut (float64)`: Cost in USD per 1 million output tokens.
        - `ContextWindow (int64)`: The maximum number of tokens the model can consider (input + output).
        - `DefaultMaxTokens (int64)`: A default maximum number of tokens to request for generation from this model.
        - `CanReason (bool)`: Indicates if the model has specific features or prompts for "reasoning" or "thinking" steps (e.g., for Anthropic's XML-based thinking tags).
        - `SupportsAttachments (bool)`: Indicates if the model supports file attachments or binary content directly in prompts.

## Important Variables/Constants
- `ProviderAnthropic`: Identifies the provider.
- `AnthropicModels`: The map containing all metadata for supported Anthropic models. This map is likely merged into a global map of all supported models from all providers (see `models.go`).

## Usage Examples

This file primarily provides data. Other parts of the application, particularly the configuration (`internal/config/config.go`) and LLM provider clients (`internal/llm/provider/anthropic.go`), would use this information.

Accessing model metadata:
```go
// import "github.com/opencode-ai/opencode/internal/llm/models"

// ...
// modelID := models.Claude35Sonnet
// modelMeta, ok := models.SupportedModels[modelID] // SupportedModels is likely a global map
// if ok && modelMeta.Provider == models.ProviderAnthropic {
//     fmt.Printf("Model Name: %s\n", modelMeta.Name)
//     fmt.Printf("API Model String: %s\n", modelMeta.APIModel)
//     fmt.Printf("Context Window: %d tokens\n", modelMeta.ContextWindow)
// }
```

The cost information is used by the agent (`internal/llm/agent/agent.go`) to track session costs. The `APIModel` string is used by the Anthropic provider client when making API requests.

## Dependencies and Interactions

- **Internal Dependencies:**
    - Relies on `ModelProvider`, `ModelID`, and `Model` types defined in `models.go` within the same package.
- **External Libraries:** None.
- **Interactions:**
    - Provides a centralized definition of supported Anthropic models and their properties.
    - This data is consumed by:
        - Configuration loading to validate model choices and set defaults.
        - The Anthropic LLM provider client (`internal/llm/provider/anthropic.go`) to know which API model string to use.
        - The agent (`internal/llm/agent/agent.go`) for cost calculation and understanding model capabilities (like `SupportsAttachments`, `CanReason`).
    - The `AnthropicModels` map is expected to be registered or merged into a global map like `SupportedModels` in `models.go` to make these definitions accessible application-wide.

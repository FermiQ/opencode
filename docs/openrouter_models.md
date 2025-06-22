# openrouter.go

## Overview

The `openrouter.go` file, part of the `internal/llm/models` package, specifies constants and metadata for a wide array of AI models that are accessible through the OpenRouter API. OpenRouter serves as an aggregator, providing a unified interface to models from various original providers like OpenAI, Anthropic, and Google. This file allows OpenCode to treat OpenRouter as a distinct provider while still leveraging the metadata (costs, context windows) of the underlying original models.

## Key Components

### Constants
- `ProviderOpenRouter (ModelProvider)`: A constant of type `ModelProvider` (defined in `models.go`) set to "openrouter". This identifies OpenRouter as the provider for the models listed herein.
- Model ID Constants: Numerous constants of type `ModelID` (defined in `models.go`) represent specific models available via OpenRouter. These IDs are typically namespaced with "openrouter." followed by a name that often reflects the original model (e.g., `OpenRouterGPT4o`, `OpenRouterClaude35Sonnet`). Examples include:
    - `OpenRouterGPT41`, `OpenRouterGPT4oMini`
    - `OpenRouterGemini25Flash`, `OpenRouterGemini25`
    - `OpenRouterClaude35Sonnet`, `OpenRouterClaude3Haiku`
    - `OpenRouterDeepSeekR1Free`

### Variables
- `OpenRouterModels (map[ModelID]Model)`: A package-level map where:
    - Keys are `ModelID` constants (e.g., `OpenRouterGPT4o`).
    - Values are `Model` structs (defined in `models.go`) containing metadata for each OpenRouter-accessible model.
    - A significant characteristic of this map is that for many models, the metadata fields (like `CostPer1MIn`, `ContextWindow`, `DefaultMaxTokens`, `CanReason`) are populated by referencing the values from the corresponding original model's definition in other files (e.g., `OpenAIModels[GPT4o]`, `AnthropicModels[Claude35Sonnet]`, `GeminiModels[Gemini25Flash]`).
    - Each `Model` struct includes:
        - `ID (ModelID)`: The unique OpenCode identifier for the OpenRouter model (e.g., `OpenRouterGPT4o`).
        - `Name (string)`: A human-readable name, often indicating "OpenRouter – [Original Model Name]".
        - `Provider (ModelProvider)`: Set to `ProviderOpenRouter`.
        - `APIModel (string)`: The specific model identifier string required by the OpenRouter API. This string often includes the original provider and model name (e.g., "openai/gpt-4o", "anthropic/claude-3.5-sonnet", "google/gemini-2.5-flash-preview:thinking").
        - Cost fields, `ContextWindow`, `DefaultMaxTokens`, `CanReason`, `SupportsAttachments`: These are typically derived from the original model's metadata. For some models like `OpenRouterDeepSeekR1Free`, costs might be explicitly set to 0 if it's a free tier offered by OpenRouter.

## Important Variables/Constants
- `ProviderOpenRouter`: Identifies OpenRouter as the LLM provider.
- `OpenRouterModels`: The map containing metadata for all supported OpenRouter-accessible models. This is intended to be merged into the global `SupportedModels` map in `models.go`.

## Usage Examples

This file primarily provides data definitions. This data is crucial for the configuration system (`internal/config/config.go`) and the OpenRouter LLM provider client (`internal/llm/provider/openrouter.go`).

Accessing OpenRouter model metadata:
```go
// import "github.com/opencode-ai/opencode/internal/llm/models"

// ...
// modelID := models.OpenRouterClaude3Opus
// modelMeta, ok := models.SupportedModels[modelID] // SupportedModels is the global map
// if ok && modelMeta.Provider == models.ProviderOpenRouter {
//     fmt.Printf("Model Name: %s\n", modelMeta.Name)
//     fmt.Printf("OpenRouter API Model String: %s\n", modelMeta.APIModel)
//     // Costs and other properties are often inherited from the original model
//     fmt.Printf("Cost per 1M Input Tokens: $%.2f\n", modelMeta.CostPer1MIn)
// }
```

The `APIModel` string is essential for the OpenRouter provider client when making API requests. Cost information, usually inherited from the original provider's rates, is used by the agent for session cost tracking.

## Dependencies and Interactions

- **Internal Dependencies:**
    - Relies on `ModelProvider`, `ModelID`, and `Model` types defined in `models.go`.
    - Heavily depends on the model metadata maps from other provider files (`OpenAIModels`, `AnthropicModels`, `GeminiModels`, etc.) to populate the characteristics of the models it exposes.
- **External Libraries:** None.
- **Interactions:**
    - Provides a centralized definition of models accessible via OpenRouter, effectively acting as a pass-through or wrapper for models from other original providers.
    - This data is consumed by:
        - The configuration system for model validation and default settings when OpenRouter is the active provider.
        - The OpenRouter LLM provider client to use the correct `APIModel` string for API calls to OpenRouter.
        - The AI agent for cost calculation and understanding model capabilities, which are generally those of the underlying original model.
    - The `OpenRouterModels` map is merged into the global `SupportedModels` map (in `models.go`) to make these definitions available application-wide.

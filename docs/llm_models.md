# models.go

## Overview

The `models.go` file in the `internal/llm/models` package serves as the central point for defining and aggregating metadata about all supported Large Language Models (LLMs) from various providers. It defines core types like `ModelID` and `ModelProvider`, the `Model` struct that holds metadata, and initializes a global map `SupportedModels` by merging model definitions from other files in the same package (e.g., `anthropic.go`, `openai.go`, `local.go`).

## Key Components

### Types
- `ModelID (string)`: A type alias for string, used to uniquely identify a specific LLM model within OpenCode (e.g., "claude-3.5-sonnet", "openai.gpt-4o", "local.my-custom-model").
- `ModelProvider (string)`: A type alias for string, used to identify the provider of an LLM (e.g., "anthropic", "openai", "local", "azure").
- `Model (struct)`: The primary struct for holding metadata about an LLM.
    - `ID (ModelID)`: Unique OpenCode identifier.
    - `Name (string)`: Human-readable name.
    - `Provider (ModelProvider)`: The provider of the model.
    - `APIModel (string)`: The actual model name/identifier to be used in API calls to the provider.
    - `CostPer1MIn (float64)`: Cost per 1 million input tokens (USD).
    - `CostPer1MOut (float64)`: Cost per 1 million output tokens (USD).
    - `CostPer1MInCached (float64)`: Cost for cached input tokens.
    - `CostPer1MOutCached (float64)`: Cost for cached output tokens.
    - `ContextWindow (int64)`: Maximum token limit (input + output).
    - `DefaultMaxTokens (int64)`: Default maximum tokens to request for generation.
    - `CanReason (bool)`: Flag indicating if the model supports specific "reasoning" or "thinking" patterns (e.g., XML tags for step-by-step thinking).
    - `SupportsAttachments (bool)`: Flag indicating if the model API supports sending file attachments/binary data.

### Constants
- Various `ModelID` constants are defined here for models that might not fit neatly into a provider-specific file or for cross-provider aliases (e.g., `BedrockClaude37Sonnet`).
- Provider constants like `ProviderBedrock` and `ProviderMock` are defined.

### Variables
- `ProviderPopularity (map[ModelProvider]int)`: A map that assigns a popularity or preference order to different `ModelProvider`s. Lower numbers usually indicate higher preference (e.g., used for selecting a default model if multiple providers are configured).
- `SupportedModels (map[ModelID]Model)`: A global map that aggregates all model definitions from all providers. It's initialized with some base models (like Bedrock's Claude) and then populated further in the `init()` function.

### Initialization (`init` function)
- This function is crucial. It runs when the package is loaded and populates the `SupportedModels` map.
- It uses `maps.Copy` to merge model definitions from various provider-specific maps (e.g., `AnthropicModels` from `anthropic.go`, `OpenAIModels` from `openai.go`, `GeminiModels`, `GroqModels`, `AzureModels`, `OpenRouterModels`, `XAIModels`, `VertexAIGeminiModels`) into the `SupportedModels` map.
- The `local.go` file also has an `init` function that can further populate `SupportedModels` with dynamically discovered local models.

## Important Variables/Constants
- `SupportedModels`: This is the most critical exported variable, providing a single point of access to the metadata of all LLMs known to the OpenCode application.
- `ProviderPopularity`: Influences default model selection logic in the configuration.
- `ModelID`, `ModelProvider`, `Model`: Core types for working with LLMs.

## Usage Examples

Accessing metadata for any supported model:
```go
// import "github.com/opencode-ai/opencode/internal/llm/models"

modelIdentifier := models.ModelID("openai.gpt-4o") // Or any other ModelID
modelInfo, isSupported := models.SupportedModels[modelIdentifier]

if isSupported {
    fmt.Printf("Model: %s, Provider: %s, Context Window: %d\n",
        modelInfo.Name, modelInfo.Provider, modelInfo.ContextWindow)
} else {
    fmt.Printf("Model %s is not supported.\n", modelIdentifier)
}
```

This `SupportedModels` map is extensively used by:
- The configuration system (`internal/config/config.go`) to validate user-selected models, set default models based on available providers and popularity, and retrieve model properties.
- LLM provider clients (`internal/llm/provider/*`) to get the correct `APIModel` string and other capabilities.
- The AI agent (`internal/llm/agent/agent.go`) to calculate costs, understand context limits, and determine if features like attachments or reasoning are supported.

## Dependencies and Interactions

- **Internal Dependencies:**
    - Relies on the provider-specific model maps defined in other `*.go` files within the `internal/llm/models` package (e.g., `AnthropicModels`, `OpenAIModels`, etc.). These files must define their maps for the `init()` function here to aggregate them.
- **External Libraries:**
    - `maps` (from Go 1.21+ standard library, or `golang.org/x/exp/maps` for older Go versions, though the import here implies standard library usage): Used for copying maps in the `init` function.
- **Interactions:**
    - Acts as the central registry for all LLM model metadata.
    - The `init()` function orchestrates the collection of model data from all other files in its package.
    - Provides the rest of the application with a unified way to access information about any supported LLM, regardless of its provider.
    - The order of `maps.Copy` calls in `init()` could matter if there were overlapping `ModelID` keys between different provider files, though `ModelID`s are typically namespaced (e.g., "openai.gpt-4o", "anthropic.claude-3-opus") to prevent such collisions.

# xai.go

## Overview

The `xai.go` file, found in the `internal/llm/models` package, is dedicated to defining constants and metadata for AI models provided by XAI (likely referring to xAI). The primary models listed are variants of "Grok". This file enables the OpenCode application to identify, configure, and estimate costs associated with using these XAI models.

## Key Components

### Constants
- `ProviderXAI (ModelProvider)`: A constant of type `ModelProvider` (defined in `models.go`) set to "xai". This identifies XAI as the provider for the models defined in this file.
- Model ID Constants: A series of constants of type `ModelID` (defined in `models.go`) that serve as unique identifiers for specific XAI Grok models within the OpenCode system. Examples:
    - `XAIGrok3Beta`
    - `XAIGrok3MiniBeta`
    - `XAIGrok3FastBeta`
    - `XAiGrok3MiniFastBeta` (Note: slight variation in casing, `XAi` vs `XAI`)

### Variables
- `XAIModels (map[ModelID]Model)`: A package-level map where:
    - Keys are the `ModelID` constants (e.g., `XAIGrok3Beta`).
    - Values are `Model` structs (defined in `models.go`), each populated with specific metadata for an XAI Grok model.
    - The metadata within each `Model` struct includes:
        - `ID (ModelID)`: The OpenCode internal unique ID for the model.
        - `Name (string)`: A user-friendly, human-readable name (e.g., "Grok3 Beta").
        - `Provider (ModelProvider)`: Consistently set to `ProviderXAI`.
        - `APIModel (string)`: The precise model identifier string required when making API calls to XAI (e.g., "grok-3-beta").
        - `CostPer1MIn (float64)`: The cost in USD for processing 1 million input tokens.
        - `CostPer1MOut (float64)`: The cost in USD for generating 1 million output tokens.
        - `CostPer1MInCached (float64)`: Cost for cached input tokens (currently set to 0 for all XAI models, suggesting this pricing tier might not apply or isn't used).
        - `CostPer1MOutCached (float64)`: Cost for cached output tokens (similarly, 0).
        - `ContextWindow (int64)`: The maximum number of tokens (input + output) that the model can handle. All listed Grok models have a 131,072 token context window.
        - `DefaultMaxTokens (int64)`: A default for the maximum number of tokens to request in a generation (set to 20,000 for all listed models).
        - `CanReason (bool)`: (Not explicitly set, defaults to `false`).
        - `SupportsAttachments (bool)`: (Not explicitly set, defaults to `false`).

## Important Variables/Constants
- `ProviderXAI`: Crucial for identifying XAI as the LLM provider.
- `XAIModels`: This map is the core of the file, containing all the metadata for the supported XAI models. It's intended to be merged into the global `SupportedModels` map (in `models.go`).

## Usage Examples

This file primarily serves as a data source. Its contents are utilized by other parts of the OpenCode application, such as the configuration system (`internal/config/config.go`) and any XAI-specific LLM provider client that might be implemented.

Accessing XAI model metadata:
```go
// import "github.com/opencode-ai/opencode/internal/llm/models"

// ...
// modelID := models.XAIGrok3MiniBeta
// modelMeta, isSupported := models.SupportedModels[modelID] // SupportedModels is the global map
//
// if isSupported && modelMeta.Provider == models.ProviderXAI {
//     fmt.Printf("Model Name: %s\n", modelMeta.Name)
//     fmt.Printf("XAI API Model String: %s\n", modelMeta.APIModel)
//     fmt.Printf("Cost per 1M Input Tokens: $%.2f\n", modelMeta.CostPer1MIn)
//     fmt.Printf("Context Window: %d tokens\n", modelMeta.ContextWindow)
// }
```

The `APIModel` string is essential for an XAI provider client when formulating API requests. Cost details help the agent (`internal/llm/agent/agent.go`) track session expenses.

## Dependencies and Interactions

- **Internal Dependencies:**
    - Depends on the `ModelProvider`, `ModelID`, and `Model` types defined in `models.go` (within the same package).
- **External Libraries:** None.
- **Interactions:**
    - Provides a structured and centralized definition for supported XAI models and their properties.
    - This data is consumed by:
        - The application's configuration system for model validation and setting defaults if XAI is chosen.
        - An XAI LLM provider client (if one exists in `internal/llm/provider/`) to use the correct `APIModel` identifier.
        - The core AI agent for cost calculation and understanding model capabilities.
    - The `XAIModels` map is merged into the global `SupportedModels` map (defined in `models.go`) during package initialization, making these XAI model definitions accessible throughout the application.

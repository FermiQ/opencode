# gemini.go

## Overview

The `gemini.go` file, within the `internal/llm/models` package, specifies constants and metadata for AI models from Google's Gemini family. This information is crucial for the OpenCode application to correctly identify, configure, and estimate costs when interacting with these Google Gemini models.

## Key Components

### Constants
- `ProviderGemini (ModelProvider)`: A constant of type `ModelProvider` (defined in `models.go`) set to "gemini". This identifies Google Gemini as the provider for the models listed in this file.
- Model ID Constants: A series of constants of type `ModelID` (defined in `models.go`) that uniquely identify specific Gemini models within OpenCode. Examples:
    - `Gemini25Flash`
    - `Gemini25` (likely referring to Gemini 2.5 Pro)
    - `Gemini20Flash`
    - `Gemini20FlashLite`

### Variables
- `GeminiModels (map[ModelID]Model)`: A package-level map where:
    - Keys are `ModelID` constants (e.g., `Gemini25Flash`).
    - Values are `Model` structs (defined in `models.go`) containing detailed metadata for each Gemini model.
    - The metadata for each `Model` includes:
        - `ID (ModelID)`: The unique OpenCode identifier for the model.
        - `Name (string)`: A human-readable name (e.g., "Gemini 2.5 Flash").
        - `Provider (ModelProvider)`: Set to `ProviderGemini`.
        - `APIModel (string)`: The specific model identifier to be used when making API calls to the Google Gemini API (e.g., "gemini-2.5-flash-preview-04-17"). Note that these can be versioned preview strings.
        - `CostPer1MIn (float64)`: Cost in USD per 1 million input tokens.
        - `CostPer1MInCached (float64)`: Cost for cached input tokens (currently set to 0 for Gemini models, suggesting caching might not be a distinct pricing factor or is handled differently).
        - `CostPer1MOutCached (float64)`: Cost for cached output tokens (similarly, 0).
        - `CostPer1MOut (float64)`: Cost in USD per 1 million output tokens.
        - `ContextWindow (int64)`: The maximum number of tokens the model can process.
        - `DefaultMaxTokens (int64)`: A default maximum number of tokens to request for generation.
        - `SupportsAttachments (bool)`: Indicates if the model supports file attachments or binary content in prompts.
        - `CanReason (bool)`: (Not explicitly set for Gemini models in the provided file, meaning it defaults to `false` or is not a primary advertised feature in this context).

## Important Variables/Constants
- `ProviderGemini`: Identifies Google Gemini as the LLM provider.
- `GeminiModels`: The map containing all metadata for supported Gemini models. This map is intended to be merged into the global `SupportedModels` map in `models.go`.

## Usage Examples

This file primarily provides data definitions. Other parts of the application, such as the configuration system (`internal/config/config.go`) and the Gemini LLM provider client (`internal/llm/provider/gemini.go`), would consume this information.

Accessing Gemini model metadata:
```go
// import "github.com/opencode-ai/opencode/internal/llm/models"

// ...
// modelID := models.Gemini25
// modelMeta, ok := models.SupportedModels[modelID] // SupportedModels is the global map
// if ok && modelMeta.Provider == models.ProviderGemini {
//     fmt.Printf("Model Name: %s\n", modelMeta.Name)
//     fmt.Printf("Gemini API Model String: %s\n", modelMeta.APIModel)
//     fmt.Printf("Cost per 1M Input Tokens: $%.2f\n", modelMeta.CostPer1MIn)
// }
```

The `APIModel` string is used by the Gemini provider client when making API requests. Cost information is used by the agent for tracking session expenses.

## Dependencies and Interactions

- **Internal Dependencies:**
    - Relies on `ModelProvider`, `ModelID`, and `Model` types defined in `models.go` within the same package.
- **External Libraries:** None.
- **Interactions:**
    - Provides a centralized definition of supported Google Gemini models and their properties.
    - This data is consumed by:
        - The configuration system (`internal/config/config.go`) for validating model choices and setting defaults when Gemini is the selected provider.
        - The Gemini LLM provider client (`internal/llm/provider/gemini.go`) to use the correct `APIModel` string for API requests.
        - The agent (`internal/llm/agent/agent.go`) for cost calculation and understanding model capabilities like `SupportsAttachments`.
    - The `GeminiModels` map is merged into the global `SupportedModels` map (in `models.go`) to make these definitions accessible throughout the application.

# groq.go

## Overview

The `groq.go` file, part of the `internal/llm/models` package, lists constants and metadata for various AI models accessible through the Groq API. Groq is known for providing fast inference speeds for open-source LLMs. This file helps OpenCode identify, configure, and estimate costs for these Groq-served models.

## Key Components

### Constants
- `ProviderGROQ (ModelProvider)`: A constant of type `ModelProvider` (defined in `models.go`) set to "groq". This identifies Groq as the provider for the models in this file.
- Model ID Constants: Constants of type `ModelID` (defined in `models.go`) that uniquely identify specific models available via Groq within OpenCode. Examples:
    - `QWENQwq`
    - `Llama4Scout`
    - `Llama4Maverick`
    - `Llama3_3_70BVersatile`
    - `DeepseekR1DistillLlama70b`
    Some of these IDs, like `meta-llama/llama-4-scout-17b-16e-instruct`, directly reflect the model names used by Groq, which can include organization prefixes.

### Variables
- `GroqModels (map[ModelID]Model)`: A package-level map where:
    - Keys are `ModelID` constants (e.g., `QWENQwq`).
    - Values are `Model` structs (defined in `models.go`) containing metadata for each Groq-served model.
    - The metadata for each `Model` includes:
        - `ID (ModelID)`: The unique OpenCode identifier.
        - `Name (string)`: A human-readable name (e.g., "Qwen Qwq", "Llama4Scout").
        - `Provider (ModelProvider)`: Set to `ProviderGROQ`.
        - `APIModel (string)`: The specific model identifier to be used when making API calls to Groq (e.g., "qwen-qwq-32b", "meta-llama/llama-4-scout-17b-16e-instruct").
        - `CostPer1MIn (float64)`: Cost in USD per 1 million input tokens.
        - `CostPer1MInCached (float64)`, `CostPer1MOutCached (float64)`: Costs for cached tokens (often 0 or specific values if applicable).
        - `CostPer1MOut (float64)`: Cost in USD per 1 million output tokens.
        - `ContextWindow (int64)`: The maximum number of tokens the model can process.
        - `DefaultMaxTokens (int64)`: A default maximum number of tokens to request for generation.
        - `CanReason (bool)`: Indicates if the model supports specific "reasoning" or "thinking" step functionalities. (Note: For `QWENQwq`, it's explicitly set to `false` with a comment about Groq API behavior).
        - `SupportsAttachments (bool)`: Indicates if the model supports file attachments.

## Important Variables/Constants
- `ProviderGROQ`: Identifies Groq as the LLM provider.
- `GroqModels`: The map containing metadata for all supported Groq-served models. This map is intended to be merged into the global `SupportedModels` map in `models.go`.

## Usage Examples

This file primarily provides data definitions. Other application components, like the configuration system (`internal/config/config.go`) and the Groq LLM provider client (`internal/llm/provider/groq.go`), would use this data.

Accessing Groq model metadata:
```go
// import "github.com/opencode-ai/opencode/internal/llm/models"

// ...
// modelID := models.Llama4Scout
// modelMeta, ok := models.SupportedModels[modelID] // SupportedModels is the global map
// if ok && modelMeta.Provider == models.ProviderGROQ {
//     fmt.Printf("Model Name: %s\n", modelMeta.Name)
//     fmt.Printf("Groq API Model String: %s\n", modelMeta.APIModel)
//     fmt.Printf("Context Window: %d tokens\n", modelMeta.ContextWindow)
// }
```

The `APIModel` string is used by the Groq provider client for API requests. Cost information helps the agent track session expenses.

## Dependencies and Interactions

- **Internal Dependencies:**
    - Relies on `ModelProvider`, `ModelID`, and `Model` types defined in `models.go` within the same package.
- **External Libraries:** None.
- **Interactions:**
    - Provides a centralized definition of supported Groq-served models and their properties.
    - This data is used by:
        - The configuration system (`internal/config/config.go`) for model validation and default settings when Groq is the chosen provider.
        - The Groq LLM provider client (`internal/llm/provider/groq.go`) to use the correct `APIModel` string in API calls.
        - The agent (`internal/llm/agent/agent.go`) for cost calculation and understanding model capabilities.
    - The `GroqModels` map is merged into the global `SupportedModels` map (in `models.go`) for application-wide access.

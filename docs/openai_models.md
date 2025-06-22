# openai.go

## Overview

The `openai.go` file, found in the `internal/llm/models` package, is dedicated to defining constants and detailed metadata for various AI models provided by OpenAI. This information enables the OpenCode application to correctly identify, configure, interact with, and estimate costs for using OpenAI's GPT series and other models (like the "o1", "o3", "o4" series, which might be internal or newer codenames).

## Key Components

### Constants
- `ProviderOpenAI (ModelProvider)`: A constant of type `ModelProvider` (defined in `models.go`) set to "openai". This clearly identifies OpenAI as the provider for all models defined within this file.
- Model ID Constants: A comprehensive list of constants of type `ModelID` (defined in `models.go`) that serve as unique identifiers for specific OpenAI models within the OpenCode system. Examples include:
    - `GPT41`
    - `GPT41Mini`
    - `GPT41Nano`
    - `GPT45Preview`
    - `GPT4o`
    - `GPT4oMini`
    - `O1`, `O1Pro`, `O1Mini` (potentially representing a new series or variants)
    - `O3`, `O3Mini`
    - `O4Mini`

### Variables
- `OpenAIModels (map[ModelID]Model)`: A package-level map where:
    - Keys are the `ModelID` constants (e.g., `GPT4o`).
    - Values are `Model` structs (defined in `models.go`), each populated with specific metadata for an OpenAI model.
    - The metadata within each `Model` struct includes:
        - `ID (ModelID)`: The OpenCode internal unique ID for the model.
        - `Name (string)`: A user-friendly, human-readable name (e.g., "GPT 4o", "o1 mini").
        - `Provider (ModelProvider)`: Consistently set to `ProviderOpenAI`.
        - `APIModel (string)`: The precise model identifier string required when making API calls to OpenAI (e.g., "gpt-4o", "o1-mini").
        - `CostPer1MIn (float64)`: The cost in USD for processing 1 million input tokens.
        - `CostPer1MOut (float64)`: The cost in USD for generating 1 million output tokens.
        - `CostPer1MInCached (float64)`: Cost for cached input tokens, if applicable to OpenAI's pricing for that model.
        - `CostPer1MOutCached (float64)`: Cost for cached output tokens (often 0.0 if not a separate pricing tier).
        - `ContextWindow (int64)`: The maximum number of tokens (input + output) that the model can handle in a single interaction.
        - `DefaultMaxTokens (int64)`: A sensible default for the maximum number of tokens to request in a generation, usually less than the full context window.
        - `CanReason (bool)`: A flag indicating if the model is considered to support specific "reasoning" or "thinking" step functionalities, which might influence how prompts are constructed for it.
        - `SupportsAttachments (bool)`: A flag indicating if the model's API endpoint supports sending file attachments or binary data directly.

## Important Variables/Constants
- `ProviderOpenAI`: Crucial for identifying OpenAI as the LLM provider.
- `OpenAIModels`: This map is the core of the file, containing all the metadata for the supported OpenAI models. It's designed to be merged into the global `SupportedModels` map (in `models.go`).

## Usage Examples

This file primarily serves as a data source. Its contents are utilized by other parts of the OpenCode application, especially the configuration system (`internal/config/config.go`) and the OpenAI LLM provider client (`internal/llm/provider/openai.go`).

Accessing OpenAI model metadata:
```go
// import "github.com/opencode-ai/opencode/internal/llm/models"

// ...
// modelID := models.GPT4o
// modelMeta, isSupported := models.SupportedModels[modelID] // SupportedModels is the global map
//
// if isSupported && modelMeta.Provider == models.ProviderOpenAI {
//     fmt.Printf("Model Name: %s\n", modelMeta.Name)
//     fmt.Printf("OpenAI API Model String: %s\n", modelMeta.APIModel)
//     fmt.Printf("Cost per 1M Input Tokens: $%.2f\n", modelMeta.CostPer1MIn)
//     fmt.Printf("Context Window: %d tokens\n", modelMeta.ContextWindow)
// }
```

The `APIModel` string is essential for the OpenAI provider client when it formulates API requests. The cost details are vital for the agent (`internal/llm/agent/agent.go`) to accurately track session expenses. The capability flags like `CanReason` and `SupportsAttachments` inform how the agent and provider interact with the model.

## Dependencies and Interactions

- **Internal Dependencies:**
    - Depends on the `ModelProvider`, `ModelID`, and `Model` types defined in `models.go` (within the same package).
- **External Libraries:** None.
- **Interactions:**
    - Provides a structured and centralized definition for all supported OpenAI models and their specific properties.
    - This data is consumed by:
        - The application's configuration system (`internal/config/config.go`) for validating user-selected models, setting appropriate defaults when OpenAI is the chosen provider, and accessing model-specific parameters.
        - The OpenAI LLM provider client (`internal/llm/provider/openai.go`) to use the correct `APIModel` identifier for API interactions.
        - The core AI agent (`internal/llm/agent/agent.go`) for calculating operational costs and understanding the capabilities (e.g., context window size, attachment support) of the selected OpenAI model.
    - The `OpenAIModels` map is merged into the global `SupportedModels` map (defined in `models.go`) during the package initialization phase, making these OpenAI model definitions readily accessible throughout the application.

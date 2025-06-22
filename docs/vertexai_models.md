# vertexai.go

## Overview

The `vertexai.go` file, located within the `internal/llm/models` package, defines constants and metadata for AI models accessible via Google Cloud Vertex AI. Similar to `azure.go` for Azure OpenAI models, this file often references base model definitions (in this case, from `gemini.go`) for characteristics like cost and context window, indicating that Vertex AI provides access to Google's Gemini models.

## Key Components

### Constants
- `ProviderVertexAI (ModelProvider)`: A constant of type `ModelProvider` (defined in `models.go`) set to "vertexai". This identifies Google Cloud Vertex AI as the provider for the models defined here.
- Model ID Constants: Constants of type `ModelID` (defined in `models.go`) representing specific Gemini models as deployed on Vertex AI. These are namespaced with "vertexai.".
    - `VertexAIGemini25Flash`
    - `VertexAIGemini25` (likely referring to Gemini 2.5 Pro on Vertex AI)

### Variables
- `VertexAIGeminiModels (map[ModelID]Model)`: A package-level map where:
    - Keys are `ModelID` constants (e.g., `VertexAIGemini25Flash`).
    - Values are `Model` structs (defined in `models.go`) containing metadata for each Vertex AI-hosted Gemini model.
    - Much like the Azure model definitions, metadata fields such as `CostPer1MIn`, `ContextWindow`, and `DefaultMaxTokens` are typically populated by referencing the corresponding values from the base Gemini model definitions in the `GeminiModels` map (from `gemini.go`). For instance, `VertexAIGeminiModels[VertexAIGemini25Flash].CostPer1MIn` is set to `GeminiModels[Gemini25Flash].CostPer1MIn`.
    - Each `Model` struct includes:
        - `ID (ModelID)`: The unique OpenCode identifier for the Vertex AI model (e.g., `VertexAIGemini25Flash`).
        - `Name (string)`: A human-readable name (e.g., "VertexAI: Gemini 2.5 Flash").
        - `Provider (ModelProvider)`: Set to `ProviderVertexAI`.
        - `APIModel (string)`: The specific model identifier or endpoint name to be used when making API calls to Google Cloud Vertex AI (e.g., "gemini-2.5-flash-preview-04-17"). This might be the same as the base Gemini API model string.
        - Cost fields, `ContextWindow`, `DefaultMaxTokens`, `SupportsAttachments`: These properties are usually derived from the base Gemini model's metadata.

## Important Variables/Constants
- `ProviderVertexAI`: Identifies Google Cloud Vertex AI as the LLM provider.
- `VertexAIGeminiModels`: The map containing metadata for all supported Vertex AI-hosted Gemini models. This map is intended to be merged into the global `SupportedModels` map in `models.go`.

## Usage Examples

This file primarily serves as a data source. Its definitions are used by other parts of the OpenCode application, notably the configuration system (`internal/config/config.go`) and the Vertex AI LLM provider client (`internal/llm/provider/vertexai.go`).

Accessing Vertex AI model metadata:
```go
// import "github.com/opencode-ai/opencode/internal/llm/models"

// ...
// modelID := models.VertexAIGemini25
// modelMeta, ok := models.SupportedModels[modelID] // SupportedModels is the global map
// if ok && modelMeta.Provider == models.ProviderVertexAI {
//     fmt.Printf("Model Name: %s\n", modelMeta.Name)
//     fmt.Printf("Vertex AI API Model String: %s\n", modelMeta.APIModel)
//     // Costs and other properties are typically inherited from the base Gemini model
//     fmt.Printf("Context Window: %d tokens\n", modelMeta.ContextWindow)
// }
```

The `APIModel` string is crucial for the Vertex AI provider client when making API requests. Cost information, inherited from the base Gemini model rates, is used by the agent for session cost tracking.

## Dependencies and Interactions

- **Internal Dependencies:**
    - Relies on `ModelProvider`, `ModelID`, and `Model` types defined in `models.go`.
    - Critically depends on the `GeminiModels` map (from `gemini.go` in the same package) to populate many of the metadata fields for Vertex AI-hosted Gemini models.
- **External Libraries:** None.
- **Interactions:**
    - Provides a centralized definition for supported Vertex AI-hosted Gemini models, referencing base Gemini model characteristics.
    - This data is utilized by:
        - The configuration system (`internal/config/config.go`) for model validation and setting defaults when Vertex AI is the chosen provider.
        - The Vertex AI LLM provider client (`internal/llm/provider/vertexai.go`) to use the correct `APIModel` string for API interactions.
        - The AI agent (`internal/llm/agent/agent.go`) for cost calculation and understanding model capabilities.
    - The `VertexAIGeminiModels` map is merged into the global `SupportedModels` map (in `models.go`) during package initialization, making these definitions accessible application-wide.

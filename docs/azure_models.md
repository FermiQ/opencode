# azure.go

## Overview

The `azure.go` file, part of the `internal/llm/models` package, defines constants and metadata for AI models available through Microsoft Azure OpenAI Service. A key characteristic of this file is that many of the Azure model definitions (like cost, context window) directly reference the corresponding base OpenAI model definitions found in `openai.go`. This implies that Azure OpenAI often provides access to OpenAI's models, potentially with different API model names or endpoints but similar underlying capabilities and pricing structures.

## Key Components

### Constants
- `ProviderAzure (ModelProvider)`: A constant of type `ModelProvider` (defined in `models.go`) set to "azure". This identifies Azure as the provider for the models defined in this file.
- Model ID Constants: A series of constants of type `ModelID` (defined in `models.go`) representing specific Azure OpenAI models. These often mirror OpenAI model names but are prefixed with "azure." to distinguish them. Examples:
    - `AzureGPT41`
    - `AzureGPT41Mini`
    - `AzureGPT4o`
    - `AzureO1` (potentially representing an Azure-specific variant or an OpenAI model deployed on Azure)
    - ... and others.

### Variables
- `AzureModels (map[ModelID]Model)`: A package-level map where:
    - Keys are `ModelID` constants (e.g., `AzureGPT41`).
    - Values are `Model` structs (defined in `models.go`) containing detailed metadata for each Azure OpenAI model.
    - For many Azure models, the metadata fields (e.g., `CostPer1MIn`, `ContextWindow`, `DefaultMaxTokens`) are populated by referencing the values from the equivalent base model in the `OpenAIModels` map (defined in `openai.go`). For example, `AzureModels[AzureGPT41].CostPer1MIn` is set to `OpenAIModels[GPT41].CostPer1MIn`.
    - Each `Model` struct includes:
        - `ID (ModelID)`: The unique OpenCode identifier for the Azure model (e.g., `AzureGPT41`).
        - `Name (string)`: A human-readable name (e.g., "Azure OpenAI – GPT 4.1").
        - `Provider (ModelProvider)`: Set to `ProviderAzure`.
        - `APIModel (string)`: The specific model name or deployment ID to be used when making API calls to Azure OpenAI Service (e.g., "gpt-4.1"). This might be different from the base OpenAI model name.
        - `CostPer1MIn`, `CostPer1MOut`, `ContextWindow`, `DefaultMaxTokens`, `CanReason`, `SupportsAttachments`: These fields are typically derived from the corresponding base OpenAI model's metadata.

## Important Variables/Constants
- `ProviderAzure`: Identifies Azure as the LLM provider.
- `AzureModels`: The map containing all metadata for supported Azure OpenAI models. This map is intended to be merged into the global `SupportedModels` map in `models.go`.

## Usage Examples

This file primarily provides data definitions. Other parts of the application, such as the configuration system (`internal/config/config.go`) and the Azure LLM provider client (`internal/llm/provider/azure.go`), would consume this information.

Accessing Azure model metadata:
```go
// import "github.com/opencode-ai/opencode/internal/llm/models"

// ...
// modelID := models.AzureGPT4o
// modelMeta, ok := models.SupportedModels[modelID] // SupportedModels is the global map
// if ok && modelMeta.Provider == models.ProviderAzure {
//     fmt.Printf("Model Name: %s\n", modelMeta.Name)
//     fmt.Printf("Azure API Model (Deployment ID): %s\n", modelMeta.APIModel)
//     // Costs and other properties are often inherited from the base OpenAI model
//     fmt.Printf("Context Window: %d tokens\n", modelMeta.ContextWindow)
// }
```

The `APIModel` string is crucial for the Azure provider client, as it often corresponds to the "deployment name" required by the Azure OpenAI API.

## Dependencies and Interactions

- **Internal Dependencies:**
    - Relies on `ModelProvider`, `ModelID`, and `Model` types defined in `models.go` within the same package.
    - Critically depends on the `OpenAIModels` map (from `openai.go` in the same package) to populate many of the metadata fields for Azure models. This highlights the close relationship between Azure's offerings and OpenAI's base models.
- **External Libraries:** None.
- **Interactions:**
    - Provides a centralized definition of supported Azure OpenAI models and their properties, often by referencing base OpenAI model characteristics.
    - This data is consumed by:
        - The configuration system (`internal/config/config.go`) for validating model choices and set defaults when Azure is the selected provider.
        - The Azure LLM provider client (`internal/llm/provider/azure.go`) to determine the correct deployment name (`APIModel`) for API requests.
        - The agent (`internal/llm/agent/agent.go`) for cost calculation and understanding model capabilities.
    - The `AzureModels` map is merged into the global `SupportedModels` map (in `models.go`) to make these definitions accessible throughout the application.

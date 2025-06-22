# azure.go (in internal/llm/provider)

## Overview

The `azure.go` file, part of the `internal/llm/provider` package, implements the `Provider` interface for interacting with models hosted on Microsoft Azure OpenAI Service. It achieves this primarily by wrapping and configuring the existing `openaiClient` (defined in `openai.go`). The key difference is the setup of Azure-specific authentication and endpoint configuration.

## Key Components

### Structs
- `azureClient`: This struct embeds an `*openaiClient`. This means it inherits all the methods and fields of `openaiClient`, and thus, the core logic for interacting with the OpenAI API (message conversion, tool handling, streaming, etc.) is reused.
    - `openaiClient (*openaiClient)`: The embedded OpenAI client that handles the actual API communication.

### Types
- `AzureClient`: An alias for `ProviderClient` (which is an alias for the `Provider` interface), used for clarity when referring to an Azure OpenAI provider instance.

### Functions
- `newAzureClient(opts providerClientOptions) AzureClient`: Constructor for `azureClient`.
    1.  It retrieves Azure-specific configuration from environment variables:
        - `AZURE_OPENAI_ENDPOINT`: The endpoint URL for the Azure OpenAI resource (e.g., `https://your-resource.openai.azure.com`).
        - `AZURE_OPENAI_API_VERSION`: The API version to use (e.g., `2025-04-01-preview`).
    2.  **Fallback to Standard OpenAI Client**: If either `AZURE_OPENAI_ENDPOINT` or `AZURE_OPENAI_API_VERSION` is not set in the environment, this constructor defaults to creating and returning a standard `openaiClient` (using `newOpenAIClient(opts)`). This allows the "azure" provider type in the config to potentially still work with standard OpenAI if Azure specifics aren't provided via environment variables, though this might be unintentional or for specific deployment scenarios.
    3.  **Azure Configuration**: If both environment variables are present:
        - It prepares a slice of `option.RequestOption` for the `openai.NewClient` call.
        - It adds Azure-specific options using `azure.WithEndpoint()` to set the endpoint and API version.
        - **Authentication**: It then attempts to configure authentication in one of two ways:
            - **API Key**: If `opts.apiKey` (passed during provider creation) is set, or if `AZURE_OPENAI_API_KEY` environment variable is set, it uses `azure.WithAPIKey()`.
            - **Azure Identity (DefaultAzureCredential)**: If no API key is found, it attempts to create Azure Identity credentials using `azidentity.NewDefaultAzureCredential(nil)`. If successful, it configures the client with `azure.WithTokenCredential()`. This allows for managed identity or other Azure AD-based authentication methods.
        - It creates an `openaiClient` instance, passing these Azure-specific request options to `openai.NewClient()`.
        - Finally, it returns an `*azureClient` that embeds this specially configured `openaiClient`.

## Important Variables/Constants
This file does not define exported package-level constants or variables beyond the type alias and constructor. It relies on environment variables for its specific configuration.

## Usage Examples

This client is not typically instantiated directly by user code but by the generic `provider.NewProvider` function (in `provider.go`) when `models.ProviderAzure` is specified as the provider type.

```go
// Conceptual instantiation via generic provider factory:
// import "github.com/opencode-ai/opencode/internal/llm/provider"
// import "github.com/opencode-ai/opencode/internal/llm/models"

// // Ensure environment variables are set:
// // export AZURE_OPENAI_ENDPOINT="https://myresource.openai.azure.com"
// // export AZURE_OPENAI_API_VERSION="2024-02-01"
// // export AZURE_OPENAI_API_KEY="your_azure_openai_key" (or use Azure AD auth)

// opts := []provider.ProviderClientOption{
//     // provider.WithAPIKey("your_azure_openai_key"), // Can be set here or via env
//     provider.WithModel(models.SupportedModels[models.AzureGPT4o]), // Model ID specific to Azure deployment
//     provider.WithSystemMessage("You are an Azure OpenAI assistant."),
//     provider.WithMaxTokens(1024),
// }
// azureProvider, err := provider.NewProvider(models.ProviderAzure, opts...)

// // Interaction with azureProvider would then use the embedded openaiClient's methods:
// // eventChannel := azureProvider.StreamResponse(context.Background(), messages, tools)
// // for event := range eventChannel {
// //     // process event
// // }
```
The `APIModel` field in the `models.Model` struct for Azure models (e.g., `models.AzureGPT4o.APIModel`) is crucial, as this often corresponds to the **deployment name** of the model within the user's Azure OpenAI resource.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/llm/models`: For provider type constants like `models.ProviderAzure`.
    - Relies heavily on `openaiClient` (from `openai.go` in the same package) by embedding it. All actual LLM interaction logic (message formatting, streaming, tool use) is delegated to the embedded `openaiClient`.
    - Depends on `providerClientOptions` and the `Provider` interface from `provider.go`.
- **External Libraries:**
    - `github.com/Azure/azure-sdk-for-go/sdk/azidentity`: For Azure Active Directory based authentication (`NewDefaultAzureCredential`).
    - `github.com/openai/openai-go`: The official OpenAI Go SDK.
    - `github.com/openai/openai-go/azure`: Specific Azure helper functions within the OpenAI SDK for configuring endpoint and authentication.
    - `github.com/openai/openai-go/option`: For request options used by the OpenAI SDK.
    - `os`: For reading environment variables.
- **Interactions:**
    - Acts as a specialized configurator for the `openaiClient` to make it compatible with Azure OpenAI Service.
    - Reads Azure-specific endpoint and API version details from environment variables.
    - Supports both API key and Azure AD token-based authentication.
    - If Azure environment variables are missing, it gracefully falls back to behaving like a standard OpenAI client, which might be useful in some flexible deployment scenarios or for testing.

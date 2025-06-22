# local.go

## Overview

The `local.go` file in the `internal/llm/models` package is responsible for discovering and integrating locally hosted Large Language Models (LLMs) that expose an OpenAI-compatible API. This allows OpenCode to use models running on the user's machine (e.g., via LM Studio, Ollama) as if they were cloud-based providers. The discovery happens at an `init()` phase by querying a configured local endpoint.

## Key Components

### Constants
- `ProviderLocal (ModelProvider)`: Identifies "local" as the provider type.
- `localModelsPath ("v1/models")`: Standard OpenAI-compatible path for listing models.
- `lmStudioBetaModelsPath ("api/v0/models")`: An alternative path, possibly specific to older or beta versions of LM Studio.

### Structs
- `localModelList`: Helper struct to unmarshal the JSON response from the local model listing endpoint (which typically returns a list under a "data" key).
    - `Data ([]localModel)`
- `localModel`: Struct to unmarshal the individual model information returned by the local endpoint. Fields include:
    - `ID (string)`: The model ID as provided by the local server (e.g., "NousResearch/Hermes-2-Pro-Llama-3-8B-Q8_0.gguf").
    - `Object (string)`, `Type (string)`: Metadata, often "model" and "llm" respectively.
    - `Publisher (string)`, `Arch (string)`, `CompatibilityType (string)`, `Quantization (string)`, `State (string)`: Additional metadata provided by some local servers (like LM Studio), indicating model state (e.g., "loaded").
    - `MaxContextLength (int64)`, `LoadedContextLength (int64)`: Information about the model's context window.

### Initialization (`init` function)
- This function runs when the package is loaded.
- It checks for a `LOCAL_ENDPOINT` environment variable (e.g., `http://localhost:1234`).
- If the endpoint is set:
    1. It attempts to list models first from `lmStudioBetaModelsPath` and then from `localModelsPath` by making HTTP GET requests.
    2. If models are found, it calls `loadLocalModels` to process them.
    3. It sets a default dummy API key (`providers.local.apiKey`) in Viper, as local models typically don't require authentication.
    4. It sets the popularity score for `ProviderLocal` to 0, likely influencing its position in default model selection logic.

### Functions
- `listLocalModels(modelsEndpoint string) []localModel`:
    - Makes an HTTP GET request to the given `modelsEndpoint`.
    - Parses the JSON response into `localModelList`.
    - Filters the results to include only supported model types (e.g., for LM Studio, `Object` should be "model" and `Type` "llm").
    - Returns a slice of `localModel` structs.
- `loadLocalModels(models []localModel)`:
    - Iterates through the discovered `localModel`s.
    - Converts each `localModel` to the application's standard `Model` struct using `convertLocalModel`.
    - Adds the converted `Model` to the global `SupportedModels` map.
    - Sets default agent models in Viper configuration to the first discovered local model or any model that is explicitly "loaded". This makes a local model the default if available.
- `convertLocalModel(model localModel) Model`:
    - Converts a `localModel` (from the local server's API) into the internal `Model` struct used by OpenCode.
    - It prefixes the `ID` with "local." (e.g., "local.NousResearch/Hermes-2-Pro-Llama-3-8B-Q8_0.gguf").
    - Generates a `friendlyModelName` from the ID.
    - Sets `Provider` to `ProviderLocal`.
    - `APIModel` is the original ID from the local server.
    - Sets `ContextWindow` and `DefaultMaxTokens` based on `LoadedContextLength` or a fallback (4096).
    - Assumes local models `CanReason` and `SupportsAttachments` (these might be optimistic defaults).
- `friendlyModelName(modelID string) string`:
    - Attempts to create a more human-readable name from a potentially complex local model ID string.
    - It tries to parse out family, version, and label from the ID string using regex and string manipulation.
    - For example, "NousResearch/Hermes-2-Pro-Llama-3-8B-Q8_0.gguf" might become "Hermes 2-Pro-Llama-3-8B-Q8_0.gguf" or a simplified version depending on the regex.

## Important Variables/Constants
- `ProviderLocal`: Identifies the local provider.
- The `init()` function is crucial as it performs the dynamic discovery of local models.

## Usage Examples

This functionality is largely automatic if the `LOCAL_ENDPOINT` environment variable is set.

**Setting up a local model server (e.g., LM Studio):**
1. Run LM Studio, download a model, and start the server (typically at `http://localhost:1234`).
2. Set the environment variable: `export LOCAL_ENDPOINT="http://localhost:1234"`
3. Run OpenCode.

During startup, the `init()` function in `local.go` will:
- Query `http://localhost:1234/api/v0/models` (or `v1/models`).
- Retrieve the list of available/loaded models.
- Populate `models.SupportedModels` with these local models.
- Potentially set one of these local models as the default for agents in the configuration.

The user can then select these "local.*" models in OpenCode's configuration or UI.

## Dependencies and Interactions

- **Internal Dependencies:**
    - Relies on `ModelProvider`, `ModelID`, and `Model` types, and the `SupportedModels` map from `models.go`.
    - `github.com/opencode-ai/opencode/internal/logging`: For debug logging during model discovery.
- **External Libraries:**
    - `net/http`, `net/url`, `encoding/json`, `os`, `regexp`, `strings`, `unicode`, `cmp`: Standard Go libraries.
    - `github.com/spf13/viper`: For setting default configuration values (API key, default agent models).
- **Interactions:**
    - Dynamically discovers LLMs by making HTTP requests to an endpoint specified by `LOCAL_ENDPOINT`.
    - Modifies the global `SupportedModels` map by adding discovered local models.
    - Influences default application configuration by potentially setting a local model as the default for agents if one is found.
    - The `friendlyModelName` function attempts to improve the display name of complex local model IDs.

# config.go

## Overview

The `config.go` file is responsible for managing the application's configuration. It defines the structure of the configuration, loads settings from various sources (config files, environment variables), sets default values, validates the configuration, and provides access to the configuration settings throughout the application. It uses the `viper` library for handling configuration loading and merging.

## Key Components

### Types
- `MCPType`: An enum-like string type for Model Control Protocol server types (`stdio`, `sse`).
- `MCPServer`: Struct defining configuration for an MCP server (command, environment, args, type, URL, headers).
- `AgentName`: An enum-like string type for different AI agent roles (`coder`, `summarizer`, `task`, `title`).
- `Agent`: Struct defining configuration for an AI agent, including the `ModelID`, `MaxTokens`, and `ReasoningEffort`.
- `Provider`: Struct defining configuration for an LLM provider (API key, disabled status).
- `Data`: Struct for storage configuration (e.g., data directory).
- `LSPConfig`: Struct for Language Server Protocol client configuration (disabled status, command, args, options).
- `TUIConfig`: Struct for Terminal User Interface configuration (e.g., theme).
- `ShellConfig`: Struct for configuring the shell used by the bash tool (path, args).
- `Config`: The main configuration struct that aggregates all other configuration types.

### Constants
- `MCPStdio`, `MCPSse`: Supported MCP server types.
- `AgentCoder`, `AgentSummarizer`, `AgentTask`, `AgentTitle`: Defined agent names.
- `defaultDataDirectory`, `defaultLogLevel`, `appName`: Basic application defaults.
- `MaxTokensFallbackDefault`: A fallback value for max tokens if not otherwise specified.
- `defaultContextPaths`: A list of default file paths/patterns to search for contextual information (e.g., `AGENTS.md` equivalents).

### Variables
- `cfg`: A package-level global variable holding the loaded `Config` instance.

### Functions
- `Load(workingDir string, debug bool) (*Config, error)`: The main function to load and initialize the configuration. It's idempotent.
    - Configures `viper` to look for config files (e.g., `$HOME/.opencode.json`).
    - Sets default values for various settings.
    - Reads the global configuration file.
    - Merges a local configuration file (e.g., `.opencode.json` in the current project).
    - Sets defaults for LLM providers based on environment variables (e.g., `OPENAI_API_KEY`) and a predefined priority.
    - Unmarshals the configuration into the `cfg` struct.
    - Applies further default values that might depend on other settings.
    - Configures the global logger (`slog`) based on debug settings and environment variables.
    - Validates the loaded configuration using `Validate()`.
    - Overrides the max tokens for the `AgentTitle` specifically.
- `configureViper()`: Sets up `viper` with config file names, paths, and environment variable bindings.
- `setDefaults(debug bool)`: Sets basic default values in `viper` (data directory, TUI theme, shell path, debug status, log level).
- `setProviderDefaults()`: Sets default LLM providers and models for agents based on the presence of API keys in environment variables or config, following a specific priority order (Anthropic, OpenAI, Gemini, etc.).
- `hasAWSCredentials() bool`: Checks for AWS credentials in common environment variables.
- `hasVertexAICredentials() bool`: Checks for Vertex AI / Google Cloud credentials.
- `readConfig(err error) error`: Helper to handle errors from `viper.ReadInConfig()`, allowing config file not found errors.
- `mergeLocalConfig(workingDir string)`: Loads and merges a project-specific `.opencode.json` file from the `workingDir`.
- `applyDefaultValues()`: Applies defaults to certain fields after initial loading, like setting default `MCPType` if not specified.
- `validateAgent(cfg *Config, name AgentName, agent Agent) error`: Validates the configuration for a specific agent.
    - Checks if the model is supported.
    - Checks if the provider for the model is configured and enabled.
    - Sets default models or max tokens if current settings are invalid or missing.
    - Validates and defaults `ReasoningEffort` for compatible models.
- `Validate() error`: Validates the entire loaded configuration.
    - Calls `validateAgent` for each configured agent.
    - Validates provider configurations (disables them if API key is missing).
    - Validates LSP configurations (disables them if command is missing).
- `getProviderAPIKey(provider models.ModelProvider) string`: Retrieves a provider's API key from the corresponding environment variable.
- `setDefaultModelForAgent(agent AgentName) bool`: Attempts to set a default model for a given agent based on available and prioritized providers.
- `updateCfgFile(updateCfg func(config *Config)) error`: Reads the user's global config file, applies an update function to its parsed content, and writes it back. This is used for persisting changes like theme or model updates.
- `Get() *Config`: Returns the global `cfg` instance.
- `WorkingDirectory() string`: Returns the working directory from the configuration.
- `UpdateAgentModel(agentName AgentName, modelID models.ModelID) error`: Updates the model for a specific agent in both the in-memory `cfg` and the user's global config file. It re-validates the agent config after update.
- `UpdateTheme(themeName string) error`: Updates the TUI theme in both the in-memory `cfg` and the user's global config file.

## Important Variables/Constants
- `cfg (*Config)`: The global instance holding all application configuration. This is accessed via `config.Get()`.
- `defaultContextPaths ([]string)`: Defines standard locations for project-specific instructions for the AI.

## Usage Examples

Loading configuration at application startup (typically in `cmd/root.go`):
```go
import "github.com/opencode-ai/opencode/internal/config"

// ...
workingDir, _ := os.Getwd()
conf, err := config.Load(workingDir, debugMode)
if err != nil {
    // handle error
}
// Global config is now accessible via config.Get()
```

Accessing configuration values:
```go
currentConfig := config.Get()
theme := currentConfig.TUI.Theme
coderAgentConfig, ok := currentConfig.Agents[config.AgentCoder]
if ok {
    model := coderAgentConfig.Model
}
```

Updating a configuration value programmatically (e.g., theme from a TUI menu):
```go
err := config.UpdateTheme("dracula")
if err != nil {
    // handle error
}
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/llm/models`: For `models.ModelID`, `models.ModelProvider`, and `models.SupportedModels`.
    - `github.com/opencode-ai/opencode/internal/logging`: For `logging.NewWriter()` and configuring the global logger.
- **External Libraries:**
    - `github.com/spf13/viper`: Core library for configuration management (reading files, environment variables, setting defaults, merging).
    - `log/slog`: Standard library for structured logging, configured by this package.
- **Interactions:**
    - Reads configuration files from user's home directory and project directory.
    - Reads environment variables (e.g., `OPENCODE_DEBUG`, `OPENAI_API_KEY`).
    - Provides the rest of the application with access to configuration settings via `config.Get()`.
    - Initializes the application's global logger.
    - Can modify the user's global configuration file when functions like `UpdateAgentModel` or `UpdateTheme` are called.

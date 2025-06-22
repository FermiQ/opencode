# coder.go (in internal/llm/prompt)

## Overview

The `coder.go` file, part of the `internal/llm/prompt` package, is dedicated to generating the system prompt specifically for the "Coder" agent. The system prompt is a crucial piece of initial text given to the LLM to set its context, define its persona, capabilities, constraints, and instructions on how to behave and use tools. This file dynamically constructs parts of the prompt, such as environment information, and selects a base prompt template based on the LLM provider (e.g., OpenAI vs. Anthropic).

## Key Components

### Constants
- `baseOpenAICoderPrompt (string)`: A long, detailed string constant containing the base system prompt specifically tailored for OpenAI models. It instructs the agent on its role as "OpenCode CLI", its capabilities (streaming responses, function calls, sandboxed workspace), expected behavior (precision, safety, helpfulness), and strict guidelines for task execution, especially when modifying files (coding guidelines, avoiding unnecessary complexity, updating documentation, style consistency, avoiding copyright headers, checking `git status`, removing inline comments). It also provides instructions for non-coding tasks and how to reference files.
- `baseAnthropicCoderPrompt (string)`: A similarly detailed string constant for Anthropic models. While sharing some general goals with the OpenAI prompt, it has distinct sections and emphasis:
    - **Memory**: Instructs the agent on using an `OpenCode.md` file for storing commands, style preferences, and codebase info, and to proactively ask to update it.
    - **Tone and style**: Emphasizes conciseness, directness, explaining non-trivial bash commands, using Github-flavored markdown, and minimizing output tokens. It includes specific examples of desired brevity.
    - **Proactiveness**: Balances taking initiative with not surprising the user.
    - **Following conventions**: Stresses mimicking existing code style, checking for library usage, and security best practices.
    - **Code style**: Discourages comments unless necessary or requested.
    - **Doing tasks**: Recommends steps like using search tools, implementing, verifying with tests (and finding how to test), and running lint/typecheck commands. It explicitly states NOT to commit changes unless asked.
    - **Tool usage policy**: Prefers the `Agent` tool for file search to reduce context and advises on parallel tool calls. It also reminds the agent to summarize tool outputs for the user.

### Functions
- `CoderPrompt(provider models.ModelProvider) string`:
    - This is the main public function of the file.
    - It selects the appropriate base prompt (`baseOpenAICoderPrompt` or `baseAnthropicCoderPrompt`) based on the `provider` argument.
    - It calls `getEnvironmentInfo()` to gather details about the current working directory, git status, OS, date, and a listing of the current directory's contents.
    - It calls `lspInformation()` to get a standard blurb about LSP diagnostics if LSP is configured.
    - It then formats and combines the selected base prompt, the environment information, and the LSP information into the final system prompt string.
- `getEnvironmentInfo() string`:
    - Retrieves the current working directory using `config.WorkingDirectory()`.
    - Checks if the directory is a git repository using `isGitRepo()`.
    - Gets the operating system using `runtime.GOOS`.
    - Formats the current date.
    - Uses `tools.NewLsTool()` to list files in the current directory.
    - Formats all this information into a structured string, typically enclosed in `<env>` and `<project>` tags, to be appended to the system prompt.
- `isGitRepo(dir string) bool`:
    - Checks for the existence of a `.git` directory within the given `dir` to determine if it's a git repository. Returns `true` if `.git` exists, `false` otherwise.
- `lspInformation() string`:
    - Checks the application configuration (`config.Get()`) to see if any LSP servers are configured and not disabled.
    - If at least one active LSP server is found, it returns a predefined string explaining that diagnostics (linting, typechecking) will be available via tools and how they will be presented (e.g., within `<file_diagnostics>` tags). It also advises the agent to fix relevant issues.
    - If no active LSP server is configured, it returns an empty string.
- `boolToYesNo(b bool) string`:
    - A simple utility function to convert a boolean value to "Yes" or "No" string.

## Important Variables/Constants
- `baseOpenAICoderPrompt` and `baseAnthropicCoderPrompt`: These two multi-line string constants are the core templates that define the agent's persona, capabilities, and operational guidelines for different LLM backends.

## Usage Examples

This `CoderPrompt` function is typically called by the agent creation logic (`internal/llm/agent/agent.go`'s `createAgentProvider` function) to generate the system message that will be sent to the LLM provider as the first message in any new conversation or interaction.

```go
// Conceptual usage in agent setup:
// import "github.com/opencode-ai/opencode/internal/llm/prompt"
// import "github.com/opencode-ai/opencode/internal/llm/models"

// providerType := models.ProviderOpenAI // or models.ProviderAnthropic, etc.
// systemPromptForCoder := prompt.CoderPrompt(providerType)

// This systemPromptForCoder would then be used as the initial system message
// when initializing an LLM provider client or sending a new message sequence.
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.WorkingDirectory()` and accessing LSP configuration.
    - `github.com/opencode-ai/opencode/internal/llm/models`: For `models.ModelProvider` type to differentiate prompts.
    - `github.com/opencode-ai/opencode/internal/llm/tools`: Specifically uses `tools.NewLsTool()` to get a directory listing for the environment info.
- **External Libraries:**
    - `fmt`, `os`, `path/filepath`, `runtime`, `time`, `context`: Standard Go libraries.
- **Interactions:**
    - The output of `CoderPrompt` is fundamental to how the Coder agent behaves, as it directly instructs the LLM.
    - The content of the prompts (especially the detailed guidelines and constraints) significantly influences the quality, safety, and relevance of the agent's responses and actions.
    - The dynamic inclusion of environment and LSP information helps the LLM make more contextually aware decisions.
    - The choice between OpenAI and Anthropic base prompts allows for tuning instructions to the specific strengths or requirements of different LLM architectures.

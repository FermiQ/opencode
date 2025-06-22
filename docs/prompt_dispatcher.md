# prompt.go (in internal/llm/prompt)

## Overview

The `prompt.go` file in the `internal/llm/prompt` package serves as a central hub for generating system prompts for different AI agents (Coder, Title, Task, Summarizer). It selects the appropriate base prompt based on the agent type and LLM provider, and importantly, it dynamically loads and appends content from project-specific context files (like `OpenCode.md` or files in `.cursor/rules/`) to the prompts for Coder and Task agents. This allows for customization of agent behavior based on project-specific instructions.

## Key Components

### Variables
- `onceContext (sync.Once)`: Ensures that the project-specific context files are read and processed only once per application run, even if `getContextFromPaths` is called multiple times.
- `contextContent (string)`: A package-level variable that caches the concatenated content from all processed project-specific context files.

### Functions
- `GetAgentPrompt(agentName config.AgentName, provider models.ModelProvider) string`:
    - This is the main public function for retrieving a system prompt.
    - It takes the `agentName` (e.g., `config.AgentCoder`) and the `provider` (e.g., `models.ProviderOpenAI`) as input.
    - It uses a `switch` statement on `agentName` to call the appropriate prompt generation function:
        - `config.AgentCoder`: Calls `CoderPrompt(provider)` (from `coder.go`).
        - `config.AgentTitle`: Calls `TitlePrompt(provider)` (from `title.go`).
        - `config.AgentTask`: Calls `TaskPrompt(provider)` (from `task.go`).
        - `config.AgentSummarizer`: Calls `SummarizerPrompt(provider)` (from `summarizer.go`).
        - Default: Returns a generic "You are a helpful assistant" prompt.
    - If the `agentName` is `config.AgentCoder` or `config.AgentTask`, it then calls `getContextFromPaths()` to retrieve content from project-specific instruction files.
    - If `contextContent` is not empty, it appends this content to the base prompt, typically under a "Project-Specific Context" heading.
    - Returns the final, potentially augmented, system prompt.
- `getContextFromPaths() string`:
    - Uses `onceContext.Do` to ensure that the file processing logic within its closure runs only once.
    - Inside the closure, it retrieves the working directory (`cfg.WorkingDir`) and the list of configured context paths (`cfg.ContextPaths`) from the global application configuration (`config.Get()`).
    - It then calls `processContextPaths(workDir, contextPaths)` to do the actual file reading and concatenation.
    - The result from `processContextPaths` is stored in the package-level `contextContent` variable.
    - Returns the cached `contextContent`.
- `processContextPaths(workDir string, paths []string) string`:
    - Takes the working directory and a list of context file paths/directory patterns.
    - It processes each path concurrently using goroutines.
    - For each path:
        - If the path ends with `/`, it's treated as a directory. `filepath.WalkDir` is used to find all files within that directory recursively.
        - If it's a file path, it's processed directly.
        - It maintains a `processedFiles` map (with a mutex for concurrent access) to avoid reading and including the same file multiple times, even if specified through different paths or found via directory walk (case-insensitive check).
        - For each unique file to be processed, it calls `processFile()`.
    - It collects all non-empty results from `processFile` (which are file contents prefixed with their path) from a channel.
    - Finally, it joins all collected file contents into a single string, separated by newlines.
- `processFile(filePath string) string`:
    - Reads the content of the file at `filePath`.
    - If successful, it returns the file content prefixed with a comment line like `"# From: path/to/file.ext\n"`.
    - If there's an error reading the file (e.g., file not found), it returns an empty string, effectively skipping that file's content.

## Important Variables/Constants
- `onceContext` and `contextContent`: These manage the caching of project-specific instructions to avoid redundant file I/O.

## Usage Examples

This package is primarily used internally by the agent creation logic (`internal/llm/agent/agent.go`) to get the system prompt when an agent is initialized.

```go
// Conceptual usage in agent setup:
// import "github.com/opencode-ai/opencode/internal/llm/prompt"
// import "github.com/opencode-ai/opencode/internal/config"
// import "github.com/opencode-ai/opencode/internal/llm/models"

// agentType := config.AgentCoder
// providerType := models.ProviderOpenAI
//
// systemPrompt := prompt.GetAgentPrompt(agentType, providerType)
// This systemPrompt will include the base coder prompt for OpenAI,
// plus environment info, LSP info (from coder.go's CoderPrompt),
// and any content found in files like OpenCode.md or .cursor/rules/
// as defined in the application's config.ContextPaths.
```

If `config.ContextPaths` includes `["OpenCode.md", ".opencode_rules/"]` and these exist in the project:
- `OpenCode.md` content would be read.
- All files under `.opencode_rules/` would be read.
- Their contents, each prefixed by its path, would be appended to the Coder or Task agent's system prompt.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.AgentName` type, agent name constants, and accessing `config.Get()` for `WorkingDir` and `ContextPaths`.
    - `github.com/opencode-ai/opencode/internal/llm/models`: For `models.ModelProvider` type.
    - `github.com/opencode-ai/opencode/internal/logging`: For debug logging.
    - Depends on other files in the `prompt` package for specific agent prompts (e.g., `CoderPrompt` from `coder.go`, `TitlePrompt` from `title.go`).
- **External Libraries:**
    - `fmt`, `os`, `path/filepath`, `strings`, `sync`: Standard Go libraries.
- **Interactions:**
    - Acts as a dispatcher for different agent prompts.
    - Significantly enhances agent prompts by dynamically loading project-specific context from the file system, based on paths defined in the application configuration. This allows users to customize agent behavior per project.
    - Uses concurrency (`sync.WaitGroup`, goroutines, channels) in `processContextPaths` for potentially faster reading of multiple context files/directories.
    - Caches the loaded context to improve performance on subsequent calls.

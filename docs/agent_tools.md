# tools.go (in internal/llm/agent)

## Overview

The `tools.go` file, located in the `internal/llm/agent` package, is responsible for defining and assembling the sets of tools available to different types of AI agents within the OpenCode application. It provides functions that return slices of `tools.BaseTool` implementations, tailored for specific agent roles like the "Coder" agent and a more restricted "Task" agent.

## Key Components

### Functions

- `CoderAgentTools(permissions permission.Service, sessions session.Service, messages message.Service, history history.Service, lspClients map[string]*lsp.Client) []tools.BaseTool`:
    - This function assembles and returns a comprehensive list of tools intended for the main "Coder" agent.
    - It takes several service dependencies (`permissions`, `sessions`, `messages`, `history`) and `lspClients` to instantiate these tools.
    - The toolset includes:
        - `tools.NewBashTool`: For executing shell commands.
        - `tools.NewEditTool`: For applying edits to files (potentially using LSP).
        - `tools.NewFetchTool`: For fetching content from URLs.
        - `tools.NewGlobTool`: For finding files using glob patterns.
        - `tools.NewGrepTool`: For searching within files using grep.
        - `tools.NewLsTool`: For listing directory contents.
        - `tools.NewSourcegraphTool`: For interacting with a Sourcegraph instance (code search).
        - `tools.NewViewTool`: For viewing file content (potentially with LSP enhancements).
        - `tools.NewPatchTool`: For applying custom patches to files.
        - `tools.NewWriteTool`: For writing or overwriting file content.
        - `NewAgentTool` (from the same `agent` package): Allows the Coder agent to delegate tasks to a sub-agent.
    - It also calls `GetMcpTools(ctx, permissions)` to discover and include any tools available via configured MCP (Model Control Protocol) servers.
    - If `lspClients` are available, it includes `tools.NewDiagnosticsTool` for fetching code diagnostics.
    - All these tools are combined into a single slice.

- `TaskAgentTools(lspClients map[string]*lsp.Client) []tools.BaseTool`:
    - This function assembles and returns a more restricted set of tools, typically for a sub-agent or a "Task" agent that focuses on information retrieval rather than modification or execution.
    - The toolset includes:
        - `tools.NewGlobTool`
        - `tools.NewGrepTool`
        - `tools.NewLsTool`
        - `tools.NewSourcegraphTool`
        - `tools.NewViewTool`
    - This set notably excludes tools that modify files (Edit, Patch, Write) or execute arbitrary code (Bash).

## Important Variables/Constants

This file does not define exported package-level constants or variables. Its primary role is to compose toolsets.

## Usage Examples

These functions are typically called when an agent instance is being created, to equip it with its designated capabilities.

```go
// When creating a Coder Agent (conceptual):
// import "github.com/opencode-ai/opencode/internal/llm/agent"
// import "github.com/opencode-ai/opencode/internal/config"

// var (
//     permissionService permission.Service
//     sessionService    session.Service
//     messageService    message.Service
//     historyService    history.Service
//     lspClients        map[string]*lsp.Client
// )
// Initialize services...

// coderTools := agent.CoderAgentTools(
//     permissionService,
//     sessionService,
//     messageService,
//     historyService,
//     lspClients,
// )
// coderAgent, err := agent.NewAgent(config.AgentCoder, sessionService, messageService, coderTools)


// When creating a Task Agent (e.g., by the AgentTool):
// taskTools := agent.TaskAgentTools(lspClients)
// taskAgent, err := agent.NewAgent(config.AgentTask, sessionService, messageService, taskTools)
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/history`: For `history.Service`, used by tools like `EditTool`, `PatchTool`, `WriteTool`.
    - `github.com/opencode-ai/opencode/internal/llm/tools`: For all the individual tool constructors (e.g., `tools.NewBashTool`, `tools.NewGlobTool`). This is a major dependency.
    - `github.com/opencode-ai/opencode/internal/lsp`: For `lsp.Client` map, passed to tools that might use LSP features.
    - `github.com/opencode-ai/opencode/internal/message`: For `message.Service`, used by `NewAgentTool`.
    - `github.com/opencode-ai/opencode/internal/permission`: For `permission.Service`, passed to tools that require permission checks.
    - `github.com/opencode-ai/opencode/internal/session`: For `session.Service`, used by `NewAgentTool`.
    - Depends on `GetMcpTools` and `NewAgentTool` from the same `agent` package.
- **External Libraries:**
    - `context`: Used internally by `GetMcpTools`.
- **Interactions:**
    - These functions act as assemblers or factories for toolsets.
    - The specific tools included depend on the intended role of the agent (Coder vs. Task).
    - The availability of LSP clients can add further tools (like DiagnosticsTool) to the Coder agent.
    - Dynamically includes tools from MCP servers.

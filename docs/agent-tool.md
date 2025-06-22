# agent-tool.go

## Overview

The `agent-tool.go` file, located in the `internal/llm/agent` package, defines a tool named "agent". This tool allows a primary AI agent (e.g., the Coder agent) to delegate a sub-task to another, new AI agent instance. This sub-agent operates with a more restricted set of tools (specifically, tools suitable for information retrieval like Glob, Grep, LS, View) and then returns its findings to the calling (primary) agent. This is a form of agent composition or hierarchical task delegation.

## Key Components

### Structs
- `agentTool`: Implements the `tools.BaseTool` interface.
    - `sessions (session.Service)`: Service for managing sessions.
    - `messages (message.Service)`: Service for managing messages.
    - `lspClients (map[string]*lsp.Client)`: A map of active LSP clients, passed to the sub-agent's tools if needed (though the description implies the sub-agent has a limited toolset that might not directly use LSP).
- `AgentParams`: Defines the expected JSON parameters for the "agent" tool.
    - `Prompt (string)`: The detailed prompt/task for the sub-agent to execute.

### Constants
- `AgentToolName ("agent")`: The name under which this tool is registered and can be called by an LLM.

### Functions
- `(b *agentTool) Info() tools.ToolInfo`: Implements `tools.BaseTool.Info`.
    - Returns metadata about the "agent" tool, including its name, a detailed description of its purpose and usage guidelines, and the expected parameters (`prompt`).
    - The description emphasizes when to use this tool (for complex searches where multiple steps might be needed) and when not to (for simple file reads or specific class searches where direct tools like `View` or `GlobTool` are better).
    - It highlights that sub-agents are stateless, their results are not directly visible to the user (the calling agent must summarize), and they cannot use tools that modify files (like Bash, Replace, Edit).
- `(b *agentTool) Run(ctx context.Context, call tools.ToolCall) (tools.ToolResponse, error)`: Implements `tools.BaseTool.Run`. This is the core logic executed when the LLM calls the "agent" tool.
    1.  Parses the `call.Input` (JSON string) into `AgentParams`.
    2.  Retrieves `sessionID` and `messageID` from the context (these belong to the calling agent's session).
    3.  Creates a new AI agent instance (`NewAgent`) specifically configured as `config.AgentTask`. This task agent is equipped with a restricted set of tools (`TaskAgentTools`).
    4.  Creates a new "task session" (`b.sessions.CreateTaskSession`) which is a child or sub-session linked to the calling agent's tool call ID and original session ID. This isolates the sub-agent's interactions.
    5.  Runs the newly created sub-agent with the prompt from `AgentParams` within the new task session (`agent.Run`). This is an asynchronous operation; it waits for the sub-agent to complete its work (`<-done`).
    6.  Once the sub-agent completes, its final message (`result.Message`) is retrieved.
    7.  The cost incurred by the sub-agent's session is fetched and added to the parent (calling agent's) session's cost.
    8.  The content of the sub-agent's final message is returned as a `tools.NewTextResponse` to the calling agent.
- `NewAgentTool(Sessions session.Service, Messages message.Service, LspClients map[string]*lsp.Client) tools.BaseTool`: Constructor function for `agentTool`. It takes necessary service dependencies and returns a new instance.

## Important Variables/Constants
- `AgentToolName`: The registered name of this tool.

## Usage Examples

This tool is not called directly by user code but by an LLM that has been configured to use it. An LLM, when faced with a complex query, might decide to invoke this tool.

Hypothetical LLM Tool Call (as part of a larger LLM response):
```json
{
  "type": "tool_use",
  "id": "tool_call_123",
  "name": "agent",
  "input": {
    "prompt": "Search through all '.go' files in the 'internal/auth' directory for functions related to 'password hashing'. Summarize the findings, listing function names and their primary purpose."
  }
}
```
When the `agentTool.Run` method processes this, it will:
1. Spin up a new `config.AgentTask` agent.
2. Provide it with the prompt: "Search through all '.go' files...".
3. This sub-agent will use its limited tools (Glob, Grep, View, LS) to find the information.
4. The sub-agent will produce a final message (e.g., "Found functions: `hashPassword` which uses bcrypt, `verifyPassword` which compares hashes.").
5. This final message is returned to the original LLM that called the "agent" tool.
6. The original LLM then uses this information to formulate its response to the user.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.AgentTask` to specify the type of sub-agent.
    - `github.com/opencode-ai/opencode/internal/llm/tools`: For `tools.BaseTool`, `tools.ToolInfo`, `tools.ToolCall`, `tools.ToolResponse` interfaces and types.
    - `github.com/opencode-ai/opencode/internal/lsp`: For `lsp.Client` (though its direct use by the sub-agent's limited tools is less clear from this file alone).
    - `github.com/opencode-ai/opencode/internal/message`: For `message.Service` and message roles.
    - `github.com/opencode-ai/opencode/internal/session`: For `session.Service` to manage task sessions.
    - Relies on other functions within the `agent` package like `NewAgent` (constructor for agents) and `TaskAgentTools` (to define the toolset for the sub-agent).
- **External Libraries:**
    - `encoding/json`: For parsing tool input parameters.
- **Interactions:**
    - This tool acts as a bridge, allowing one agent to invoke another specialized agent.
    - It creates temporary, isolated "task sessions" for these sub-agents.
    - It aggregates costs from sub-agent sessions back to the parent session.
    - The sub-agent created by this tool has a restricted set of tools, focused on information retrieval, and cannot modify files.

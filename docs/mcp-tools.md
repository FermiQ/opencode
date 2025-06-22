# mcp-tools.go

## Overview

The `mcp-tools.go` file, within the `internal/llm/agent` package, facilitates the integration of external tools that conform to the Model Control Protocol (MCP). MCP is a standardized way for LLMs or agents to discover and interact with tools provided by separate processes or services. This file defines structures and functions to wrap these MCP-compliant tools so they can be used by the OpenCode agent system.

## Key Components

### Structs
- `mcpTool`: Implements the `tools.BaseTool` interface, acting as a wrapper around an MCP-defined tool.
    - `mcpName (string)`: The name of the MCP server/source providing the tool (e.g., "my_mcp_service").
    - `tool (mcp.Tool)`: The actual tool definition retrieved from the MCP server, containing its name, description, and schema.
    - `mcpConfig (config.MCPServer)`: Configuration for the MCP server (command for stdio, URL for SSE, etc.).
    - `permissions (permission.Service)`: Service to handle permission requests before executing the MCP tool.
- `MCPClient`: An interface defining the methods for interacting with an MCP server. This allows for different MCP client implementations (e.g., Stdio, SSE).
    - `Initialize(ctx context.Context, request mcp.InitializeRequest) (*mcp.InitializeResult, error)`
    - `ListTools(ctx context.Context, request mcp.ListToolsRequest) (*mcp.ListToolsResult, error)`
    - `CallTool(ctx context.Context, request mcp.CallToolRequest) (*mcp.CallToolResult, error)`
    - `Close() error`

### Variables
- `mcpTools ([]tools.BaseTool)`: A package-level cache for discovered MCP tools.

### Functions
- `(b *mcpTool) Info() tools.ToolInfo`: Implements `tools.BaseTool.Info`. It constructs a `tools.ToolInfo` object from the `mcp.Tool` definition, prefixing the MCP tool's name with `mcpName_` to ensure uniqueness within the OpenCode agent's toolset.
- `runTool(ctx context.Context, c MCPClient, toolName string, input string) (tools.ToolResponse, error)`: A helper function that handles the common logic of calling an MCP tool via an `MCPClient`.
    1. Initializes the MCP client.
    2. Constructs an `mcp.CallToolRequest` with the `toolName` and parsed `input` arguments.
    3. Calls the tool via `c.CallTool()`.
    4. Formats the tool's output (assuming text content) into a `tools.ToolResponse`.
    5. Closes the MCP client.
- `(b *mcpTool) Run(ctx context.Context, params tools.ToolCall) (tools.ToolResponse, error)`: Implements `tools.BaseTool.Run`. This is executed when the OpenCode agent decides to use a specific MCP tool.
    1. Requests permission from the `permission.Service` to execute the tool with the given parameters.
    2. If permission is denied, returns an error response.
    3. Based on `b.mcpConfig.Type` (either `config.MCPStdio` or `config.MCPSse`):
        - Creates the appropriate MCP client (`client.NewStdioMCPClient` or `client.NewSSEMCPClient`).
        - Calls `runTool` to execute the tool via the client.
- `NewMcpTool(name string, tool mcp.Tool, permissions permission.Service, mcpConfig config.MCPServer) tools.BaseTool`: Constructor for `mcpTool`.
- `getTools(ctx context.Context, name string, m config.MCPServer, permissions permission.Service, c MCPClient) []tools.BaseTool`: Helper function to discover tools from a single MCP client.
    1. Initializes the client.
    2. Calls `c.ListTools()` to get available tools.
    3. For each tool returned by the MCP server, creates an `mcpTool` wrapper using `NewMcpTool`.
    4. Closes the client and returns the list of wrapped tools.
- `GetMcpTools(ctx context.Context, permissions permission.Service) []tools.BaseTool`: The main function to discover and return all available MCP tools from all configured MCP servers.
    1. Returns cached `mcpTools` if already populated.
    2. Iterates through `config.Get().MCPServers`.
    3. For each configured MCP server:
        - Creates the appropriate client (Stdio or SSE).
        - Calls `getTools` to fetch and wrap tools from that server.
        - Appends these tools to the `mcpTools` cache.
    4. Returns the populated `mcpTools` slice.

## Important Variables/Constants
- `mcpTools`: Serves as a cache to avoid re-discovering MCP tools on every request.

## Usage Examples

MCP tools are not used directly by end-user code but are made available to the AI agent. The agent, based on its configuration and the user's prompt, can decide to use one of these discovered MCP tools.

**Configuration (in `~/.opencode.json` or `.opencode.json`):**
```json
{
  "mcpServers": {
    "my_calculator_service": {
      "command": "/path/to/my_calculator_mcp_server",
      "type": "stdio"
    },
    "weather_service_sse": {
      "url": "http://localhost:8080/mcp",
      "type": "sse"
    }
  }
  // ... other configs
}
```

When the OpenCode agent starts, `GetMcpTools` would be called. This function would:
1. For `my_calculator_service`:
    - Launch `/path/to/my_calculator_mcp_server` as a stdio MCP client.
    - Communicate with it to list its tools (e.g., "add", "subtract").
    - Wrap these as `mcpTool` instances (e.g., `my_calculator_service_add`, `my_calculator_service_subtract`).
2. For `weather_service_sse`:
    - Connect to `http://localhost:8080/mcp` as an SSE MCP client.
    - List its tools (e.g., "get_current_weather").
    - Wrap it (e.g., `weather_service_sse_get_current_weather`).

These wrapped tools then become available to the OpenCode agent like any other built-in tool.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.MCPServer` configuration and `config.Get()`.
    - `github.com/opencode-ai/opencode/internal/llm/tools`: For `tools.BaseTool`, `tools.ToolInfo`, etc.
    - `github.com/opencode-ai/opencode/internal/logging`: For logging errors during MCP client interactions.
    - `github.com/opencode-ai/opencode/internal/permission`: For `permission.Service` to authorize tool execution.
    - `github.com/opencode-ai/opencode/internal/version`: For providing client version info during MCP initialization.
- **External Libraries:**
    - `github.com/mark3labs/mcp-go/client`: For MCP client implementations (Stdio, SSE).
    - `github.com/mark3labs/mcp-go/mcp`: For MCP protocol message types and constants.
    - `encoding/json`: For parsing tool input parameters.
- **Interactions:**
    - This module extends the agent's capabilities by dynamically discovering and integrating tools from external MCP-compliant servers.
    - It handles the lifecycle of MCP clients (creation, initialization, closing) for each tool discovery or execution.
    - It translates between the OpenCode agent's internal tool system and the MCP standard.
    - All MCP tool executions are subject to the OpenCode permission system.

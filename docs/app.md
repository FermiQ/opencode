# app.go

## Overview

The `app.go` file defines the central `App` struct, which encapsulates the core services and functionalities of the OpenCode application. It is responsible for initializing these services, managing their lifecycles, and coordinating their interactions. This includes session management, message handling, history tracking, permissions, AI agent operations, and Language Server Protocol (LSP) client management.

## Key Components

### Structs
- `App`: The main application struct. It holds instances of various services:
    - `Sessions`: Manages user sessions (`session.Service`).
    - `Messages`: Manages messages within sessions (`message.Service`).
    - `History`: Manages file history and related operations (`history.Service`).
    - `Permissions`: Handles permission requests from the AI agent (`permission.Service`).
    - `CoderAgent`: The AI agent responsible for processing user prompts and interacting with tools (`agent.Service`).
    - `LSPClients`: A map of active LSP clients, keyed by language or project root.
    - Internal fields for managing LSP client initialization and shutdown synchronization (`clientsMutex`, `watcherCancelFuncs`, `cancelFuncsMutex`, `watcherWG`).

### Functions
- `New(ctx context.Context, conn *sql.DB) (*App, error)`: Constructor for the `App` struct. It initializes all the services (sessions, messages, history, permissions, coder agent) and starts the LSP client initialization in the background. It also initializes the TUI theme based on the configuration.
- `(app *App) initTheme()`: Initializes the application's TUI theme based on the `Theme` setting in the global configuration. If the theme is not set or invalid, it defaults to a pre-defined theme.
- `(a *App) RunNonInteractive(ctx context.Context, prompt string, outputFormat string, quiet bool) error`: Handles the application flow when `opencode` is run with a specific prompt directly from the command line (non-interactive mode). It creates a temporary session, runs the coder agent with the prompt, automatically approves permissions for this session, and prints the agent's response to standard output in the specified format.
- `(app *App) Shutdown()`: Performs a graceful shutdown of the application. This includes:
    - Cancelling any background watcher goroutines related to LSP or other services.
    - Shutting down all active LSP clients.

## Important Variables/Constants

This file does not define package-level exported constants or variables. The primary component is the `App` struct and its methods.

## Usage Examples

The `App` struct is typically instantiated in the main command execution flow (e.g., in `cmd/root.go`):

```go
// Example (conceptual, actual instantiation in cmd/root.go)
conn, _ := db.Connect() // Assume db connection is established
ctx := context.Background()
application, err := app.New(ctx, conn)
if err != nil {
    // Handle error
}
// Use application instance...

// Later, on program exit:
application.Shutdown()
```

Running in non-interactive mode is handled by `root.go` calling `RunNonInteractive`:
```go
// Conceptual call from cmd/root.go
err := application.RunNonInteractive(ctx, "Explain this code", "text", false)
if err != nil {
    // Handle error
}
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: To access global configuration, like the TUI theme.
    - `github.com/opencode-ai/opencode/internal/db`: For database query interface (`db.New`).
    - `github.com/opencode-ai/opencode/internal/format`: For output formatting in non-interactive mode and spinner.
    - `github.com/opencode-ai/opencode/internal/history`: Provides `history.Service`.
    - `github.com/opencode-ai/opencode/internal/llm/agent`: Provides `agent.Service` (CoderAgent) and agent tools.
    - `github.com/opencode-ai/opencode/internal/logging`: For application-wide logging.
    - `github.com/opencode-ai/opencode/internal/lsp`: For LSP client management. (The `initLSPClients` and related fields suggest this, though the actual initialization logic for `LSPClients` isn't fully shown in `app.go` but in `lsp.go`).
    - `github.com/opencode-ai/opencode/internal/message`: Provides `message.Service`.
    - `github.com/opencode-ai/opencode/internal/permission`: Provides `permission.Service`.
    - `github.com/opencode-ai/opencode/internal/session`: Provides `session.Service`.
    - `github.com/opencode-ai/opencode/internal/tui/theme`: For setting the TUI theme.
- **External Libraries:**
    - `database/sql`: For the `sql.DB` connection object.
- **Interactions:**
    - The `App` struct acts as a central coordinator for various services.
    - It is created by the main command (`cmd/root.go`) and its lifecycle (creation and shutdown) is managed there.
    - `CoderAgent` interacts with `Sessions`, `Messages`, `Permissions`, `History`, and `LSPClients` (via tools) to fulfill user requests.
    - The `RunNonInteractive` method orchestrates a complete request-response cycle with the `CoderAgent` for command-line prompts.
    - It manages the startup and shutdown of LSP clients, which are used by the agent for code-related tasks.

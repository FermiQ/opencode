# lsp.go

## Overview

The `lsp.go` file, within the `app` package, is responsible for managing the lifecycle of Language Server Protocol (LSP) clients. It handles the initialization, starting, monitoring, and restarting of LSP clients based on the application's configuration. Each LSP client typically corresponds to a specific language server defined in the user's settings.

## Key Components

This file extends the `App` struct defined in `app.go` with methods related to LSP client management.

### Functions
- `(app *App) initLSPClients(ctx context.Context)`: This method is called during the `App` initialization. It iterates through the LSP configurations defined in `config.Get().LSP` and launches a goroutine for each one to create and start the LSP client using `createAndStartLSPClient`.
- `(app *App) createAndStartLSPClient(ctx context.Context, name string, command string, args ...string)`: This function handles the creation and initialization of a single LSP client.
    - It creates a new LSP client (`lsp.NewClient`).
    - It initializes the LSP client with the server (`lspClient.InitializeLSPClient`), using a timeout.
    - It waits for the server to report readiness (`lspClient.WaitForServerReady`).
    - It creates a `watcher.NewWorkspaceWatcher` to monitor workspace changes for this client.
    - It stores a cancel function for the watcher's context, allowing it to be stopped gracefully during application shutdown.
    - It adds the newly created and initialized `lspClient` to the `app.LSPClients` map.
    - It starts the workspace watcher in a new goroutine via `app.runWorkspaceWatcher`.
- `(app *App) runWorkspaceWatcher(ctx context.Context, name string, workspaceWatcher *watcher.WorkspaceWatcher)`: This function runs the `workspaceWatcher.WatchWorkspace` method. It's designed to run in its own goroutine. It includes panic recovery logic, which attempts to restart the LSP client if the watcher crashes.
- `(app *App) restartLSPClient(ctx context.Context, name string)`: This method is called to attempt a restart of an LSP client, typically after a crash. It retrieves the client's original configuration, cleans up the old client instance (including attempting a graceful shutdown), and then calls `createAndStartLSPClient` to set up a new instance.

## Important Variables/Constants

This file does not define package-level exported constants or variables. Its primary focus is on methods attached to the `App` struct for LSP management. The `app.LSPClients` map (defined in `app.go`) is populated and managed by the functions in this file.

## Usage Examples

The LSP client initialization is triggered internally when a new `App` instance is created:

```go
// Conceptual: When app.New() is called, it internally calls app.initLSPClients(ctx)
// app.initLSPClients(ctx) then launches goroutines for each configured LSP server.

// Example of how a client might be configured in the config file (e.g., config.yaml)
// lsp:
//   gopls:
//     command: "gopls"
//     args: ["-logfile=auto", "-rpc.trace"]
//   pylsp:
//     command: "pylsp"
```

The `restartLSPClient` is typically called from a panic recovery handler within `runWorkspaceWatcher`.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: To get LSP server configurations (`config.Get().LSP`) and the working directory (`config.WorkingDirectory()`).
    - `github.com/opencode-ai/opencode/internal/logging`: For application-wide logging.
    - `github.com/opencode-ai/opencode/internal/lsp`: For the core LSP client functionality (`lsp.NewClient`, `lspClient.InitializeLSPClient`, etc.) and LSP client state constants.
    - `github.com/opencode-ai/opencode/internal/lsp/watcher`: For the `watcher.WorkspaceWatcher` which monitors file system events and communicates them to the LSP server.
- **External Libraries:**
    - None directly, but the `lsp` package itself interacts with external LSP server processes.
- **Interactions:**
    - This file's functions are methods of the `App` struct and are responsible for populating and managing `app.LSPClients`.
    - It interacts with the global configuration to discover and configure LSP clients.
    - It starts background goroutines for each LSP client's workspace watcher. These watchers monitor the filesystem and interact with their respective LSP clients.
    - The `App.Shutdown()` method (in `app.go`) uses `app.watcherCancelFuncs` (populated by `createAndStartLSPClient`) to signal these watchers to stop.
    - The LSP clients stored in `app.LSPClients` are made available to other parts of the application, particularly the AI agent's tools, for code intelligence features.

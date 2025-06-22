# handlers.go (in internal/lsp)

## Overview

The `handlers.go` file in the `internal/lsp` package contains handler functions for various requests and notifications that an LSP server can send to the LSP client. These handlers define how the OpenCode application (acting as the client) should respond to or process these server-initiated communications. This includes handling workspace edits, dynamic capability registrations, server messages, and diagnostic updates.

## Key Components

### Request Handlers
These functions are designed to be registered with the `lsp.Client` and are invoked when the LSP server sends a corresponding request to the client.

- `HandleWorkspaceConfiguration(params json.RawMessage) (any, error)`:
    - Handles the `workspace/configuration` request from the server.
    - Currently, it returns a default empty configuration (`[]map[string]any{{}}`), implying that the client doesn't provide specific dynamic configurations to the server via this mechanism, or that the servers being interacted with don't require detailed responses here.
- `HandleRegisterCapability(params json.RawMessage) (any, error)`:
    - Handles the `client/registerCapability` request. This is used by servers to dynamically register capabilities with the client after initialization.
    - It unmarshals the `protocol.RegistrationParams`.
    - It iterates through the registrations. Of particular interest is the `workspace/didChangeWatchedFiles` method:
        - If a server registers for file watching, this handler parses the `DidChangeWatchedFilesRegistrationOptions`.
        - It then calls `notifyFileWatchRegistration` to pass these watcher configurations to a registered handler (likely in the `lsp/watcher` package).
    - Returns `nil, nil` indicating success.
- `HandleApplyEdit(params json.RawMessage) (any, error)`:
    - Handles the `workspace/applyEdit` request, which a server sends when it wants the client to apply textual changes to workspace files.
    - It unmarshals the `protocol.ApplyWorkspaceEditParams`.
    - It calls `util.ApplyWorkspaceEdit(edit.Edit)` (from `lsp/util/edit.go`) to perform the actual file modifications based on the `WorkspaceEdit` data.
    - Returns a `protocol.ApplyWorkspaceEditResult` indicating whether the edit was applied successfully and any failure reason.

### File Watch Registration Handling
- `FileWatchRegistrationHandler (func(id string, watchers []protocol.FileSystemWatcher))`: A type definition for a function that handles file watch registrations.
- `fileWatchHandler (FileWatchRegistrationHandler)`: A package-level variable to hold the currently registered handler.
- `RegisterFileWatchHandler(handler FileWatchRegistrationHandler)`: Allows other packages (specifically `lsp/watcher`) to set the `fileWatchHandler`.
- `notifyFileWatchRegistration(id string, watchers []protocol.FileSystemWatcher)`: If `fileWatchHandler` is set, this function calls it with the registration ID and the list of watchers provided by the LSP server.

### Notification Handlers
These functions are registered with the `lsp.Client` and are invoked when the server sends a corresponding notification.

- `HandleServerMessage(params json.RawMessage)`:
    - Handles the `window/showMessage` notification from the server.
    - It unmarshals the message parameters (type and message string).
    - If `DebugLSP` is enabled in the config, it logs the server message using `logging.Debug()`. (Currently, it doesn't display these messages to the end-user in the TUI, only logs them).
- `HandleDiagnostics(client *Client, params json.RawMessage)`:
    - Handles the `textDocument/publishDiagnostics` notification, which is how LSP servers send errors, warnings, etc., for documents.
    - It unmarshals the `protocol.PublishDiagnosticsParams`.
    - It then updates the `diagnostics` cache within the provided `lsp.Client` instance, storing the received diagnostics keyed by their document URI. This is protected by `client.diagnosticsMu.Lock()`.

## Important Variables/Constants
- `fileWatchHandler`: Central point for dispatching file watch registration details to the actual file watching mechanism.

## Usage Examples

These handlers are not called directly by typical application code but are registered with an `lsp.Client` instance, usually during or shortly after the client's initialization.

Registering a handler in `lsp/client.go` (conceptual):
```go
// Inside lsp.Client.InitializeLSPClient or similar:
// c.RegisterServerRequestHandler("workspace/applyEdit", lsp.HandleApplyEdit)
// c.RegisterNotificationHandler("textDocument/publishDiagnostics",
//     func(params json.RawMessage) { lsp.HandleDiagnostics(c, params) })
```

When the LSP server sends a `textDocument/publishDiagnostics` notification, the `lsp.Client`'s message handling loop (in `protocol.go` or `client.go`) would identify the method and call the registered `HandleDiagnostics` function with the client instance and the notification parameters.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.Get()` to check `DebugLSP`.
    - `github.com/opencode-ai/opencode/internal/logging`: For logging errors and debug messages.
    - `github.com/opencode-ai/opencode/internal/lsp/protocol`: For all LSP-specific data structures (e.g., `RegistrationParams`, `ApplyWorkspaceEditParams`, `PublishDiagnosticsParams`).
    - `github.com/opencode-ai/opencode/internal/lsp/util`: For `util.ApplyWorkspaceEdit` to apply changes to files.
    - Relies on the `Client` struct (from `client.go` in the same package) for `HandleDiagnostics` to update the client's diagnostic cache.
- **External Libraries:**
    - `encoding/json`: For unmarshalling LSP message parameters.
- **Interactions:**
    - These handlers are crucial for the client to correctly process and react to messages initiated by the LSP server.
    - `HandleApplyEdit` allows the LSP server to remotely modify files in the user's workspace (with user permission presumably handled at a higher level or by the nature of the edit).
    - `HandleRegisterCapability` (specifically for file watching) bridges the LSP server's dynamic file watching requests with the client-side file watching implementation (`internal/lsp/watcher`).
    - `HandleDiagnostics` is fundamental for collecting and caching diagnostic information, which is then used by tools like the "diagnostics" tool.

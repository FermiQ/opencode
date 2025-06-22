# permission.go (in internal/permission)

## Overview

The `permission.go` file, located in the `internal/permission` package, implements a service for managing permissions for tool executions. When a tool (like "bash" or "edit") attempts to perform a potentially sensitive action (e.g., writing to a file, executing a non-read-only command), it uses this service to request permission. The service can then, for example, publish a request to the TUI, allowing the user to grant or deny the action. It also supports "persisted" session permissions and an auto-approval mechanism for specific sessions (likely non-interactive ones).

## Key Components

### Errors
- `ErrorPermissionDenied`: An exported error variable that tools can return when a permission request is denied.

### Structs
- `CreatePermissionRequest`: Struct used to initiate a permission request.
    - `SessionID (string)`: The session in which the permission is requested.
    - `ToolName (string)`: The name of the tool requesting permission.
    - `Description (string)`: A human-readable description of the action requiring permission.
    - `Action (string)`: The specific action being requested (e.g., "write", "execute").
    - `Params (any)`: The parameters of the tool call, for context.
    - `Path (string)`: The file or directory path relevant to the permission.
- `PermissionRequest`: Struct representing a pending or granted permission.
    - `ID (string)`: Unique ID for this permission request instance.
    - Fields are the same as `CreatePermissionRequest`.
- `service`: The concrete implementation of the `Service` interface.
    - `Broker (*pubsub.Broker[PermissionRequest])`: Pub/sub broker to publish `PermissionRequest` events (likely to the TUI).
    - `sessionPermissions ([]PermissionRequest)`: A slice to store permissions that have been "persistently" granted for the current application lifecycle (not across restarts).
    - `pendingRequests (sync.Map)`: A map to store channels for pending permission requests, keyed by request ID. This allows the `Request` method to block until a grant/deny response is received.
    - `autoApproveSessions ([]string)`: A list of session IDs for which all permission requests are automatically approved.

### Interfaces
- `Service`: Defines the contract for the permission management service.
    - `pubsub.Suscriber[PermissionRequest]`: Embeds pub/sub subscription for permission request events.
    - `GrantPersistant(permission PermissionRequest)`: Grants the permission and adds it to `sessionPermissions` so subsequent identical requests in the same session/path/tool/action are auto-approved for the lifetime of the service. Signals the pending request.
    - `Grant(permission PermissionRequest)`: Grants a one-time permission by signaling the pending request.
    - `Deny(permission PermissionRequest)`: Denies the permission by signaling the pending request.
    - `Request(opts CreatePermissionRequest) bool`: The main method called by tools to request permission.
        - If the session is in `autoApproveSessions`, returns `true`.
        - Normalizes the `Path` (takes `filepath.Dir` if it's a file path).
        - Checks if an identical persistent permission already exists in `sessionPermissions`; if so, returns `true`.
        - Otherwise, creates a new `PermissionRequest`, stores a response channel in `pendingRequests`, publishes a `pubsub.CreatedEvent` (which the TUI would listen for), and blocks waiting for a boolean response on the channel.
        - Returns the boolean response (true for grant, false for deny).
    - `AutoApproveSession(sessionID string)`: Adds a `sessionID` to the `autoApproveSessions` list.

### Functions
- `NewPermissionService() Service`: Constructor for the `permissionService`. Initializes the broker and `sessionPermissions` slice.

## Important Variables/Constants
- `ErrorPermissionDenied`: A standard error for tools to use.

## Usage Examples

How a tool (e.g., the "bash" tool) would request permission:
```go
// import "github.com/opencode-ai/opencode/internal/permission"
// import "github.com/opencode-ai/opencode/internal/config"

// var permService permission.Service // Assume initialized

// func (b *bashTool) Run(ctx context.Context, call ToolCall) (ToolResponse, error) {
//     // ... parse params ...
//     sessionID, _ := tools.GetContextValues(ctx)

//     if !isSafeReadOnlyCommand(params.Command) {
//         approved := b.permissions.Request(
//             permission.CreatePermissionRequest{
//                 SessionID:   sessionID,
//                 Path:        config.WorkingDirectory(), // Or a more specific path
//                 ToolName:    BashToolName,
//                 Action:      "execute",
//                 Description: fmt.Sprintf("Execute command: %s", params.Command),
//                 Params:      params, // The tool's parameters
//             },
//         )
//         if !approved {
//             return ToolResponse{}, permission.ErrorPermissionDenied
//         }
//     }
//     // ... execute command ...
// }
```

The TUI would subscribe to `permissionService.Subscribe()` and, upon receiving a `PermissionRequest` event, would display a dialog to the user. The user's response (grant/deny/grant persistently) would then call the corresponding `Grant()`, `Deny()`, or `GrantPersistant()` methods on the `permissionService` instance, which unblocks the original `Request()` call in the tool.

For non-interactive sessions (e.g., CLI prompt execution), `AutoApproveSession()` would be called for that session ID.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.WorkingDirectory()`.
    - `github.com/opencode-ai/opencode/internal/pubsub`: For `pubsub.Broker`.
- **External Libraries:**
    - `github.com/google/uuid`: For generating unique IDs for `PermissionRequest`s.
    - `errors`, `path/filepath`, `slices` (Go 1.21+), `sync`: Standard Go libraries.
- **Interactions:**
    - This service acts as a gatekeeper for potentially sensitive tool operations.
    - It decouples tools from the UI/user interaction layer needed for approval. Tools make a synchronous `Request()` call.
    - The service uses a pub/sub mechanism to send permission requests to listeners (typically the TUI).
    - The `pendingRequests` map and channels are used to block the `Request()` call until the user (or an automated process) responds via `Grant()`, `Deny()`, or `GrantPersistant()`.
    - `sessionPermissions` provides a way to "remember" granted permissions for the duration of the application's run, reducing repeated prompts for the same action by the same tool in the same context.
    - `autoApproveSessions` allows bypassing prompts entirely for certain sessions.

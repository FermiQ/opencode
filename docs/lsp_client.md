# client.go (in internal/lsp)

## Overview

The `client.go` file, part of the `internal/lsp` package, implements a Language Server Protocol (LSP) client. This client is responsible for launching, communicating with, and managing the lifecycle of an external LSP server process. It handles sending requests (like `initialize`, `textDocument/didOpen`), notifications, and processing responses and notifications from the server (like `textDocument/publishDiagnostics`). It also manages a cache of diagnostics and tracks open files.

## Key Components

### Structs
- `Client`: The main struct representing an LSP client instance.
    - `Cmd (*exec.Cmd)`: The command used to run the LSP server process.
    - `stdin (io.WriteCloser)`: Pipe to the LSP server's standard input.
    - `stdout (*bufio.Reader)`: Buffered reader for the LSP server's standard output.
    - `stderr (io.ReadCloser)`: Pipe for the LSP server's standard error.
    - `nextID (atomic.Int32)`: Atomically incremented counter for generating unique JSON-RPC request IDs.
    - `handlers (map[int32]chan *Message)`: Maps request IDs to channels, used to route responses back to the correct requester.
    - `handlersMu (sync.RWMutex)`: Mutex for `handlers` map.
    - `serverRequestHandlers (map[string]ServerRequestHandler)`: Maps LSP method names to handlers for requests initiated by the server (e.g., `workspace/applyEdit`).
    - `serverHandlersMu (sync.RWMutex)`: Mutex for `serverRequestHandlers`.
    - `notificationHandlers (map[string]NotificationHandler)`: Maps LSP method names to handlers for notifications from the server (e.g., `textDocument/publishDiagnostics`).
    - `notificationMu (sync.RWMutex)`: Mutex for `notificationHandlers`.
    - `diagnostics (map[protocol.DocumentUri][]protocol.Diagnostic)`: Cache for diagnostics received from the server, keyed by document URI.
    - `diagnosticsMu (sync.RWMutex)`: Mutex for `diagnostics` cache.
    - `openFiles (map[string]*OpenFileInfo)`: Tracks files currently considered open by this LSP client, keyed by file URI.
    - `openFilesMu (sync.RWMutex)`: Mutex for `openFiles`.
    - `serverState (atomic.Value)`: Stores the current `ServerState` (Starting, Ready, Error).
- `OpenFileInfo`: Stores information about an open file.
    - `Version (int32)`: The version number of the open file, incremented on changes.
    - `URI (protocol.DocumentUri)`: The URI of the open file.

### Types
- `ServerState (int)`: Enum for server readiness state (`StateStarting`, `StateReady`, `StateError`).
- `ServerType (int)`: Enum to categorize LSP servers (`ServerTypeUnknown`, `Go`, `TypeScript`, `Rust`, `Python`, `Generic`).

### Core Functions
- `NewClient(ctx context.Context, command string, args ...string) (*Client, error)`: Constructor for `Client`.
    - Creates and starts the LSP server process using `exec.CommandContext`.
    - Sets up stdin, stdout, and stderr pipes.
    - Initializes maps for handlers, diagnostics, and open files.
    - Starts a goroutine to read and process stderr from the LSP server.
    - Starts the main message handling loop (`client.handleMessages()`) in a goroutine.
- `(c *Client) RegisterNotificationHandler(method string, handler NotificationHandler)`: Registers a handler for server-sent notifications.
- `(c *Client) RegisterServerRequestHandler(method string, handler ServerRequestHandler)`: Registers a handler for server-initiated requests.
- `(c *Client) InitializeLSPClient(ctx context.Context, workspaceDir string) (*protocol.InitializeResult, error)`: Sends the `initialize` request to the LSP server with client capabilities and workspace information. Registers default handlers for common server requests/notifications (e.g., `workspace/applyEdit`, `textDocument/publishDiagnostics` which calls `HandleDiagnostics`). Sends `initialized` notification.
- `(c *Client) Close() error`: Shuts down the LSP client and server. Closes open files, closes stdin to signal server, waits for process exit with timeout, and kills if necessary.
- `(c *Client) GetServerState() ServerState`, `(c *Client) SetServerState(state ServerState)`: Get/set server readiness state.
- `(c *Client) WaitForServerReady(ctx context.Context) error`: Polls the server to check if it's ready to accept requests. Uses `pingServerByType` which employs different strategies based on `detectServerType()`. For TypeScript, it might try opening key config files (`openKeyConfigFiles`).
- `(c *Client) detectServerType() ServerType`: Heuristically determines the type of LSP server based on its command path.
- `(c *Client) openKeyConfigFiles(ctx context.Context)`: Opens common configuration files (e.g., `tsconfig.json`, `go.mod`) to help some LSP servers initialize correctly.
- `(c *Client) pingServerByType(ctx context.Context, serverType ServerType) error`: Sends a lightweight request suitable for the detected server type to check readiness.
- `(c *Client) pingTypeScriptServer(ctx context.Context)`: Specific ping logic for TypeScript servers, may try `textDocument/documentSymbol` on an open file.
- `(c *Client) openTypeScriptFiles(ctx context.Context, workDir string)`: Helper to find and open a few TypeScript files to aid server initialization.
- `(c *Client) pingWithWorkspaceSymbol(ctx context.Context)`, `(c *Client) pingWithServerCapabilities(ctx context.Context)`: Generic ping methods.
- `(c *Client) OpenFile(ctx context.Context, filepath string) error`: Sends `textDocument/didOpen` notification to the server.
- `(c *Client) NotifyChange(ctx context.Context, filepath string) error`: Sends `textDocument/didChange` notification.
- `(c *Client) CloseFile(ctx context.Context, filepath string) error`: Sends `textDocument/didClose` notification.
- `(c *Client) IsFileOpen(filepath string) bool`: Checks if a file is tracked as open.
- `(c *Client) CloseAllFiles(ctx context.Context)`: Closes all tracked open files.
- `(c *Client) GetFileDiagnostics(uri protocol.DocumentUri) []protocol.Diagnostic`: Gets cached diagnostics for a URI.
- `(c *Client) GetDiagnostics() map[protocol.DocumentUri][]protocol.Diagnostic`: Gets all cached diagnostics.
- `(c *Client) OpenFileOnDemand(ctx context.Context, filepath string) error`: Opens a file if not already open.
- `(c *Client) GetDiagnosticsForFile(ctx context.Context, filepath string) ([]protocol.Diagnostic, error)`: Ensures a file is open and returns its diagnostics, waiting briefly if just opened.
- `(c *Client) ClearDiagnosticsForURI(uri protocol.DocumentUri)`: Clears cached diagnostics for a URI.
- Note: The file also implicitly relies on `handleMessages` (likely in `protocol.go` or `handlers.go`) for the main message dispatch loop, and `Call`/`Notify` methods (likely in `protocol.go`) for sending requests/notifications.

## Important Variables/Constants
- `StateStarting`, `StateReady`, `StateError`: Define LSP server states.
- `ServerType*` constants: Categorize LSP servers.

## Usage Examples

Creating and initializing an LSP client (typically in `internal/app/app.go` or `internal/app/lsp.go`):
```go
// import "github.com/opencode-ai/opencode/internal/lsp"
// import "github.com/opencode-ai/opencode/internal/config"

// ctx := context.Background()
// lspServerCommand := "gopls"
// lspClient, err := lsp.NewClient(ctx, lspServerCommand)
// if err != nil { /* handle error */ }

// workspaceDir := config.WorkingDirectory()
// _, err = lspClient.InitializeLSPClient(ctx, workspaceDir)
// if err != nil { /* handle error */ }

// err = lspClient.WaitForServerReady(ctx)
// if err != nil { /* handle error, server might not be functional */ }

// Now lspClient can be used by tools (e.g., ViewTool, EditTool, DiagnosticsTool)
// tools.NewViewTool(map[string]*lsp.Client{"gopls": lspClient})
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.Get()`, `config.WorkingDirectory()`, `DebugLSP`.
    - `github.com/opencode-ai/opencode/internal/logging`: For logging.
    - `github.com/opencode-ai/opencode/internal/lsp/protocol`: For all LSP message structures and constants (e.g., `protocol.InitializeParams`, `protocol.Message`).
    - Relies on handler functions like `HandleApplyEdit`, `HandleDiagnostics` (defined in `handlers.go`).
    - Relies on `Message` struct and JSON-RPC processing logic (likely in `protocol.go`).
- **External Libraries:**
    - `bufio`, `context`, `encoding/json`, `fmt`, `io`, `os`, `os/exec`, `path/filepath`, `strings`, `sync`, `sync/atomic`, `time`: Standard Go libraries.
- **Interactions:**
    - Manages the full lifecycle of an LSP server subprocess.
    - Implements the client side of the JSON-RPC communication defined by LSP.
    - Provides methods for other parts of OpenCode (especially tools) to interact with the LSP server (e.g., open files, get diagnostics, potentially request completions or definitions in the future).
    - Caches diagnostics received via `textDocument/publishDiagnostics`.
    - Attempts to robustly initialize and check the readiness of various types of LSP servers.

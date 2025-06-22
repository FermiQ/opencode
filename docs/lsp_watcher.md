# watcher.go (in internal/lsp/watcher)

## Overview

The `watcher.go` file, part of the `internal/lsp/watcher` package, defines the `WorkspaceWatcher`. This component is responsible for monitoring file system events (creations, writes, deletes, renames) within a specified workspace path. It uses the `fsnotify` library for low-level file watching. When relevant events occur, especially those matching patterns registered by an LSP server, the watcher notifies the `lsp.Client` via `textDocument/didChangeWatchedFiles`. It also includes sophisticated logic for pre-opening "high-priority" or common project files for specific LSP server types (like TypeScript, Go, Java) to aid their initialization and improve performance.

## Key Components

### Structs
- `WorkspaceWatcher`: Manages LSP file watching for a workspace.
    - `client (*lsp.Client)`: The LSP client instance to send notifications to.
    - `workspacePath (string)`: The root path of the workspace being watched.
    - `debounceTime (time.Duration)`: Duration for debouncing file events (default 300ms).
    - `debounceMap (map[string]*time.Timer)`: Maps event keys (URI + change type) to timers for debouncing.
    - `debounceMu (sync.Mutex)`: Mutex for `debounceMap`.
    - `registrations ([]protocol.FileSystemWatcher)`: Stores file watching patterns registered by the LSP server.
    - `registrationMu (sync.RWMutex)`: Mutex for `registrations`.

### Functions
- `NewWorkspaceWatcher(client *lsp.Client) *WorkspaceWatcher`: Constructor for `WorkspaceWatcher`.
- `(w *WorkspaceWatcher) AddRegistrations(ctx context.Context, id string, watchers []protocol.FileSystemWatcher)`:
    - Called when the LSP server registers file system watchers (via `client/registerCapability` handled in `lsp/handlers.go`).
    - Appends new `watchers` to `w.registrations`.
    - **File Pre-loading Logic**:
        - Determines the server type using `getServerNameFromContext()`.
        - If `shouldPreloadFiles()` returns true for the server type OR if the server sent no explicit file watchers:
            - It launches a goroutine to proactively open files.
            - First, `openHighPriorityFiles()` is called to open common project/config files specific to the server type (e.g., `tsconfig.json` for TypeScript, `go.mod` for Go).
            - Then, it walks the `workspacePath` (respecting `shouldExcludeDir`) and opens additional source files (up to a `maxFilesToOpen` limit, which varies by server type) using `openMatchingFile()`. This aims to provide the LSP server with enough context early on.
- `(w *WorkspaceWatcher) openHighPriorityFiles(ctx context.Context, serverName string) int`:
    - Identifies a list of high-priority glob patterns based on `serverName`.
    - Uses `doublestar.Glob` to find matching files in the workspace.
    - Opens these files using `w.client.OpenFile()`, respecting `shouldExcludeFile` and limiting the number opened per pattern.
- `(w *WorkspaceWatcher) WatchWorkspace(ctx context.Context, workspacePath string)`:
    - The main loop for the watcher, intended to run in a goroutine.
    - Stores `workspacePath` and adds itself to the context.
    - Registers a handler with `lsp.RegisterFileWatchHandler` to receive server-registered `FileSystemWatcher`s via `AddRegistrations`.
    - Initializes an `fsnotify.Watcher`.
    - Recursively walks `workspacePath` using `filepath.WalkDir` to add all non-excluded directories to the `fsnotify.Watcher`.
    - Enters a loop to process events from `fsnotify.Watcher.Events` and `watcher.Errors`:
        - If a new directory is created (and not excluded), adds it to the `fsnotify.Watcher`.
        - If a new file is created (and not excluded), calls `w.openMatchingFile()` to potentially open it if it matches server-registered patterns or preloading criteria.
        - For `Write`, `Create` (file), `Remove`, `Rename` events on paths that `w.isPathWatched()` determines are relevant:
            - For `Write` and `Create`, calls `w.debounceHandleFileEvent()`.
            - For `Remove` and `Rename` (treated as delete then potential create), calls `w.handleFileEvent()` directly or via debounce.
- `(w *WorkspaceWatcher) isPathWatched(path string) (bool, protocol.WatchKind)`:
    - Checks if a given `path` matches any of the `FileSystemWatcher` patterns registered by the LSP server.
    - If no registrations exist, it defaults to watching all changes (create, change, delete).
    - Returns `true` and the relevant `WatchKind` if a match is found.
- `(w *WorkspaceWatcher) matchesPattern(path string, pattern protocol.GlobPattern) bool`:
    - Converts the `protocol.GlobPattern` (which can be a string or `protocol.RelativePattern`) into a pattern string and base path using `pattern.AsPattern()`.
    - Implements matching logic using `matchesGlob()`.
- `matchesGlob(pattern, path string) bool`, `matchesSimpleGlob(pattern, path string) bool`: Unexported helpers for glob matching, with some custom logic for `**` and `{}` alternatives. (Note: `matchesSimpleGlob` seems to be more robust than `filepath.Match` for certain LSP glob patterns).
- `(w *WorkspaceWatcher) debounceHandleFileEvent(ctx context.Context, uri string, changeType protocol.FileChangeType)`: Debounces file events to avoid sending rapid-fire notifications for a sequence of quick changes to the same file.
- `(w *WorkspaceWatcher) handleFileEvent(ctx context.Context, uri string, changeType protocol.FileChangeType)`:
    - If the event is `Changed` and the file is open in the `lsp.Client`, it sends a `textDocument/didChange` notification.
    - Otherwise (or for `Created`, `Deleted`), it sends a `workspace/didChangeWatchedFiles` notification via `notifyFileEvent()`.
    - If a file is deleted, it also calls `w.client.ClearDiagnosticsForURI()`.
- `(w *WorkspaceWatcher) notifyFileEvent(ctx context.Context, uri string, changeType protocol.FileChangeType) error`: Sends `workspace/didChangeWatchedFiles` to the LSP client.
- `getServerNameFromContext(ctx context.Context) string`: Utility to guess the LSP server name based on context value or client command path.
- `shouldPreloadFiles(serverName string) bool`: Determines if files should be preloaded for a given server type.
- `shouldExcludeDir(dirPath string) bool`, `shouldExcludeFile(filePath string) bool`: Helpers to check against predefined lists of excluded directory names, file extensions, and large binary files.
- `(w *WorkspaceWatcher) openMatchingFile(ctx context.Context, path string)`: Opens a file if it's not excluded and matches server-registered patterns or specific preloading criteria for the detected server type.
- `isHighPriorityFile(path string, serverName string) bool`: Checks if a file matches predefined high-priority patterns for a given server type.

### Package Variables
- `excludedDirNames`, `excludedFileExtensions`, `largeBinaryExtensions`: Maps defining common patterns to exclude.
- `maxFileSize`: Limit for opening files (5MB).

## Important Variables/Constants
- `excludedDirNames`, `excludedFileExtensions`, `largeBinaryExtensions`, `maxFileSize`: Define filtering rules for watched/opened files.

## Usage Examples

The `WorkspaceWatcher` is typically created and its `WatchWorkspace` method is run in a goroutine when an `lsp.Client` is initialized and ready.

```go
// Conceptual usage in internal/app/lsp.go or similar:
// import "github.com/opencode-ai/opencode/internal/lsp/watcher"

// lspClient := /* ... initialized lsp.Client ... */
// workspacePath := config.WorkingDirectory()

// wsWatcher := watcher.NewWorkspaceWatcher(lspClient)
// go wsWatcher.WatchWorkspace(context.Background(), workspacePath)
```
The watcher then runs in the background. When the LSP server registers file watching capabilities via `client/registerCapability`, the `lsp.HandleRegisterCapability` handler will call `lsp.RegisterFileWatchHandler`, which in turn will call `wsWatcher.AddRegistrations`.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.Get()`, `DebugLSP`.
    - `github.com/opencode-ai/opencode/internal/logging`: For logging.
    - `github.com/opencode-ai/opencode/internal/lsp`: For `lsp.Client` and `lsp.RegisterFileWatchHandler`.
    - `github.com/opencode-ai/opencode/internal/lsp/protocol`: For LSP types like `FileSystemWatcher`, `GlobPattern`, `FileChangeType`.
- **External Libraries:**
    - `github.com/bmatcuk/doublestar/v4`: For `doublestar.Glob` used in `openHighPriorityFiles`.
    - `github.com/fsnotify/fsnotify`: For actual file system event monitoring.
    - Standard Go libraries: `context`, `fmt`, `os`, `path/filepath`, `strings`, `sync`, `time`.
- **Interactions:**
    - Closely tied to the `lsp.Client` for sending notifications (`workspace/didChangeWatchedFiles`, `textDocument/didChange`) and opening/managing files.
    - Reacts to server-sent `client/registerCapability` requests (for `workspace/didChangeWatchedFiles`) by updating its watch patterns.
    - Implements a debouncing mechanism to manage floods of file events.
    - Contains heuristics (`shouldPreloadFiles`, `openHighPriorityFiles`, `openMatchingFile`) to proactively open files for certain LSP servers to improve their startup performance and accuracy. This logic involves detecting server type and knowing common project file patterns for different languages/ecosystems.
    - Filters out commonly ignored files and directories to reduce noise and resource usage.

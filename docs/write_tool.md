# write.go (in internal/llm/tools)

## Overview

The `write.go` file, part of the `internal/llm/tools` package, implements the "write" tool. This tool enables an AI agent to create new files or completely overwrite existing files with specified content. It includes safety checks, such as verifying if a file has been modified since it was last read by the agent, and integrates with permission handling, file history recording, and LSP diagnostics.

## Key Components

### Structs
- `WriteParams`: Defines the JSON parameters for the write tool.
    - `FilePath (string)`: The absolute or relative path to the file to be written.
    - `Content (string)`: The full content to write to the file.
- `WritePermissionsParams`: Struct used for permission requests, containing `FilePath` and a `Diff` of the proposed change (or full content for new files).
- `writeTool`: Implements the `BaseTool` interface.
    - `lspClients (map[string]*lsp.Client)`: Map of active LSP clients for fetching diagnostics after writing.
    - `permissions (permission.Service)`: Service for handling user permissions.
    - `files (history.Service)`: Service for recording file history/versions.
- `WriteResponseMetadata`: Struct to hold metadata about the write operation.
    - `Diff (string)`: The unified diff of the changes made (or full content if it's a new file).
    - `Additions (int)`: Number of lines added.
    - `Removals (int)`: Number of lines removed (relevant if overwriting).

### Constants
- `WriteToolName ("write")`: The registered name of the tool.
- `writeDescription (string)`: A detailed description for the LLM on how and when to use this tool. It covers:
    - Purpose: Creating new files or updating existing ones by overwriting.
    - Usage: Provide file path and content. Parent directories are created if needed.
    - Features: Creates/overwrites, creates parent dirs, checks for modification since last read, avoids no-op writes.
    - Limitations: Must read a file before overwriting, cannot append (always rewrites).
    - Tips: Use View tool first, LS to verify location, combine with Glob/Grep, add descriptive comments (though this is general coding advice, not specific to the tool's mechanics).

### Functions
- `NewWriteTool(lspClients map[string]*lsp.Client, permissions permission.Service, files history.Service) BaseTool`: Constructor for `writeTool`.
- `(w *writeTool) Info() ToolInfo`: Returns metadata about the tool.
- `(w *writeTool) Run(ctx context.Context, call ToolCall) (ToolResponse, error)`: The core logic when the write tool is invoked.
    1.  Parses `call.Input` into `WriteParams`. Validates `FilePath` and `Content` are provided.
    2.  Ensures `FilePath` is absolute.
    3.  **Safety Checks (if file exists)**:
        - If the file exists and is a directory, returns an error.
        - Checks if the file was modified since last read using `getLastReadTime()`. If so, returns an error prompting the user to re-read.
        - Reads the existing content. If it's identical to `params.Content`, returns an error indicating no changes are needed.
    4.  If the file doesn't exist, or passes the above checks, it proceeds.
    5.  Creates parent directories for `FilePath` if they don't exist.
    6.  Retrieves `sessionID` and `messageID` from context.
    7.  Generates a diff between `oldContent` (empty if new file) and `params.Content`.
    8.  Requests permission via `w.permissions.Request()` for the "write" action.
    9.  If permission is granted, writes `params.Content` to `filePath` using `os.WriteFile()`.
    10. **History Update**:
        - If the file is new or its content changed from the last known version in history, creates a new version in `w.files` service.
    11. Records file write and read times using `recordFileWrite()` and `recordFileRead()`.
    12. Calls `waitForLspDiagnostics()` and then `getDiagnostics()` to fetch and include diagnostics for the written file.
    13. Formats a success message, includes it in `<result>` tags, appends diagnostics, and returns it with `WriteResponseMetadata`.

## Important Variables/Constants
- `WriteToolName`: The registered name.
- `writeDescription`: Key instructions for the LLM.

## Usage Examples

This tool is invoked by an LLM.

LLM wants to create a new file `src/app.js` with some JavaScript content:
```json
{
  "type": "tool_use",
  "id": "tool_write_1",
  "name": "write",
  "input": {
    "file_path": "src/app.js",
    "content": "console.log('Hello, OpenCode!');"
  }
}
```
`writeTool.Run` would:
1. Ensure `src/` directory exists (create if not).
2. Request permission to create `src/app.js`.
3. If approved, write the content to the file.
4. Record history, update access times.
5. Fetch diagnostics for `src/app.js`.
6. Return a success message, diff metadata (showing all lines as added), and diagnostics.

LLM wants to overwrite `config.txt`:
```json
{
  "type": "tool_use",
  "id": "tool_write_2",
  "name": "write",
  "input": {
    "file_path": "/abs/path/to/config.txt",
    "content": "NEW_SETTING=true"
  }
}
```
(Assuming `/abs/path/to/config.txt` was read previously and hasn't changed since).
`writeTool.Run` would:
1. Perform safety checks.
2. Request permission to write.
3. If approved, overwrite the file.
4. Update history, access times.
5. Fetch diagnostics.
6. Return success message, diff, and diagnostics.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.WorkingDirectory()`.
    - `github.com/opencode-ai/opencode/internal/diff`: For `diff.GenerateDiff()`.
    - `github.com/opencode-ai/opencode/internal/history`: For `history.Service` to log file versions.
    - `github.com/opencode-ai/opencode/internal/logging`: For debug logging.
    - `github.com/opencode-ai/opencode/internal/lsp`: For `lsp.Client` map.
    - `github.com/opencode-ai/opencode/internal/permission`: For `permission.Service`.
    - Relies on types from the parent `tools` package and utility functions like `getLastReadTime`, `recordFileWrite`, `recordFileRead`, `waitForLspDiagnostics`, `getDiagnostics`.
- **External Libraries:**
    - `encoding/json`, `fmt`, `os`, `path/filepath`, `strings`, `time`, `context`: Standard Go libraries.
- **Interactions:**
    - Directly modifies the filesystem by creating or overwriting files.
    - All write operations are subject to user permission.
    - Integrates with a file history system to version changes.
    - Checks for file staleness to prevent accidental overwrites of externally modified files.
    - Triggers LSP clients to provide diagnostics for the newly written/modified file.

# edit.go (in internal/llm/tools)

## Overview

The `edit.go` file, part of the `internal/llm/tools` package, defines the "edit" tool. This tool allows an AI agent to perform precise text replacements in files, create new files, or delete specific content from files. It emphasizes the need for unique identification of the text to be replaced (`old_string`) by requiring extensive context. The tool also interacts with the permission service for write operations and the history service to record file versions.

## Key Components

### Structs
- `EditParams`: Defines the JSON parameters for the edit tool.
    - `FilePath (string)`: Absolute path to the file.
    - `OldString (string)`: The exact text to be replaced.
    - `NewString (string)`: The text to replace `OldString` with.
- `EditPermissionsParams`: Struct used for permission requests, containing `FilePath` and a `Diff` of the proposed change.
- `EditResponseMetadata`: Struct to hold metadata about the edit operation.
    - `Diff (string)`: The unified diff of the changes made.
    - `Additions (int)`: Number of lines added.
    - `Removals (int)`: Number of lines removed.
- `editTool`: Implements the `BaseTool` interface.
    - `lspClients (map[string]*lsp.Client)`: Map of active LSP clients (used for triggering diagnostics after edit).
    - `permissions (permission.Service)`: Service for handling user permissions.
    - `files (history.Service)`: Service for recording file history/versions.

### Constants
- `EditToolName ("edit")`: The registered name of the tool.
- `editDescription (string)`: A detailed description for the LLM on how to use this tool. It stresses:
    - Using `FileRead` (ViewTool) first to understand context.
    - Special cases:
        - Create new file: `OldString` is empty.
        - Delete content: `NewString` is empty.
    - **Critical Requirements**:
        - **Uniqueness**: `OldString` must be unique and include 3-5 lines of context before and after the change point, matching whitespace and indentation exactly.
        - **Single Instance**: Only one instance can be changed per call. Multiple changes require multiple calls.
    - **Verification**: Advises checking for multiple instances of `OldString` and gathering unique context for each.
    - Warnings about failures if requirements are not met.
    - General advice on making idiomatic and correct code edits and using absolute paths.
    - Recommends sending multiple edits to the same file in a single message with multiple tool calls.

### Functions
- `NewEditTool(lspClients map[string]*lsp.Client, permissions permission.Service, files history.Service) BaseTool`: Constructor for `editTool`.
- `(e *editTool) Info() ToolInfo`: Returns metadata about the tool, including its name, the detailed `editDescription`, and parameter schema.
- `(e *editTool) Run(ctx context.Context, call ToolCall) (ToolResponse, error)`: The core logic when the edit tool is invoked.
    1.  Parses `call.Input` into `EditParams`.
    2.  Ensures `FilePath` is absolute.
    3.  Dispatches to helper methods based on `OldString` and `NewString`:
        - `createNewFile(...)` if `OldString` is empty.
        - `deleteContent(...)` if `NewString` is empty.
        - `replaceContent(...)` otherwise.
    4.  After a successful edit, calls `waitForLspDiagnostics` (from `diagnostics.go` in the same package) to allow LSPs to process changes.
    5.  Appends any diagnostics found using `getDiagnostics` to the tool's response, wrapped in `<result>` and diagnostic tags.
- `(e *editTool) createNewFile(ctx context.Context, filePath, content string) (ToolResponse, error)`:
    - Checks if the file already exists or if the path is a directory.
    - Creates parent directories if they don't exist.
    - Generates a diff for the new file creation.
    - Requests permission via `e.permissions.Request()`.
    - Writes the file using `os.WriteFile()`.
    - Creates an initial entry and a versioned entry in the file history using `e.files.Create()` and `e.files.CreateVersion()`.
    - Records file write and read times (using unexported `recordFileWrite` and `recordFileRead` from `tools.go`).
    - Returns a success message along with diff metadata.
- `(e *editTool) deleteContent(ctx context.Context, filePath, oldString string) (ToolResponse, error)`:
    - Validates file existence and checks if it was read recently (using unexported `getLastReadTime`).
    - Reads file content.
    - Finds `oldString`; returns an error if not found or if multiple instances exist.
    - Creates `newContent` by removing `oldString`.
    - Generates a diff.
    - Requests permission.
    - Writes the modified content back to the file.
    - Updates file history.
    - Records file write/read times.
    - Returns a success message with diff metadata.
- `(e *editTool) replaceContent(ctx context.Context, filePath, oldString, newString string) (ToolResponse, error)`:
    - Similar validation and file reading steps as `deleteContent`.
    - Finds `oldString`; returns an error if not found or if multiple instances exist.
    - Creates `newContent` by replacing `oldString` with `newString`.
    - Checks if `newContent` is identical to `oldContent`; if so, returns an error indicating no changes.
    - Generates a diff.
    - Requests permission.
    - Writes the modified content.
    - Updates file history.
    - Records file write/read times.
    - Returns a success message with diff metadata.

## Important Variables/Constants
- `EditToolName`: The registered name.
- `editDescription`: Critical for guiding the LLM on the precise and constrained usage of this tool.

## Usage Examples

This tool is invoked by an LLM.

LLM wants to replace "foo" with "bar" in `main.go` (assuming "foo" with its context is unique):
```json
{
  "type": "tool_use",
  "id": "tool_edit_1",
  "name": "edit",
  "input": {
    "file_path": "/abs/path/to/main.go",
    "old_string": "context_line_1\ncontext_line_2\n  foo_to_replace\ncontext_line_3\ncontext_line_4",
    "new_string": "context_line_1\ncontext_line_2\n  bar_was_inserted\ncontext_line_3\ncontext_line_4"
  }
}
```
`editTool.Run` would:
1. Call `replaceContent`.
2. Perform safety checks (file exists, was read, `old_string` is unique).
3. Request permission with a diff.
4. If approved, write the change, update history.
5. Wait for LSP diagnostics.
6. Return a success message, diff metadata, and any new diagnostics.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.WorkingDirectory()`.
    - `github.com/opencode-ai/opencode/internal/diff`: For `diff.GenerateDiff()`.
    - `github.com/opencode-ai/opencode/internal/history`: For `history.Service` to log file versions.
    - `github.com/opencode-ai/opencode/internal/logging`: For debug logging.
    - `github.com/opencode-ai/opencode/internal/lsp`: For `lsp.Client` map.
    - `github.com/opencode-ai/opencode/internal/permission`: For `permission.Service`.
    - Relies on types from the parent `tools` package and utility functions like `waitForLspDiagnostics` and `getDiagnostics` (also in `diagnostics.go`).
    - Uses unexported `recordFileWrite`, `recordFileRead`, `getLastReadTime` (likely from `tools.go` in the same package) for tracking file access.
- **External Libraries:**
    - `encoding/json`, `fmt`, `os`, `path/filepath`, `strings`, `time`, `context`: Standard Go libraries.
- **Interactions:**
    - Modifies files on the user's filesystem.
    - Requires explicit user permission for all modifications.
    - Records changes in a file history system.
    - Its effectiveness heavily depends on the LLM's ability to adhere to the strict contextual requirements for `old_string`.
    - Triggers LSP clients to re-evaluate diagnostics after an edit.

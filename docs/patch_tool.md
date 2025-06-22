# patch_tool.go (in internal/llm/tools)

## Overview

The `patch_tool.go` file (renamed for clarity, originally `patch.go` in this directory), part of the `internal/llm/tools` package, defines the "patch" tool. This tool allows an AI agent to apply a set of changes to multiple files in a single operation using a custom patch format. The actual parsing and application logic for this custom patch format is provided by the `internal/diff` package (specifically `internal/diff/patch.go`). This tool orchestrates the process, including file validation, permission requests, history updates, and LSP diagnostic checks.

## Key Components

### Structs
- `PatchParams`: Defines the JSON parameters for the patch tool.
    - `PatchText (string)`: The full text of the patch, conforming to the custom format defined in `internal/diff/patch.go`.
- `PatchResponseMetadata`: Struct for metadata about the patch operation.
    - `FilesChanged ([]string)`: List of file paths that were modified by the patch.
    - `Additions (int)`: Total number of lines added across all files.
    - `Removals (int)`: Total number of lines removed across all files.
- `patchTool`: Implements the `BaseTool` interface.
    - `lspClients (map[string]*lsp.Client)`: Map of active LSP clients for diagnostics.
    - `permissions (permission.Service)`: Service for handling user permissions.
    - `files (history.Service)`: Service for recording file history/versions.

### Constants
- `PatchToolName ("patch")`: The registered name of the tool.
- `patchDescription (string)`: A detailed description for the LLM on how to use this tool. It specifies the custom patch format (Begin Patch, Update File, Add File, Delete File, End Patch, context lines `@@`, change lines `+`/`-`). It emphasizes:
    - Using `FileRead` (ViewTool) first.
    - Verifying paths with `LS` tool.
    - **Critical Requirements**: Uniqueness of context, precision in matching whitespace/indentation, ensuring valid code, and using absolute paths.
    - States that all changes are applied atomically (though atomicity depends on the underlying `diff.ApplyCommit` and filesystem operations).

### Functions
- `NewPatchTool(lspClients map[string]*lsp.Client, permissions permission.Service, files history.Service) BaseTool`: Constructor for `patchTool`.
- `(p *patchTool) Info() ToolInfo`: Returns metadata about the tool.
- `(p *patchTool) Run(ctx context.Context, call ToolCall) (ToolResponse, error)`: The core logic when the patch tool is invoked.
    1.  Parses `call.Input` into `PatchParams`. Validates `PatchText` is provided.
    2.  **File Validation**:
        - Identifies files needed for update/delete using `diff.IdentifyFilesNeeded()`.
        - For each, checks if it was recently read (using `getLastReadTime`), exists, is not a directory, and hasn't been modified since last read.
        - Identifies files to be added using `diff.IdentifyFilesAdded()`.
        - For each, checks if it already exists.
    3.  **Load Files**: Loads content of all files needed for update/delete.
    4.  **Parse Patch**: Calls `diff.TextToPatch()` to parse `params.PatchText` against `currentFiles`. Checks fuzz level (allows up to 3).
    5.  **Convert to Commit**: Calls `diff.PatchToCommit()` to convert the parsed `diff.Patch` into a `diff.Commit` object.
    6.  **Permissions**: Iterates through each change in the `commit.Changes`:
        - Generates a diff for the specific change.
        - Requests permission from `p.permissions.Request()` for the action (create, update, delete) on the file/directory. If any permission is denied, the whole operation fails.
    7.  **Apply Commit**: Calls `diff.ApplyCommit()` to apply the changes to the filesystem. The write function provided to `ApplyCommit` ensures parent directories are created and uses `os.WriteFile`. The remove function uses `os.Remove`.
    8.  **Update History & Metadata**:
        - For each changed file:
            - Calculates additions/removals using `diff.GenerateDiff()`.
            - Updates file history using `p.files.Create()` or `p.files.CreateVersion()`.
            - Records file write/read times using `recordFileWrite()` and `recordFileRead()`.
    9.  **LSP Diagnostics**: For each changed file, calls `waitForLspDiagnostics()` and then collects diagnostics using `getDiagnostics()`.
    10. Formats a success message including change counts and any diagnostics.
    11. Returns the response with `PatchResponseMetadata`.

## Important Variables/Constants
- `PatchToolName`: The registered name.
- `patchDescription`: Critical for guiding the LLM on the custom patch format and usage constraints.

## Usage Examples

This tool is invoked by an LLM. The LLM must generate a patch string in the custom format.

LLM wants to apply changes to `file1.txt` and create `file2.txt`:
```json
{
  "type": "tool_use",
  "id": "tool_patch_1",
  "name": "patch",
  "input": {
    "patch_text": "*** Begin Patch\n*** Update File: /abs/path/to/file1.txt\n@@ old line context\n-old line\n+new line\n@@ more context\n*** Add File: /abs/path/to/file2.txt\n+This is a new file.\n+With two lines.\n*** End Patch"
  }
}
```
`patchTool.Run` would:
1. Validate `file1.txt` (exists, read recently, not modified).
2. Validate `file2.txt` (does not exist).
3. Load `file1.txt`.
4. Parse the patch text.
5. Request permission for updating `file1.txt` and creating `file2.txt`.
6. If approved, apply changes to `file1.txt` and create `file2.txt`.
7. Update history for both files.
8. Check LSP diagnostics for `file1.txt` and `file2.txt`.
9. Return a success message with change counts and diagnostics.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.WorkingDirectory()`.
    - `github.com/opencode-ai/opencode/internal/diff`: Critically relies on this package for parsing the custom patch format (`diff.TextToPatch`), converting to a commit structure (`diff.PatchToCommit`), applying changes (`diff.ApplyCommit`), and identifying files from patch text.
    - `github.com/opencode-ai/opencode/internal/history`: For `history.Service` to log file versions.
    - `github.com/opencode-ai/opencode/internal/logging`: For debug logging.
    - `github.com/opencode-ai/opencode/internal/lsp`: For `lsp.Client` map.
    - `github.com/opencode-ai/opencode/internal/permission`: For `permission.Service`.
    - Relies on types from the parent `tools` package and utility functions like `getLastReadTime`, `recordFileWrite`, `recordFileRead`, `waitForLspDiagnostics`, `getDiagnostics`.
- **External Libraries:**
    - `encoding/json`, `fmt`, `os`, `path/filepath`, `time`, `context`: Standard Go libraries.
- **Interactions:**
    - Provides a powerful multi-file modification capability to the agent using a specific text-based patch format.
    - All file operations (read, write, delete) are subject to prior validation (e.g., file staleness checks) and user permissions.
    - Integrates with file history and LSP diagnostics.
    - The success of this tool heavily depends on the LLM's ability to generate syntactically correct patches adhering to the custom format and its strict contextual requirements.

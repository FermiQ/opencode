# view.go (in internal/llm/tools)

## Overview

The `view.go` file, part of the `internal/llm/tools` package, implements the "view" tool (also referred to as "FileRead" in some descriptions). This tool enables an AI agent to read and display the contents of specified files. It includes features like line numbering, reading from an offset, limiting the number of lines read, truncating long lines, and providing suggestions for similarly named files if the requested file isn't found. It also checks for file size limits and identifies image files (though it doesn't display them).

## Key Components

### Structs
- `ViewParams`: Defines the JSON parameters for the view tool.
    - `FilePath (string)`: The path to the file to read.
    - `Offset (int)`: The 0-based line number to start reading from.
    - `Limit (int)`: The maximum number of lines to read (defaults to `DefaultReadLimit`).
- `viewTool`: Implements the `BaseTool` interface.
    - `lspClients (map[string]*lsp.Client)`: Map of active LSP clients, used to fetch diagnostics for the viewed file.
- `ViewResponseMetadata`: Struct for metadata about the view operation.
    - `FilePath (string)`: The path of the file that was viewed.
    - `Content (string)`: The actual content read from the file (before line numbering).
- `LineScanner`: An unexported wrapper around `bufio.Scanner` to facilitate line-by-line reading.

### Constants
- `ViewToolName ("view")`: The registered name of the tool.
- `MaxReadSize (250 * 1024)`: 250KB, the maximum file size that can be read.
- `DefaultReadLimit (2000)`: Default number of lines to read if `Limit` is not specified.
- `MaxLineLength (2000)`: Maximum length of a single line before it's truncated with "...".
- `viewDescription (string)`: A detailed description for the LLM on how and when to use this tool. It covers:
    - Purpose: Examining code, logs, text data.
    - Usage: Provide file path, optional offset, and limit.
    - Features: Line numbers, offset reading, large file handling via line limit, long line truncation, file suggestions on not found.
    - Limitations: Max file size, default line limit, line length truncation, no binary/image display (though images are identified).
    - Tips: Use with Glob or Grep, use offset for large files.

### Functions
- `NewViewTool(lspClients map[string]*lsp.Client) BaseTool`: Constructor for `viewTool`.
- `(v *viewTool) Info() ToolInfo`: Returns metadata about the tool.
- `(v *viewTool) Run(ctx context.Context, call ToolCall) (ToolResponse, error)`: The core logic when the view tool is invoked.
    1.  Parses `call.Input` into `ViewParams`.
    2.  Validates `FilePath` is provided and makes it absolute if relative.
    3.  Checks file existence and type (not a directory). If not found, attempts to suggest similar file names.
    4.  Checks file size against `MaxReadSize`.
    5.  Sets default `Limit` if not provided.
    6.  Checks if the file is an image using `isImageFile()`; if so, returns an error message.
    7.  Calls `readTextFile()` to read the specified portion of the file.
    8.  Notifies LSP clients about the file being opened using `notifyLspOpenFile()` (from `diagnostics.go`).
    9.  Formats the content with line numbers using `addLineNumbers()`.
    10. Appends a truncation note if the file has more lines than read.
    11. Wraps the content in `<file>` tags.
    12. Appends diagnostics for the file using `getDiagnostics()` (from `diagnostics.go`).
    13. Records that the file was read using `recordFileRead()` (from `file.go` in the same package).
    14. Returns the formatted content and diagnostics, along with `ViewResponseMetadata`.
- `addLineNumbers(content string, startLine int) string`: Prepends line numbers (padded to 6 digits, followed by '|') to each line of the content.
- `readTextFile(filePath string, offset, limit int) (string, int, error)`:
    - Opens the specified file.
    - Uses a `LineScanner` to read lines.
    - Skips lines up to `offset`.
    - Reads up to `limit` lines, truncating individual lines longer than `MaxLineLength`.
    - Continues scanning to get the total `lineCount` in the file.
    - Returns the joined lines of the requested segment, the total line count of the file, and any error.
- `isImageFile(filePath string) (bool, string)`: Checks the file extension against a list of common image extensions (jpg, png, gif, bmp, svg, webp) and returns if it's an image and its type.
- `NewLineScanner(r io.Reader) *LineScanner`, `(s *LineScanner) Scan() bool`, `(s *LineScanner) Text() string`, `(s *LineScanner) Err() error`: Wrapper methods around `bufio.Scanner`.

## Important Variables/Constants
- `ViewToolName`, `MaxReadSize`, `DefaultReadLimit`, `MaxLineLength`: Define the tool's behavior and constraints.
- `viewDescription`: Critical for LLM guidance.

## Usage Examples

LLM wants to view the first 50 lines of `src/main.go`:
```json
{
  "type": "tool_use",
  "id": "tool_view_1",
  "name": "view",
  "input": {
    "file_path": "src/main.go",
    "limit": 50
  }
}
```
`viewTool.Run` would:
1. Read lines 0-49 from `src/main.go`.
2. Prepend line numbers (1-50).
3. Fetch diagnostics for `src/main.go`.
4. Return the numbered lines and diagnostics to the LLM, wrapped in `<file>` tags.

LLM wants to view lines 100-110 of `README.md`:
```json
{
  "type": "tool_use",
  "id": "tool_view_2",
  "name": "view",
  "input": {
    "file_path": "README.md",
    "offset": 99,
    "limit": 11
  }
}
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.WorkingDirectory()`.
    - `github.com/opencode-ai/opencode/internal/lsp`: For `lsp.Client` map.
    - Relies on types from the parent `tools` package and utility functions like `notifyLspOpenFile`, `getDiagnostics` (from `diagnostics.go`), and `recordFileRead` (from `file.go` in the same package).
- **External Libraries:**
    - `bufio`, `context`, `encoding/json`, `fmt`, `io`, `os`, `path/filepath`, `strings`: Standard Go libraries.
- **Interactions:**
    - Reads files from the user's filesystem.
    - Interacts with LSP clients to provide diagnostics alongside file content.
    - Uses an internal file access tracking mechanism (`recordFileRead`).
    - Provides suggestions for typos in file names.
    - Implements various limits (file size, lines read, line length) to manage resource usage and output size.

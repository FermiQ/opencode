# grep.go (in internal/llm/tools)

## Overview

The `grep.go` file, part of the `internal/llm/tools` package, defines the "grep" tool. This tool enables an AI agent to search for text or regular expression patterns within the contents of files. It prioritizes using `rg` (ripgrep) if available for performance, falling back to a Go-native regex search across files. Results include the matching file path, line number, and the line text, sorted by modification time (newest first) and limited in number.

## Key Components

### Structs
- `GrepParams`: Defines the JSON parameters for the grep tool.
    - `Pattern (string)`: The regex pattern to search for.
    - `Path (string)`: The directory to search in (defaults to current working directory).
    - `Include (string)`: A glob pattern to filter which files are searched (e.g., `*.go`).
    - `LiteralText (bool)`: If true, the `Pattern` is treated as literal text, and special regex characters are escaped.
- `grepMatch`: An unexported struct to hold information about a single match.
    - `path (string)`: Path to the file containing the match.
    - `modTime (time.Time)`: Modification time of the file.
    - `lineNum (int)`: Line number of the match.
    - `lineText (string)`: The actual text of the matching line.
- `GrepResponseMetadata`: Struct for metadata about the grep operation.
    - `NumberOfMatches (int)`: Total number of matches found (up to the limit).
    - `Truncated (bool)`: Indicates if the results were truncated.
- `grepTool`: Implements the `BaseTool` interface (empty struct).

### Constants
- `GrepToolName ("grep")`: The registered name of the tool.
- `grepDescription (string)`: A detailed description for the LLM on how to use the tool, including:
    - When to use it (finding files with specific text/patterns).
    - How to use it (regex pattern, optional `literal_text`, path, include filter).
    - Regex syntax notes and include pattern examples.
    - Limitations (results limited to 100 files, skips large binaries and hidden files).
    - Tips (combine with Glob, consider Agent tool for iteration, check for truncation, use `literal_text` for special characters).

### Functions
- `NewGrepTool() BaseTool`: Constructor for `grepTool`.
- `(g *grepTool) Info() ToolInfo`: Returns metadata about the tool.
- `escapeRegexPattern(pattern string) string`: Escapes special regex characters in a string to treat it as a literal search term.
- `(g *grepTool) Run(ctx context.Context, call ToolCall) (ToolResponse, error)`: The core logic when the grep tool is invoked.
    1.  Parses `call.Input` into `GrepParams`.
    2.  Validates `Pattern` is provided.
    3.  Escapes `Pattern` if `LiteralText` is true.
    4.  Sets `searchPath` to current working directory if not provided.
    5.  Calls `searchFiles()` to perform the content search, limiting results to 100 matches.
    6.  Formats the output:
        - "No files found" if no matches.
        - Otherwise, lists matches grouped by file path, showing line number and text.
        - Adds a truncation message if applicable.
    7.  Returns the output string with `GrepResponseMetadata`.
- `searchFiles(pattern, rootPath, include string, limit int) ([]grepMatch, bool, error)`:
    - Attempts to use `ripgrep` first via `searchWithRipgrep()`.
    - If `rg` fails or is not found, it falls back to `searchFilesWithRegex()`.
    - Sorts the collected `grepMatch` items by `modTime` (newest first).
    - Truncates results if they exceed `limit`.
    - Returns the matches, truncation status, and any error.
- `searchWithRipgrep(pattern, path, include string) ([]grepMatch, error)`:
    - Checks if `rg` is available.
    - Constructs and executes an `rg` command (e.g., `rg -n pattern --glob include path`).
    - Parses `rg`'s output (format: `file:line:content`) into `grepMatch` structs.
    - Retrieves file modification times for each match.
    - Returns the list of matches.
- `searchFilesWithRegex(pattern, rootPath, include string) ([]grepMatch, error)`:
    - Compiles the search `pattern` into a Go `regexp.Regexp`.
    - If `include` is provided, converts the glob `include` pattern to a regex using `globToRegex()` and compiles it.
    - Walks the `rootPath` using `filepath.Walk()`:
        - Skips directories and hidden files (via `fileutil.SkipHidden()`).
        - If an `includePattern` exists, skips files not matching it.
        - Calls `fileContainsPattern()` for each eligible file.
        - If a match is found, creates a `grepMatch` and adds it to the results.
        - Stops walking if 200 matches are found (an internal soft limit before final truncation).
    - Returns the collected matches.
- `fileContainsPattern(filePath string, pattern *regexp.Regexp) (bool, int, string, error)`:
    - Opens and scans `filePath` line by line.
    - Returns `true`, line number, and line text upon the first match of `pattern`.
- `globToRegex(glob string) string`: A simple converter from glob-like patterns (with `*`, `?`, `{,}`) to basic regex syntax.

## Important Variables/Constants
- `GrepToolName`: The registered name.
- `grepDescription`: Crucial for LLM guidance.

## Usage Examples

LLM wants to find all occurrences of "TODO:" in Go files within the "pkg" directory:
```json
{
  "type": "tool_use",
  "id": "tool_grep_1",
  "name": "grep",
  "input": {
    "pattern": "TODO:",
    "path": "pkg",
    "include": "*.go",
    "literal_text": true
  }
}
```
`grepTool.Run` would:
1. Call `searchFiles("TODO:", "pkg", "*.go", 100)`.
2. `searchFiles` would try `rg`. If `rg -n "TODO:" --glob "*.go" pkg` succeeds, its output is processed.
3. Otherwise, it falls back to Go regex search, walking "pkg", filtering by `*.go`, and scanning files for "TODO:".
4. Results are formatted (e.g., `pkg/feature/file.go:\n  Line 42: // TODO: Fix this later`) and returned.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.WorkingDirectory()`.
    - `github.com/opencode-ai/opencode/internal/fileutil`: For `fileutil.SkipHidden()`.
    - Relies on types from the parent `tools` package.
- **External Libraries:**
    - `bufio`, `context`, `encoding/json`, `fmt`, `os`, `os/exec`, `path/filepath`, `regexp`, `sort`, `strconv`, `strings`, `time`: Standard Go libraries.
- **Interactions:**
    - Provides content searching capabilities to the agent.
    - Prioritizes `rg` for performance. The fallback Go regex search can be slower on large codebases.
    - Sorting by modification time helps surface recent relevant changes first.
    - Output formatting is designed to be human-readable and parsable by the LLM.

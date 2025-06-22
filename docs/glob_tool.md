# glob.go (in internal/llm/tools)

## Overview

The `glob.go` file, part of the `internal/llm/tools` package, implements the "glob" tool. This tool allows an AI agent to find files by name and pattern using glob expressions. It prioritizes using `rg` (ripgrep) for speed if available, and falls back to a Go-native doublestar implementation. Results are sorted by modification time (newest first by `fileutil.GlobWithDoublestar`, though `runRipgrep` sorts by path length after its execution) and are limited to a certain number of files.

## Key Components

### Structs
- `GlobParams`: Defines the JSON parameters for the glob tool.
    - `Pattern (string)`: The glob pattern to match files against (e.g., `*.go`, `src/**/*.js`).
    - `Path (string)`: The directory to start the search from. Defaults to the current working directory.
- `GlobResponseMetadata`: Struct to hold metadata about the glob operation.
    - `NumberOfFiles (int)`: The number of files found (up to the limit).
    - `Truncated (bool)`: Indicates if the results were truncated due to the limit.
- `globTool`: Implements the `BaseTool` interface. (It's an empty struct as the tool's logic is self-contained in its methods).

### Constants
- `GlobToolName ("glob")`: The registered name of the tool.
- `globDescription (string)`: A detailed description for the LLM on when and how to use this tool. It explains:
    - When to use it (finding files by name patterns/extensions).
    - How to use it (provide pattern, optional path).
    - Glob pattern syntax (`*`, `**`, `?`, `[...]`).
    - Common pattern examples.
    - Limitations (results limited to 100 files, doesn't search content, skips hidden files).
    - Tips (combine with Grep, consider Agent tool for iterative exploration, check for truncation).

### Functions
- `NewGlobTool() BaseTool`: Constructor for `globTool`.
- `(g *globTool) Info() ToolInfo`: Returns metadata about the tool, including its name, the detailed `globDescription`, and parameter schema.
- `(g *globTool) Run(ctx context.Context, call ToolCall) (ToolResponse, error)`: The core logic when the glob tool is invoked.
    1.  Parses `call.Input` into `GlobParams`.
    2.  Validates that `Pattern` is provided.
    3.  Sets `searchPath` to the current working directory if not provided in params.
    4.  Calls `globFiles()` to perform the search, limiting results to 100 files.
    5.  Formats the output:
        - "No files found" if no matches.
        - A newline-separated list of matched file paths.
        - A truncation message if `truncated` is true.
    6.  Returns the output string along with `GlobResponseMetadata`.
- `globFiles(pattern, searchPath string, limit int) ([]string, bool, error)`:
    - Attempts to use `ripgrep` (`rg`) first by calling `fileutil.GetRgCmd(pattern)`.
    - If `rg` is available and the command is constructed:
        - Sets `rg`'s working directory to `searchPath`.
        - Calls `runRipgrep()` to execute `rg` and process its output.
        - If successful, returns the matches from `rg` and a boolean indicating if truncation occurred based on the limit.
        - If `rg` fails, logs a warning and falls back to `fileutil.GlobWithDoublestar()`.
    - If `rg` is not available or failed, it directly calls `fileutil.GlobWithDoublestar(pattern, searchPath, limit)` as a fallback.
- `runRipgrep(cmd *exec.Cmd, searchRoot string, limit int) ([]string, error)`:
    - Executes the provided `rg` command (`cmd.CombinedOutput()`).
    - Handles `rg`'s exit code 1 (no matches found) as a non-error scenario (returns `nil, nil`).
    - Parses `rg`'s null-terminated output into a slice of file paths.
    - Ensures paths are absolute.
    - Filters out hidden files using `fileutil.SkipHidden()`.
    - **Sorts matches by path length (shortest first)**. This is different from `fileutil.GlobWithDoublestar` which sorts by modification time.
    - Truncates results if `limit` is positive and the number of matches exceeds it.
    - Returns the processed list of matches and any execution error.

## Important Variables/Constants
- `GlobToolName`: The registered name.
- `globDescription`: Provides crucial usage instructions to the LLM.

## Usage Examples

This tool is invoked by an LLM.

LLM wants to find all Python files in the `scripts` directory:
```json
{
  "type": "tool_use",
  "id": "tool_glob_1",
  "name": "glob",
  "input": {
    "pattern": "*.py",
    "path": "scripts"
  }
}
```
`globTool.Run` would:
1. Call `globFiles("*.py", "scripts", 100)`.
2. `globFiles` would try to use `rg`. If `rg --files --null --glob /*.py` (run in `scripts` dir) succeeds, its output is processed.
3. If `rg` fails or isn't available, `fileutil.GlobWithDoublestar("*.py", "scripts", 100)` is used.
4. The list of found Python files (up to 100, sorted appropriately by the chosen method) is returned to the LLM.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.WorkingDirectory()`.
    - `github.com/opencode-ai/opencode/internal/fileutil`: For `fileutil.GetRgCmd()` and `fileutil.GlobWithDoublestar()`, and `fileutil.SkipHidden()`.
    - `github.com/opencode-ai/opencode/internal/logging`: For warning if `rg` execution fails.
    - Relies on types from the parent `tools` package.
- **External Libraries:**
    - `encoding/json`, `fmt`, `os/exec`, `path/filepath`, `sort`, `strings`, `bytes`, `context`: Standard Go libraries.
- **Interactions:**
    - Provides file searching capabilities to the agent.
    - Prioritizes the external `rg` command for performance if available, otherwise uses a Go-native implementation. This means behavior (especially sorting order and exact pattern interpretation nuances) might differ slightly depending on `rg`'s presence.
    - The `runRipgrep` function sorts results by path length, while the fallback `fileutil.GlobWithDoublestar` sorts by modification time. This difference in sorting might be an important detail depending on how the LLM uses the results.
    - Always filters out hidden files as defined by `fileutil.SkipHidden`.

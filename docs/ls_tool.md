# ls.go (in internal/llm/tools)

## Overview

The `ls.go` file, part of the `internal/llm/tools` package, implements the "ls" tool. This tool allows an AI agent to list the contents of a directory, displaying files and subdirectories in a hierarchical tree structure. It includes features for ignoring specific patterns and limits the number of files listed to prevent overwhelming output.

## Key Components

### Structs
- `LSParams`: Defines the JSON parameters for the ls tool.
    - `Path (string)`: The directory path to list. Defaults to the current working directory if empty.
    - `Ignore ([]string)`: A list of glob patterns for files/directories to ignore from the listing.
- `TreeNode`: An unexported struct used to build the directory tree representation.
    - `Name (string)`: Name of the file or directory.
    - `Path (string)`: Full path to the file or directory.
    - `Type (string)`: "file" or "directory".
    - `Children ([]*TreeNode, omitempty)`: Child nodes (for directories).
- `LSResponseMetadata`: Struct for metadata about the ls operation.
    - `NumberOfFiles (int)`: Total number of files and directories listed (up to the limit).
    - `Truncated (bool)`: Indicates if the listing was truncated due to the limit.
- `lsTool`: Implements the `BaseTool` interface (empty struct).

### Constants
- `LSToolName ("ls")`: The registered name of the tool.
- `MaxLSFiles (1000)`: The maximum number of files and directories to include in the listing before truncating.
- `lsDescription (string)`: A detailed description for the LLM on when and how to use this tool. It explains:
    - Purpose: Exploring directory structure, understanding project organization.
    - Usage: Provide a path, optional ignore patterns.
    - Features: Hierarchical view, automatic skipping of hidden files/common system dirs (like `__pycache__`, `.git`), pattern-based filtering.
    - Limitations: Results limited to `MaxLSFiles`, very large directories truncated, no file sizes/permissions, not for full recursive listing of large projects.
    - Tips: Use Glob for pattern-based finding, Grep for content search.

### Functions
- `NewLsTool() BaseTool`: Constructor for `lsTool`.
- `(l *lsTool) Info() ToolInfo`: Returns metadata about the tool.
- `(l *lsTool) Run(ctx context.Context, call ToolCall) (ToolResponse, error)`: The core logic when the ls tool is invoked.
    1.  Parses `call.Input` into `LSParams`.
    2.  Sets `searchPath` to current working directory if not provided, ensuring it's absolute.
    3.  Validates that `searchPath` exists.
    4.  Calls `listDirectory()` to get a flat list of files and directories, respecting `ignorePatterns` and `MaxLSFiles`.
    5.  Calls `createFileTree()` to convert the flat list into a hierarchical `[]*TreeNode` structure.
    6.  Calls `printTree()` to generate a string representation of the tree.
    7.  If `truncated` is true, prepends a warning message to the output.
    8.  Returns the tree string with `LSResponseMetadata`.
- `listDirectory(initialPath string, ignorePatterns []string, limit int) ([]string, bool, error)`:
    - Walks the `initialPath` using `filepath.Walk()`.
    - For each entry, calls `shouldSkip()` to determine if it should be ignored.
    - If not skipped and not the `initialPath` itself, adds the path to `results` (appending a separator for directories).
    - Stops walking if `results` count reaches `limit`, setting `truncated` to true.
    - Returns the list of paths, truncation status, and any error.
- `shouldSkip(path string, ignorePatterns []string) bool`:
    - Checks if a path should be skipped. Returns `true` if:
        - Base name starts with `.` (hidden).
        - Path contains common ignored directory names (e.g., `__pycache__`, `node_modules`, `.git`).
        - Base name matches common ignored file names/extensions (e.g., `*.pyc`, `.DS_Store`).
        - Path matches any of the user-provided `ignorePatterns` (glob matching).
- `createFileTree(sortedPaths []string) []*TreeNode`:
    - Takes a flat, sorted list of file/directory paths.
    - Builds a tree structure (`[]*TreeNode`) by parsing path components and linking parent/child nodes using a `pathMap`.
- `printTree(tree []*TreeNode, rootPath string) string`:
    - Takes the root of the `TreeNode` structure and the original `rootPath`.
    - Recursively calls `printNode` to generate an indented, string-based tree representation.
- `printNode(builder *strings.Builder, node *TreeNode, level int)`:
    - Helper for `printTree`. Appends the current `node` (with indentation based on `level`) to the `strings.Builder`.
    - Recursively calls itself for children if the node is a directory.

## Important Variables/Constants
- `LSToolName`: The registered name.
- `MaxLSFiles`: Limit on the number of items listed.
- `lsDescription`: Key instructions for the LLM.
- The unexported `commonIgnored` list in `shouldSkip` defines common patterns/names to exclude.

## Usage Examples

LLM wants to list the contents of the `src/app` directory, ignoring `*.test.go` files:
```json
{
  "type": "tool_use",
  "id": "tool_ls_1",
  "name": "ls",
  "input": {
    "path": "src/app",
    "ignore": ["*.test.go"]
  }
}
```
`lsTool.Run` would:
1. Call `listDirectory("src/app", ["*.test.go"], 1000)`.
2. Build a tree from the results.
3. Print the tree to a string, e.g.:
   ```
   - src/app/
     - models/
       - user.go
       - product.go
     - services/
       - auth.go
     - main.go
   ```
4. Return this string to the LLM.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.WorkingDirectory()`.
    - Relies on types from the parent `tools` package.
- **External Libraries:**
    - `encoding/json`, `fmt`, `os`, `path/filepath`, `strings`, `context`: Standard Go libraries.
- **Interactions:**
    - Provides directory exploration capabilities.
    - Filters results based on common ignore patterns and user-specified ignores.
    - Output is a text-based tree, designed for easy parsing or display.
    - Complements tools like `Glob` (for pattern-based file finding) and `Grep` (for content searching).

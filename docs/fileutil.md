# fileutil.go

## Overview

The `fileutil.go` file in the `internal/fileutil` package provides helper functions for file system interactions. Its main responsibilities include:
- Checking for the availability of `rg` (ripgrep) and `fzf` command-line tools.
- Constructing `exec.Cmd` objects for `rg` and `fzf` with appropriate arguments.
- Implementing a globbing function (`GlobWithDoublestar`) that supports `**` patterns, filters hidden files/directories, sorts results by modification time, and can limit the number of returned matches.
- Providing a utility to skip common hidden or build-artifact directories and files.

## Key Components

### Package Variables
- `rgPath (string)`: Stores the path to the `rg` executable if found; otherwise, it's an empty string.
- `fzfPath (string)`: Stores the path to the `fzf` executable if found; otherwise, it's an empty string.

### Initialization (`init` function)
- The `init()` function runs when the package is loaded. It uses `exec.LookPath` to find `rg` and `fzf` in the system's PATH.
- If these tools are not found, it logs a warning message and sets the respective path variables to empty strings.

### Functions
- `GetRgCmd(globPattern string) *exec.Cmd`:
    - Returns an `*exec.Cmd` configured to run `rg` for finding files.
    - If `rgPath` is empty (rg not found), it returns `nil`.
    - Arguments passed to `rg` include `--files` (list files), `-L` (follow symlinks), `--null` (null-terminate output for safe parsing).
    - If `globPattern` is provided, it adds `--glob` with the pattern to `rg`'s arguments. It ensures the glob pattern is treated as relative to the root of the search if not absolute.
    - Sets the command's working directory to `.` (current directory).
- `GetFzfCmd(query string) *exec.Cmd`:
    - Returns an `*exec.Cmd` configured to run `fzf`.
    - If `fzfPath` is empty (fzf not found), it returns `nil`.
    - Arguments passed to `fzf` include `--filter` (with the provided `query`), `--read0` (read null-terminated input), `--print0` (print null-terminated output).
    - Sets the command's working directory to `.` (current directory).
- `SkipHidden(path string) bool`:
    - Determines if a given `path` should be skipped based on common conventions for hidden files/directories or build artifacts.
    - Returns `true` if:
        - The base name of the path starts with a `.` (e.g., `.git`, `.env`).
        - Any part of the path matches a predefined list of common ignored directories (e.g., `node_modules`, `vendor`, `dist`, `.opencode`).
    - Otherwise, returns `false`.
- `GlobWithDoublestar(pattern, searchPath string, limit int) ([]string, bool, error)`:
    - Performs a file search using a glob `pattern` (supporting `**` for recursive matching via `doublestar.GlobWalk`) within the `searchPath`.
    - It walks the file system using `os.DirFS(searchPath)`.
    - Filters out directories and any paths that `SkipHidden()` returns `true` for.
    - Collects matching file paths along with their modification times into a `FileInfo` struct.
    - If `limit` is greater than 0, it stops collecting further matches if `limit * 2` entries are found (an early exit optimization).
    - Sorts the collected `FileInfo` entries by modification time in descending order (most recent first).
    - If `limit` is greater than 0 and the number of matches exceeds `limit`, it truncates the results to `limit` entries and sets a `truncated` flag to `true`.
    - Returns a slice of matched file paths (as strings), a boolean indicating if the results were truncated, and any error encountered.

### Structs
- `FileInfo`: A simple struct to hold a file's `Path` and `ModTime` (modification time), used internally by `GlobWithDoublestar` for sorting.

## Important Variables/Constants
- The unexported `commonIgnoredDirs` map within `SkipHidden` defines a set of directory names that are typically excluded from searches and indexing.

## Usage Examples

Getting an `rg` command for listing all Go files:
```go
rgCmd := fileutil.GetRgCmd("*.go")
if rgCmd != nil {
    // Execute rgCmd and process its output
    // output, err := rgCmd.Output()
}
```

Globbing for files with a limit and sorting by modification time:
```go
matches, truncated, err := fileutil.GlobWithDoublestar("**/*.js", ".", 100)
if err != nil {
    // Handle error
}
// 'matches' will contain up to 100 most recently modified .js files in the current directory and subdirectories.
// 'truncated' will be true if more than 100 such files were found.
```

Checking if a file should be skipped:
```go
if fileutil.SkipHidden("node_modules/somepackage/file.js") {
    // This path would be skipped
}
if fileutil.SkipHidden(".git/config") {
    // This path would also be skipped
}
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/logging`: For logging warnings if `rg` or `fzf` are not found.
- **External Libraries:**
    - `github.com/bmatcuk/doublestar/v4`: For `**` glob pattern matching.
    - `os`, `os/exec`, `path/filepath`, `strings`, `time`, `sort`, `io/fs`: Standard Go libraries for file system operations, command execution, path manipulation, etc.
- **Interactions:**
    - Interacts with the operating system to check for the existence of `rg` and `fzf` executables.
    - `GetRgCmd` and `GetFzfCmd` prepare commands to be run as external processes.
    - `GlobWithDoublestar` directly interacts with the file system to list and stat files.
    - The effectiveness of `GetRgCmd` and `GetFzfCmd` depends on whether the user has `ripgrep` and `fzf` installed and available in their PATH. The package degrades gracefully by returning `nil` if they are not found, allowing calling code to implement fallbacks.

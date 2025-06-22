# files-folders.go

## Overview

The `files-folders.go` file, part of the `completions` package, provides functionality for generating file and folder path completions, typically for use in input fields or command palettes within the TUI. It implements the `dialog.CompletionProvider` interface to integrate with UI components that require completion suggestions. The core logic involves using tools like `rg` (ripgrep) and `fzf` (fuzzy finder) if available, or falling back to Go-native globbing and fuzzy matching to find relevant files and folders based on a user's query.

## Key Components

### Structs
- `filesAndFoldersContextGroup`: This struct implements the `dialog.CompletionProvider` interface.
    - `prefix`: A string field, likely used to identify this completion group (e.g., "file").

### Functions
- `(cg *filesAndFoldersContextGroup) GetId() string`: Returns the prefix ID of the completion group.
- `(cg *filesAndFoldersContextGroup) GetEntry() dialog.CompletionItemI`: Returns a `dialog.CompletionItem` that represents the main entry for this provider (e.g., an item titled "Files & Folders").
- `processNullTerminatedOutput(outputBytes []byte) []string`: A helper function to process null-terminated output (common from tools like `rg --null` or `fzf -0`) into a slice of strings. It also filters out hidden files/folders based on `fileutil.SkipHidden`.
- `(cg *filesAndFoldersContextGroup) getFiles(query string) ([]string, error)`: This is the core function for fetching file and folder matches. It employs a strategy based on available command-line tools:
    1.  **rg + fzf**: If both `ripgrep` and `fzf` are available, `rg` lists files, and its output is piped to `fzf` for fuzzy searching.
    2.  **rg only**: If only `ripgrep` is available, it lists all files, and then Go's `fuzzy.Find` is used to filter based on the query.
    3.  **fzf only**: If only `fzf` is available, `fileutil.GlobWithDoublestar` lists files, and this list is piped to `fzf`.
    4.  **Fallback**: If neither `rg` nor `fzf` is available, `fileutil.GlobWithDoublestar` lists files, and `fuzzy.Find` is used for filtering.
    It ensures hidden files are skipped.
- `(cg *filesAndFoldersContextGroup) GetChildEntries(query string) ([]dialog.CompletionItemI, error)`: Implements the `CompletionProvider` interface method. It calls `getFiles(query)` to get matching file paths and then converts these paths into `dialog.CompletionItemI` objects.
- `NewFileAndFolderContextGroup() dialog.CompletionProvider`: A constructor function that creates and returns a new instance of `filesAndFoldersContextGroup`.

## Important Variables/Constants

This file does not define exported package-level constants or variables.

## Usage Examples

This provider is typically registered with a UI component that handles completions, such as a fuzzy finder dialog or an input field.

```go
// Somewhere in the TUI setup
// import "github.com/opencode-ai/opencode/internal/completions"
// import "github.com/opencode-ai/opencode/internal/tui/components/dialog"

// Create the provider
fileCompleter := completions.NewFileAndFolderContextGroup()

// Register it with a component that uses dialog.CompletionProvider
// e.g., a fuzzy search component
// fuzzyFinder.AddProvider(fileCompleter)

// When the user types in the associated input field, the component would call:
// items, err := fileCompleter.GetChildEntries("some/path/query")
// And then display these items to the user.
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/fileutil`: For utility functions like `GetRgCmd`, `GetFzfCmd`, `GlobWithDoublestar`, and `SkipHidden`.
    - `github.com/opencode-ai/opencode/internal/logging`: For debug logging.
    - `github.com/opencode-ai/opencode/internal/tui/components/dialog`: For the `dialog.CompletionProvider` interface and `dialog.CompletionItemI` (and its concrete type `dialog.CompletionItem`).
- **External Libraries:**
    - `github.com/lithammer/fuzzysearch/fuzzy`: For fuzzy string matching when `fzf` tool is not available or not used in a particular execution path.
- **Interactions:**
    - Interacts with the operating system by attempting to execute `rg` and `fzf` commands if they are found in the system's PATH.
    - Provides completion items to TUI dialog components.
    - Relies on `fileutil` to determine the availability and command structure for `rg` and `fzf`, and for performing glob operations.

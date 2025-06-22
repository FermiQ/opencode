# patch.go

## Overview

The `patch.go` file, also part of the `internal/diff` package, defines and implements a custom text-based patch format. This format is used to describe changes to files, including adding, deleting, and updating file content, as well as moving files. The file provides a parser for this custom format (`TextToPatch`), logic to convert these patches into a `Commit` structure representing file changes, and functions to apply these commits to the file system. It also includes validation for these patches.

This custom patch format appears to be designed for AI agent interactions, where an agent might output changes in this specific textual format.

## Key Components

### Enums and Types
- `ActionType`: String enum (`add`, `delete`, `update`) representing the type of file change.
- `FileChange`: Struct representing a change to a single file.
    - `Type (ActionType)`
    - `OldContent (*string)`: Pointer to original content (for delete/update).
    - `NewContent (*string)`: Pointer to new content (for add/update).
    - `MovePath (*string)`: Optional new path if the file is moved during an update.
- `Commit`: Represents a set of changes to be applied.
    - `Changes (map[string]FileChange)`: Maps file paths to `FileChange` objects.
- `Chunk`: Represents a section of changes within an updated file in the custom patch format.
    - `OrigIndex (int)`: Line index in the original file where the chunk starts.
    - `DelLines ([]string)`: Lines to delete from `OrigIndex`.
    - `InsLines ([]string)`: Lines to insert after deleted lines (or at `OrigIndex` if no deletions).
- `PatchAction`: Represents an action (add, delete, update) for a single file within the parsed patch.
    - `Type (ActionType)`
    - `NewFile (*string)`: Content for a new file (for `ActionAdd`).
    - `Chunks ([]Chunk)`: Chunks of changes (for `ActionUpdate`).
    - `MovePath (*string)`: Optional new path if the file is moved.
- `Patch`: Represents the fully parsed custom patch.
    - `Actions (map[string]PatchAction)`: Maps original file paths to `PatchAction` objects.
- `DiffError`: Custom error type for patch-related errors.

### Parser
- `Parser`: Struct managing the state of parsing the custom patch format.
    - `currentFiles (map[string]string)`: Content of files relevant to the patch.
    - `lines ([]string)`: Lines of the patch text.
    - `index (int)`: Current line being parsed.
    - `patch (Patch)`: The resulting parsed patch.
    - `fuzz (int)`: A counter for "fuzziness" if context lines don't match exactly.
- `NewParser(currentFiles map[string]string, lines []string) *Parser`: Constructor.
- `(p *Parser) Parse() error`: Main parsing loop. It reads lines and dispatches to specific parsers based on prefixes like `*** Update File: `, `*** Delete File: `, `*** Add File: `.
- `(p *Parser) parseUpdateFile(text string) (PatchAction, error)`: Parses the content for an `ActionUpdate`. It looks for context lines (prefixed with `@@ `) and then extracts delete (`-`) and insert (`+`) lines.
- `(p *Parser) parseAddFile() (PatchAction, error)`: Parses the content for an `ActionAdd`, collecting all lines prefixed with `+`.
- `findContext(lines []string, context []string, start int, eof bool) (int, int)`: Tries to find the `context` lines within the `lines` of an actual file, starting from `start` index. It allows for some "fuzziness" by trying exact matches, then matches ignoring trailing whitespace, then matches ignoring all whitespace. Returns the found index and a fuzz score.
- `peekNextSection(lines []string, initialIndex int) ([]string, []Chunk, int, bool)`: Reads ahead in the patch text to extract context lines and the delete/insert lines that form `Chunk`s for an update operation.

### Public API and Processing Logic
- `TextToPatch(text string, orig map[string]string) (Patch, int, error)`: Top-level function to parse a complete patch `text` into a `Patch` object. `orig` provides the current content of files that might be updated or deleted. Returns the parsed `Patch`, a fuzz score, and an error.
- `IdentifyFilesNeeded(text string) []string`: Scans patch `text` and returns a list of file paths mentioned in `*** Update File: ` or `*** Delete File: ` sections.
- `IdentifyFilesAdded(text string) []string`: Scans patch `text` for `*** Add File: ` sections and returns those paths.
- `getUpdatedFile(text string, action PatchAction, path string) (string, error)`: Applies the `Chunks` from a `PatchAction` (of type `ActionUpdate`) to the original file `text` to produce the new content.
- `PatchToCommit(patch Patch, orig map[string]string) (Commit, error)`: Converts a parsed `Patch` object into a `Commit` object. This involves transforming `PatchAction`s into `FileChange`s.
- `AssembleChanges(orig map[string]string, updatedFiles map[string]string) Commit`: Creates a `Commit` by comparing a map of original file contents (`orig`) with a map of new/updated file contents (`updatedFiles`).
- `LoadFiles(paths []string, openFn func(string) (string, error)) (map[string]string, error)`: Loads content for specified `paths` using the provided `openFn`.
- `ApplyCommit(commit Commit, writeFn func(string, string) error, removeFn func(string) error) error`: Applies the changes in a `Commit` to the filesystem using provided `writeFn` and `removeFn`.
- `ProcessPatch(text string, openFn func(string) (string, error), writeFn func(string, string) error, removeFn func(string) error) (string, error)`: A high-level function that orchestrates the entire patch application process: identifies needed files, loads them, parses the patch text, converts to a commit, and applies the commit.
- `OpenFile(p string) (string, error)`: Default implementation for `openFn` using `os.ReadFile`.
- `WriteFile(p string, content string) error`: Default implementation for `writeFn` using `os.WriteFile`. Creates directories if needed. Disallows absolute paths.
- `RemoveFile(p string) error`: Default implementation for `removeFn` using `os.Remove`.
- `ValidatePatch(patchText string, files map[string]string) (bool, string, error)`: Validates a given `patchText` against a map of current file contents. Checks for format, file existence, fuzziness, and ability to convert to a commit.

## Custom Patch Format Example Snippets

**Update File:**
```
*** Update File: path/to/file.txt
@@ existing context line 1  // Context line (must match in original file)
-old line to delete
+new line to insert
@@ existing context line 2
*** End of File (optional, if changes are at the end)
```

**Add File:**
```
*** Add File: path/to/new_file.txt
+content of line 1
+content of line 2
```

**Delete File:**
```
*** Delete File: path/to/delete_me.txt
```

A full patch starts with `*** Begin Patch` and ends with `*** End Patch`.

## Usage Examples

Applying a patch string to the filesystem:
```go
patchString := `
*** Begin Patch
*** Update File: main.go
@@ func main() {
-    fmt.Println("Hello, world!")
+    fmt.Println("Hello, OpenCode!")
@@ }
*** End Patch
`
// openFn, writeFn, removeFn can be diff.OpenFile, diff.WriteFile, diff.RemoveFile
// or custom functions for testing or virtual filesystems.
message, err := diff.ProcessPatch(patchString, diff.OpenFile, diff.WriteFile, diff.RemoveFile)
if err != nil {
    // Handle error
}
fmt.Println(message) // "Patch applied successfully"
```

Validating a patch:
```go
// Assume 'files' is map[string]string of current file contents
isValid, message, err := diff.ValidatePatch(patchString, files)
if err != nil {
    // Handle processing error during validation
}
if !isValid {
    fmt.Printf("Patch invalid: %s\n", message)
}
```

## Dependencies and Interactions

- **Internal Dependencies:** None beyond standard Go packages.
- **External Libraries:** None.
- **Interactions:**
    - This file defines a self-contained system for a custom patch format.
    - It interacts with the file system through functions like `OpenFile`, `WriteFile`, `RemoveFile` (or user-provided equivalents).
    - The format is line-oriented and uses specific prefixes (`***`, `@@`, `+`, `-`) to denote operations and content.
    - The parser includes logic for "fuzzy" context matching, which can tolerate some whitespace differences between the patch's context lines and the actual file content.

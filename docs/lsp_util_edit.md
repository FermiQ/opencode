# edit.go (in internal/lsp/util)

## Overview

The `edit.go` file, part of the `internal/lsp/util` package, provides core utility functions for applying Language Server Protocol (LSP) `WorkspaceEdit` operations to the local filesystem. A `WorkspaceEdit` can contain a mix of text edits to existing files and resource operations like creating, deleting, or renaming files. This file handles the logic of reading files, applying text changes in the correct order, and performing file system operations.

## Key Components

### Functions
- `applyTextEdits(uri protocol.DocumentUri, edits []protocol.TextEdit) error`:
    - Applies a slice of `protocol.TextEdit`s to the document specified by `uri`.
    - Reads the file content.
    - Detects the original line ending style (`\r\n` or `\n`) and whether the file ends with a newline to preserve it.
    - **Overlap Check**: Checks if any of the provided `edits` overlap; if so, returns an error as overlapping edits are ambiguous.
    - **Sorts Edits**: Sorts the `edits` in reverse order (from the end of the file to the beginning, and from right to left on a line). This is crucial for applying multiple non-overlapping edits correctly without subsequent edits' ranges being invalidated by prior ones.
    - Iterates through the sorted edits and applies each one to an in-memory representation of the file lines using `applyTextEdit()`.
    - Joins the modified lines back together, preserving the original line ending style and ensuring a final newline if the original had one.
    - Writes the modified content back to the file using `os.WriteFile()`.
- `applyTextEdit(lines []string, edit protocol.TextEdit) ([]string, error)`:
    - Applies a single `protocol.TextEdit` to a slice of `lines` (representing the file content).
    - Extracts start/end line and character positions from `edit.Range`.
    - Validates that start/end line numbers are within the bounds of the `lines` slice.
    - Constructs the new content by taking the prefix before the edit range, inserting `edit.NewText`, and appending the suffix after the edit range.
    - Handles multi-line `edit.NewText` correctly.
    - If `edit.NewText` is empty, it effectively deletes the content within the range.
    - Returns the modified slice of lines or an error if positions are invalid.
- `applyDocumentChange(change protocol.DocumentChange) error`:
    - Applies a single `protocol.DocumentChange` (which can be a `TextDocumentEdit`, `CreateFile`, `RenameFile`, or `DeleteFile`).
    - **CreateFile**: If `change.CreateFile` is set, it creates an empty file at the specified URI. It respects `Overwrite` and `IgnoreIfExists` options.
    - **DeleteFile**: If `change.DeleteFile` is set, it deletes the file or directory (recursively if `Recursive` option is set).
    - **RenameFile**: If `change.RenameFile` is set, it renames the file. It respects the `Overwrite` option.
    - **TextDocumentEdit**: If `change.TextDocumentEdit` is set, it converts the edits (which can be `AnnotatedTextEdit` or `SnippetTextEdit`) to basic `TextEdit`s using `edit.AsTextEdit()` (from `lsp/protocol/interface.go`) and then calls `applyTextEdits` to apply them.
- `ApplyWorkspaceEdit(edit protocol.WorkspaceEdit) error`:
    - The main public function to apply a full `protocol.WorkspaceEdit`.
    - First, it processes the `edit.Changes` map (which is an older way to specify text edits, mapping URIs to `[]TextEdit`). It iterates through this map and calls `applyTextEdits` for each entry.
    - Then, it processes the `edit.DocumentChanges` slice (the preferred way, allowing mixed resource operations and versioned edits). It iterates through this slice and calls `applyDocumentChange` for each `DocumentChange`.
    - Returns an error if any part of applying the edit fails.
- `rangesOverlap(r1, r2 protocol.Range) bool`:
    - A helper function to determine if two LSP `Range` objects overlap.

## Important Variables/Constants
This file does not define exported package-level constants or variables.

## Usage Examples

This utility is primarily used by the LSP client's handler for the `workspace/applyEdit` request (`lsp.HandleApplyEdit` in `lsp/handlers.go`). When an LSP server sends a `workspace/applyEdit` request, the handler calls `util.ApplyWorkspaceEdit` to make the requested changes to the filesystem.

Conceptual usage in an LSP handler:
```go
// import "github.com/opencode-ai/opencode/internal/lsp/util"
// import "github.com/opencode-ai/opencode/internal/lsp/protocol"

// func HandleApplyEdit(params json.RawMessage) (any, error) {
//     var applyEditParams protocol.ApplyWorkspaceEditParams
//     if err := json.Unmarshal(params, &applyEditParams); err != nil {
//         return nil, err
//     }
//
//     err := util.ApplyWorkspaceEdit(applyEditParams.Edit)
//     if err != nil {
//         // Log error and report failure to LSP server
//         return protocol.ApplyWorkspaceEditResult{Applied: false, FailureReason: err.Error()}, nil
//     }
//     return protocol.ApplyWorkspaceEditResult{Applied: true}, nil
// }
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/lsp/protocol`: For all LSP data structures like `DocumentUri`, `TextEdit`, `Range`, `DocumentChange`, `WorkspaceEdit`, `CreateFile`, `RenameFile`, `DeleteFile`, and `TextDocumentEdit`. Also uses `edit.AsTextEdit()` for converting union types.
- **External Libraries:**
    - `bytes`: For `bytes.Contains` and `bytes.HasSuffix`.
    - `fmt`, `os`, `sort`, `strings`: Standard Go libraries.
- **Interactions:**
    - Directly interacts with the filesystem by reading, writing, creating, deleting, and renaming files based on LSP `WorkspaceEdit` instructions.
    - The logic for applying text edits, especially sorting them in reverse, is critical for correctness when multiple edits are applied to the same file.
    - It handles different types of document changes specified by the LSP, providing a bridge between the abstract edit operations defined by the protocol and concrete filesystem actions.
    - Preservation of line endings and final newlines during text edits shows attention to detail in maintaining file integrity.

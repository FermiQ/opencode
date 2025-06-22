# tsdocument-changes.go (in internal/lsp/protocol)

## Overview

The `tsdocument-changes.go` file, part of the `internal/lsp/protocol` package, defines the `DocumentChange` struct. This struct represents a "union type" or "sum type" as used in the Language Server Protocol, specifically for describing various kinds of changes to documents or files within a workspace. A `DocumentChange` can represent a text edit to an existing document, or a file operation like creating, renaming, or deleting a file. The file includes custom JSON marshalling and unmarshalling logic to handle the polymorphic nature of this type.

The copyright notice "Copyright 2022 The Go Authors" suggests that this file, or parts of it, might be adapted from or inspired by the Go team's LSP implementation (e.g., `gopls` or its underlying libraries).

## Key Components

### Structs
- `DocumentChange`: Represents a single change operation on a document or file. It's a union type, meaning exactly one of its fields should be non-nil.
    - `TextDocumentEdit (*TextDocumentEdit)`: If non-nil, represents a textual edit to an existing document. `TextDocumentEdit` itself (defined elsewhere) typically includes the document's URI, version, and a list of text edits.
    - `CreateFile (*CreateFile)`: If non-nil, represents a file creation operation. `CreateFile` (defined elsewhere) includes the URI of the file to be created and optional options.
    - `RenameFile (*RenameFile)`: If non-nil, represents a file rename operation. `RenameFile` (defined elsewhere) includes the old and new URIs and optional options.
    - `DeleteFile (*DeleteFile)`: If non-nil, represents a file deletion operation. `DeleteFile` (defined elsewhere) includes the URI of the file to be deleted and optional options.

### Methods on `DocumentChange`
- `Valid() bool`:
    - Checks if the `DocumentChange` instance is valid, meaning that exactly one of its pointer fields (`TextDocumentEdit`, `CreateFile`, `RenameFile`, `DeleteFile`) is non-nil.
    - Returns `true` if valid, `false` otherwise.
- `UnmarshalJSON(data []byte) error`:
    - Implements the `json.Unmarshaler` interface for custom JSON deserialization.
    - This is necessary because the type of change needs to be determined from the JSON structure.
    - It first unmarshals the data into a generic `map[string]any`.
    - If a "textDocument" field is present, it assumes the change is a `TextDocumentEdit` and unmarshals into that type.
    - Otherwise, it looks for a "kind" field (which is common in LSP for file operations: "create", "rename", "delete"). Based on the "kind", it unmarshals the data into the corresponding `CreateFile`, `RenameFile`, or `DeleteFile` struct.
    - Returns an error if the kind is unexpected or if unmarshalling fails.
- `MarshalJSON() ([]byte, error)`:
    - Implements the `json.Marshaler` interface for custom JSON serialization.
    - It marshals the non-nil field of the `DocumentChange` struct. For example, if `TextDocumentEdit` is set, it marshals `d.TextDocumentEdit`.
    - Returns an error if all fields are nil (an invalid state for the union type).

## Important Variables/Constants
This file does not define exported package-level constants or variables beyond the `DocumentChange` type itself.

## Usage Examples

The `DocumentChange` struct is primarily used as part of a `WorkspaceEdit` (defined elsewhere in the `protocol` package). A `WorkspaceEdit` can have a list of `DocumentChange`s, allowing an LSP server to request multiple atomic changes (text edits, file creations, renames, deletions) from the client.

Conceptual example of how a `WorkspaceEdit` might be processed:
```go
// import "github.com/opencode-ai/opencode/internal/lsp/protocol"
// import "encoding/json"

// var workspaceEditPayload json.RawMessage // Received from LSP server
// var workspaceEdit protocol.WorkspaceEdit
// if err := json.Unmarshal(workspaceEditPayload, &workspaceEdit); err != nil { /* ... */ }

// if workspaceEdit.DocumentChanges != nil {
//     for _, change := range *workspaceEdit.DocumentChanges { // change is of type protocol.DocumentChange
//         if !change.Valid() {
//             // Log error: invalid DocumentChange
//             continue
//         }
//         if change.TextDocumentEdit != nil {
//             // Process text document edit
//             // Apply change.TextDocumentEdit.Edits to change.TextDocumentEdit.TextDocument.URI
//         } else if change.CreateFile != nil {
//             // Process file creation
//             // Create file at change.CreateFile.URI
//         } else if change.RenameFile != nil {
//             // Process file rename
//             // Rename from change.RenameFile.OldURI to change.RenameFile.NewURI
//         } else if change.DeleteFile != nil {
//             // Process file deletion
//             // Delete file at change.DeleteFile.URI
//         }
//     }
// }
```
The custom `UnmarshalJSON` and `MarshalJSON` methods are invoked automatically by Go's `encoding/json` package when `DocumentChange` instances are part of larger structures being serialized or deserialized.

## Dependencies and Interactions

- **Internal Dependencies:**
    - Relies on other LSP types defined within the `internal/lsp/protocol` package:
        - `TextDocumentEdit`
        - `CreateFile`
        - `RenameFile`
        - `DeleteFile`
        - (Implicitly) `Location`, `Range`, `DocumentUri` as part of the above types.
- **External Libraries:**
    - `encoding/json`: For custom JSON marshalling and unmarshalling.
    - `fmt`: For error formatting.
- **Interactions:**
    - This struct is a key part of the LSP's `WorkspaceEdit` capability, allowing servers to request complex, multi-file changes.
    - The custom JSON handling is essential for correctly interpreting the varied structures that can represent a document change in LSP messages.
    - The `HandleApplyEdit` function in `lsp/handlers.go` would typically receive a `WorkspaceEdit` containing these `DocumentChange` objects and be responsible for applying them.

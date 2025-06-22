# interface.go (in internal/lsp/protocol)

## Overview

The `interface.go` file, located in the `internal/lsp/protocol` package, defines several Go interfaces (`WorkspaceSymbolResult`, `DocumentSymbolResult`, `TextEditResult`) and associated methods. These interfaces serve to abstract over different concrete LSP types that might represent similar concepts but have slightly different structures. This is common in LSP where a result can often be one of several types (e.g., a symbol can be a full `DocumentSymbol` with hierarchy or a simpler `SymbolInformation`). The methods on the `Or_...` types (which are union types generated from the LSP specification) use these interfaces to provide a unified way to access common fields.

## Key Components

### Interfaces

- `WorkspaceSymbolResult`:
    - An interface for types that represent workspace symbols (symbols found across a project).
    - Methods:
        - `GetName() string`: Returns the name of the symbol.
        - `GetLocation() Location`: Returns the location (URI and range) of the symbol.
        - `isWorkspaceSymbol()`: A marker method to identify types that satisfy this interface.
    - Implementers shown in this file: `*WorkspaceSymbol` and `*SymbolInformation`.

- `DocumentSymbolResult`:
    - An interface for types that represent document symbols (symbols within a single document).
    - Methods:
        - `GetRange() Range`: Returns the range of the symbol within its document.
        - `GetName() string`: Returns the name of the symbol.
        - `isDocumentSymbol()`: A marker method.
    - Implementers shown in this file: `*DocumentSymbol` and `*SymbolInformation` (note `SymbolInformation` implements both `WorkspaceSymbolResult` and `DocumentSymbolResult`).

- `TextEditResult`:
    - An interface for types that can be used as text edits (representing a change to a document).
    - Methods:
        - `GetRange() Range`: The range of text to be replaced or where new text is inserted.
        - `GetNewText() string`: The new text to be inserted.
        - `isTextEdit()`: A marker method.
    - Implementer shown in this file: `*TextEdit`.

### Methods on Concrete Types
For each interface, there are corresponding getter methods implemented on the concrete LSP types that satisfy them. For example:
- `(ws *WorkspaceSymbol) GetName() string`, `(ws *WorkspaceSymbol) GetLocation() Location`, `(ws *WorkspaceSymbol) isWorkspaceSymbol()`
- `(si *SymbolInformation) GetName() string`, `(si *SymbolInformation) GetLocation() Location`, `(si *SymbolInformation) isWorkspaceSymbol()`
- `(ds *DocumentSymbol) GetRange() Range`, `(ds *DocumentSymbol) GetName() string`, `(ds *DocumentSymbol) isDocumentSymbol()`
- `(si *SymbolInformation) GetRange() Range` (uses `si.Location.Range`), `(si *SymbolInformation) isDocumentSymbol()`
- `(te *TextEdit) GetRange() Range`, `(te *TextEdit) GetNewText() string`, `(te *TextEdit) isTextEdit()`

### Methods on `Or_...` (Union) Types
These methods provide a convenient way to get a slice of a common interface type from an LSP "Or" type (which represents a value that could be one of several underlying types).
- `(r Or_Result_workspace_symbol) Results() ([]WorkspaceSymbolResult, error)`:
    - Converts the `Value` of an `Or_Result_workspace_symbol` (which could be `[]WorkspaceSymbol` or `[]SymbolInformation`) into a `[]WorkspaceSymbolResult`.
    - This allows code to iterate over workspace symbols using a common interface regardless of the specific type returned by the LSP server.
- `(r Or_Result_textDocument_documentSymbol) Results() ([]DocumentSymbolResult, error)`:
    - Converts the `Value` of an `Or_Result_textDocument_documentSymbol` (which could be `[]DocumentSymbol` or `[]SymbolInformation`) into a `[]DocumentSymbolResult`.
- `(e Or_TextDocumentEdit_edits_Elem) AsTextEdit() (TextEdit, error)`:
    - Converts an `Or_TextDocumentEdit_edits_Elem` (which could be `TextEdit` or `AnnotatedTextEdit`) into a common `TextEdit` struct. This is useful for simplifying the handling of workspace edits.

## Important Variables/Constants
This file primarily defines interfaces and methods; it does not export package-level variables or constants.

## Usage Examples

These interfaces and methods are used internally when processing responses from an LSP server that can return different but related types for the same logical concept.

Processing workspace symbols:
```go
// Assume 'responsePayload' is the result of an LSP call like 'workspace/symbol'
// var workspaceSymbolResponse protocol.Or_Result_workspace_symbol
// err := json.Unmarshal(responsePayload.Result, &workspaceSymbolResponse)
// if err != nil { /* handle error */ }

// symbols, err := workspaceSymbolResponse.Results()
// if err != nil { /* handle error */ }

// for _, symbol := range symbols {
//     // symbol is of type protocol.WorkspaceSymbolResult
//     fmt.Printf("Symbol Name: %s, Location: %s\n", symbol.GetName(), symbol.GetLocation().URI)
// }
```

Processing document symbols:
```go
// Assume 'responsePayload' is the result of 'textDocument/documentSymbol'
// var documentSymbolResponse protocol.Or_Result_textDocument_documentSymbol
// ... unmarshal ...

// docSymbols, err := documentSymbolResponse.Results()
// if err != nil { /* handle error */ }

// for _, docSymbol := range docSymbols {
//     // docSymbol is of type protocol.DocumentSymbolResult
//     fmt.Printf("Symbol: %s, Range: %v\n", docSymbol.GetName(), docSymbol.GetRange())
// }
```

Working with text edits from a workspace edit:
```go
// Assume 'editElement' is an element from a 'protocol.WorkspaceEdit.DocumentChanges'
// or 'protocol.WorkspaceEdit.Changes' which might be of type 'Or_TextDocumentEdit_edits_Elem'.

// if editElement.Value != nil { // Assuming Or_TextDocumentEdit_edits_Elem or similar structure
//     textEdit, err := editElement.AsTextEdit() // Assuming editElement is Or_TextDocumentEdit_edits_Elem
//     if err == nil {
//         // Now you have a common protocol.TextEdit to work with
//         fmt.Printf("Edit Range: %v, New Text: %s\n", textEdit.Range, textEdit.NewText)
//     }
// }
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - Deeply tied to the concrete LSP types defined elsewhere in the `internal/lsp/protocol` package (e.g., `WorkspaceSymbol`, `SymbolInformation`, `DocumentSymbol`, `TextEdit`, `AnnotatedTextEdit`, `Location`, `Range`, and the various `Or_...` union types). These types are typically auto-generated from the LSP JSON specification.
- **External Libraries:**
    - `fmt`: For error formatting.
- **Interactions:**
    - Provides a layer of abstraction over the raw, often complex, and sometimes union-typed structures defined by the Language Server Protocol.
    - Simplifies client-side code that needs to process LSP responses by offering unified interfaces for common operations (like getting a symbol's name or location) regardless of the exact type variant returned by the server.
    - The marker methods (e.g., `isWorkspaceSymbol()`) are a Go idiom for ensuring interface satisfaction without necessarily adding functional behavior to the interface itself.
    - This file is likely part of a larger set of auto-generated Go bindings for the LSP specification.

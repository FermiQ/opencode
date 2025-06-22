# tsprotocol.go (in internal/lsp/protocol)

## Overview

The `tsprotocol.go` file, found within the `internal/lsp/protocol` package, is an **auto-generated Go source file**. It should **not** be manually edited, as any changes would be overwritten by the code generation process. This file contains a comprehensive set of Go type definitions (structs, type aliases, constants for enums) that directly mirror the data structures and enumerations specified in the Language Server Protocol (LSP) version 3.17.0.

The comments at the top of the file indicate its origin: "Code generated from protocol/metaModel.json at ref release/protocol/3.17.6-next.9 (hash c94395b5da53729e6dff931293b051009ccaaaa4). https://github.com/microsoft/vscode-languageserver-node/blob/release/protocol/3.17.6-next.9/protocol/metaModel.json". This means these Go types are derived from the canonical JSON meta-model of the LSP, ensuring they accurately represent the protocol's structures.

## Key Components

This file is extensive and primarily consists of:

1.  **Struct Definitions**:
    *   Go structs corresponding to every object type defined in the LSP specification. Examples include:
        *   `InitializeParams`, `InitializeResult`, `ClientCapabilities`, `ServerCapabilities`
        *   `TextDocumentItem`, `TextDocumentIdentifier`, `VersionedTextDocumentIdentifier`
        *   `Position`, `Range`, `Location`, `LocationLink`
        *   `Diagnostic`, `CodeAction`, `CompletionItem`, `Hover`, `SignatureHelp`
        *   `WorkspaceEdit`, `TextEdit`, `AnnotatedTextEdit`
        *   Various `*Options` structs for server capabilities (e.g., `CompletionOptions`, `HoverOptions`)
        *   Various `*Params` structs for requests and notifications (e.g., `CompletionParams`, `DidOpenTextDocumentParams`)
        *   Structs for notebook documents, call hierarchy, type hierarchy, semantic tokens, inlay hints, inline values, etc.
    *   These structs have fields tagged with `json:"..."` to control their serialization and deserialization to/from JSON, matching the LSP specification. Fields are often `omitempty` if they are optional.

2.  **Type Aliases**:
    *   For simple LSP types that are essentially strings or other basic types with semantic meaning, Go type aliases are used. Examples:
        *   `DocumentUri = string`
        *   `URI = string`
        *   `ChangeAnnotationIdentifier = string`
        *   `Pattern = string` (for glob patterns)
        *   `RegularExpressionEngineKind = string`
        *   `LanguageKind = string` (e.g., "go", "typescript")

3.  **"Or" Types (Union Types)**:
    *   The LSP specification frequently uses fields that can be one of several types (union types). This file defines Go structs to represent these, typically named `Or_...` (e.g., `Or_CompletionItem_documentation`, `Or_Definition`).
    *   Each `Or_...` struct usually has a single field, often `Value interface{}`, which will hold the actual concrete type.
    *   Custom JSON marshalling and unmarshalling for these `Or_...` types are handled in `tsjson.go`.

4.  **Constants for Enumerated Values**:
    *   LSP defines many enumerations (e.g., `SymbolKind`, `CompletionItemKind`, `MessageType`, `ErrorCode`). This file defines corresponding Go constants for each member of these enumerations.
    *   Examples:
        *   `File SymbolKind = 1`, `Class SymbolKind = 5`
        *   `TextCompletion CompletionItemKind = 1`
        *   `QuickFix CodeActionKind = "quickfix"`
        *   `Created FileChangeType = 1`
        *   `PlainText MarkupKind = "plaintext"`
        *   `Error MessageType = 1`

## Important Variables/Constants
The file is almost entirely composed of type and constant definitions. There are no significant package-level variables. The constants representing enum members (like `QuickFix`, `PlainText`, `SeverityError`) are crucial for working with LSP messages.

## Usage Examples

These types are used throughout the LSP client and server implementation whenever LSP messages are constructed, sent, received, or parsed.

Creating an LSP parameter struct:
```go
// import "github.com/opencode-ai/opencode/internal/lsp/protocol"

params := protocol.InitializeParams{
    XInitializeParams: protocol.XInitializeParams{
        ProcessID: 12345,
        ClientInfo: &protocol.ClientInfo{Name: "OpenCode", Version: "0.1.0"},
        RootURI:   "file:///path/to/workspace",
        Capabilities: protocol.ClientCapabilities{ /* ... filled in ... */ },
    },
}
// This 'params' struct can then be sent as part of an 'initialize' request.
```

Interpreting a response:
```go
// Assume 'responseMsg' is an lsp.Message received from the server
// var completionResult protocol.Or_Result_textDocument_completion
// if err := json.Unmarshal(responseMsg.Result, &completionResult); err == nil {
//     if completionList, ok := completionResult.Value.(protocol.CompletionList); ok {
//         // Process CompletionList
//     } else if completionItems, ok := completionResult.Value.([]protocol.CompletionItem); ok {
//         // Process []CompletionItem
//     }
// }
```

Using an enum constant:
```go
// diagnostic := protocol.Diagnostic{
//     Severity: protocol.SeverityError, // Using the constant
//     // ... other fields
// }
```

## Dependencies and Interactions

- **Internal Dependencies:** None from other OpenCode packages, as this file is at the base of the `protocol` package, defining its types.
- **External Libraries:**
    - `encoding/json`: Used implicitly by the `json:"..."` tags for serialization/deserialization.
- **Interactions:**
    - This file provides the Go type system for the entire Language Server Protocol version 3.17.0 as used by this application.
    - It is fundamental to:
        - `lsp/client.go`: For constructing requests and interpreting responses.
        - `lsp/methods.go`: For the typed client methods that wrap generic calls.
        - `lsp/handlers.go`: For unmarshalling parameters of server-initiated requests/notifications.
        - `lsp/transport.go` and `lsp_base_protocol.go` (at `internal/lsp/`): For the basic JSON-RPC message structure which carries these typed parameters and results.
        - `lsp/protocol/tsjson.go`: Contains the custom JSON marshalling/unmarshalling logic for the `Or_...` types defined here.
    - Any interaction with an LSP server involves creating or parsing instances of the types defined in this file.
    - Its accuracy and completeness with respect to the LSP specification are critical for correct LSP communication.

# tsjson.go (in internal/lsp/protocol)

## Overview

The `tsjson.go` file, located within the `internal/lsp/protocol` package, is **auto-generated code** and should **not** be manually edited. Its primary purpose is to provide custom JSON marshalling (`MarshalJSON`) and unmarshalling (`UnmarshalJSON`) methods for numerous "Or" types (also known as union types or sum types) defined in the Language Server Protocol (LSP). These "Or" types represent fields in LSP messages that can hold values of different, but related, underlying Go types.

The comment at the top indicates it was generated from `protocol/metaModel.json` (likely from the `microsoft/vscode-languageserver-node` repository), corresponding to LSP version 3.17.0. This means the file contains boilerplate code necessary for correctly serializing and deserializing these complex LSP structures.

## Key Components

The file consists almost entirely of:

- `UnmarshalError (struct)`: A custom error type used to indicate that a JSON value did not conform to any of the expected types within an "Or" type during unmarshalling.
- **`MarshalJSON()` methods for `Or_...` types**:
    - For each `Or_...` type (e.g., `Or_CancelParams_id`, `Or_CompletionItem_documentation`, `Or_ServerCapabilities_hoverProvider`), a `MarshalJSON()` method is defined.
    - This method uses a `switch` statement on the actual type of `t.Value` (the field holding the union's current value).
    - It then calls `json.Marshal()` on that concrete value.
    - Handles `nil` values by returning `[]byte("null")`.
    - Returns an error if `t.Value` is of an unexpected type.
- **`UnmarshalJSON()` methods for `Or_...` types**:
    - For each `Or_...` type, an `UnmarshalJSON()` method is defined.
    - This method attempts to unmarshal the input JSON `x` into each of the possible underlying types for that union.
    - It tries one type, and if that fails, it tries the next, and so on. `json.NewDecoder(bytes.NewReader(x))` is used for each attempt to ensure the decoder starts fresh. `DisallowUnknownFields()` is often used to make parsing stricter.
    - If an unmarshalling attempt is successful for one of the types, `t.Value` is set to the unmarshalled value, and the method returns.
    - If `string(x) == "null"`, `t.Value` is set to `nil`.
    - If none of the attempts succeed, it returns an `UnmarshalError`.

**Examples of `Or_...` types with custom marshallers/unmarshallers defined in this file:**
(This is not an exhaustive list, but illustrates the pattern)
- `Or_CancelParams_id`: Can be `int32` or `string`.
- `Or_ClientSemanticTokensRequestOptions_full`: Can be `ClientSemanticTokensRequestFullDelta` or `bool`.
- `Or_CompletionItem_documentation`: Can be `MarkupContent` or `string`.
- `Or_Definition`: Can be `Location` or `[]Location`.
- `Or_Hover_contents`: Can be `MarkedString`, `MarkupContent`, or `[]MarkedString`.
- `Or_LSPAny`: Can be `LSPArray`, `LSPObject`, `bool`, `float64`, `int32`, `string`, or `uint32`. This is a very broad union type.
- `Or_ServerCapabilities_textDocumentSync`: Can be `TextDocumentSyncKind` (an enum) or `TextDocumentSyncOptions` (a struct).

## Important Variables/Constants
- `UnmarshalError`: The custom error type.
- The file itself is a table of generated functions rather than defining standalone variables or constants.

## Usage Examples

These `MarshalJSON` and `UnmarshalJSON` methods are not typically called directly by application code. Instead, they are invoked automatically by Go's standard `encoding/json` package whenever an LSP structure containing one of these `Or_...` types is marshalled or unmarshalled.

For example, when the `lsp.Client` receives a JSON message from an LSP server and unmarshals it into an `lsp.Message` struct, if that message's `Result` or `Params` field corresponds to an LSP type that itself contains an `Or_...` field, the custom unmarshaller for that `Or_...` type will be called.

```go
// Conceptual:
// import "encoding/json"
// import "github.com/opencode-ai/opencode/internal/lsp/protocol"

// var jsonDataFromServer []byte = ... // Received JSON payload for a CompletionItem

// var completionItem protocol.CompletionItem
// err := json.Unmarshal(jsonDataFromServer, &completionItem)
// if err != nil { /* handle error */ }

// // When the above Unmarshal happens, if CompletionItem.Documentation (which is
// // of type Or_CompletionItem_documentation) is present in the JSON, its
// // UnmarshalJSON() method from tsjson.go would have been called to correctly
// // determine if it's a string or a MarkupContent object.

// // Similarly, when sending data:
// responseParams := protocol.SomeResponse{
//     AFieldThatIsAnOrType: protocol.Or_Definition{Value: protocol.Location{ /* ... */ }},
// }
// jsonBytes, err := json.Marshal(responseParams)
// // The MarshalJSON() for Or_Definition would be invoked here.
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - Deeply coupled with all the concrete LSP types and `Or_...` union type definitions found throughout the `internal/lsp/protocol` package (many of which are in `tsprotocol.go`).
- **External Libraries:**
    - `bytes`: For `bytes.NewReader`.
    - `encoding/json`: This entire file provides custom hooks for this package.
    - `fmt`: For error messages.
- **Interactions:**
    - This file is essential for the correct serialization and deserialization of LSP messages that use union types. Without these custom handlers, `encoding/json` would not know how to correctly interpret a field that could be, for example, either a simple string or a complex object.
    - It is a core part of the LSP type bindings, ensuring that Go structs accurately represent the flexible JSON structures defined by the Language Server Protocol.
    - The code is highly repetitive due to the nature of needing a marshaller/unmarshaller pair for each distinct union type defined in the LSP specification. This is a strong indicator of its auto-generated origin.

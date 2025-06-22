# lsp_base_protocol.go (in internal/lsp)

## Overview

The `protocol.go` file in the `internal/lsp` package (this documentation refers to the one at the root of `internal/lsp`, not `internal/lsp/protocol/`) defines the fundamental structures for JSON-RPC 2.0 messages, which form the basis of communication in the Language Server Protocol (LSP). It provides the `Message` struct to represent requests, responses, and notifications, and a `ResponseError` struct for error responses. It also includes constructor functions for creating new request and notification messages.

## Key Components

### Structs
- `Message`: Represents a generic JSON-RPC 2.0 message. This struct is versatile enough to model:
    - **Requests**: `ID` is set, `Method` is set, `Params` may be set. `Result` and `Error` are omitted.
    - **Responses**: `ID` is set (matching a request ID), `Result` or `Error` is set. `Method` and `Params` are omitted.
    - **Notifications**: `Method` is set, `Params` may be set. `ID`, `Result`, and `Error` are omitted.
    - Fields:
        - `JSONRPC (string)`: Always "2.0".
        - `ID (int32, omitempty)`: The identifier for a request or response. Omitted for notifications.
        - `Method (string, omitempty)`: The name of the method to be invoked (for requests and notifications).
        - `Params (json.RawMessage, omitempty)`: The parameters for the request or notification, stored as raw JSON to be unmarshalled later into specific types.
        - `Result (json.RawMessage, omitempty)`: The result of a successful request, stored as raw JSON.
        - `Error (*ResponseError, omitempty)`: An error object if a request failed.
- `ResponseError`: Represents a JSON-RPC error object, typically part of a response `Message`.
    - `Code (int)`: A number that indicates the error type that occurred.
    - `Message (string)`: A string providing a short description of the error.

### Functions
- `NewRequest(id int32, method string, params any) (*Message, error)`:
    - A constructor function to create a new JSON-RPC request `Message`.
    - Takes a request `id`, the `method` name (e.g., "textDocument/completion"), and `params` (any Go struct that can be marshalled to JSON).
    - Marshals the `params` into `json.RawMessage`.
    - Returns a pointer to the created `Message` or an error if marshalling `params` fails.
- `NewNotification(method string, params any) (*Message, error)`:
    - A constructor function to create a new JSON-RPC notification `Message`.
    - Takes the `method` name (e.g., "textDocument/didOpen") and `params`.
    - Marshals the `params` into `json.RawMessage`.
    - Returns a pointer to the created `Message` (ID field will be omitted as it's a notification) or an error if marshalling `params` fails.

## Important Variables/Constants
This file does not define exported package-level constants or variables beyond the struct and function definitions.

## Usage Examples

Creating a new LSP request:
```go
// import "github.com/opencode-ai/opencode/internal/lsp"
// import "github.com/opencode-ai/opencode/internal/lsp/protocol" // For param types

// requestID := int32(1)
// method := "textDocument/hover"
// hoverParams := protocol.HoverParams{
//     TextDocumentPositionParams: protocol.TextDocumentPositionParams{
//         TextDocument: protocol.TextDocumentIdentifier{URI: "file:///path/to/file.go"},
//         Position:     protocol.Position{Line: 10, Character: 5},
//     },
// }

// requestMessage, err := lsp.NewRequest(requestID, method, hoverParams)
// if err != nil {
//     // Handle error
// }
// // requestMessage can now be encoded and sent to an LSP server.
```

Creating a new LSP notification:
```go
// import "github.com/opencode-ai/opencode/internal/lsp"
// import "github.com/opencode-ai/opencode/internal/lsp/protocol"

// method := "textDocument/didSave"
// didSaveParams := protocol.DidSaveTextDocumentParams{
//     TextDocument: protocol.TextDocumentIdentifier{URI: "file:///path/to/file.go"},
//     // Text: optional content
// }

// notificationMessage, err := lsp.NewNotification(method, didSaveParams)
// if err != nil {
//     // Handle error
// }
// // notificationMessage can now be encoded and sent.
```

These `Message` structs are then typically encoded to JSON and sent over a transport layer (e.g., stdio pipes) to an LSP server by the `lsp.Client`'s internal `sendRequest` or `sendNotification` methods (which are likely part of `lsp.Client` or a lower-level communication handler in `protocol.go` or `transport.go`). The `lsp.Client` also uses these structures to decode incoming messages from the server.

## Dependencies and Interactions

- **Internal Dependencies:** None directly from other OpenCode packages within this specific file, but it's a foundational part of the `internal/lsp` package. The types defined here are used by `client.go`, `handlers.go`, and the methods in `methods.go`.
- **External Libraries:**
    - `encoding/json`: For `json.RawMessage` and marshalling parameters.
- **Interactions:**
    - Defines the basic wire format for all LSP communications.
    - `json.RawMessage` is used for `Params` and `Result` to allow for deferred unmarshalling into specific LSP parameter/result types, which are defined in the more extensive `internal/lsp/protocol/` directory.
    - The `lsp.Client` uses these structures to construct outgoing messages and parse incoming ones.
    - The `methods.go` file (which is auto-generated) contains numerous methods on `lsp.Client` that use `NewRequest` and `NewNotification` internally to create these `Message` objects for specific LSP methods.

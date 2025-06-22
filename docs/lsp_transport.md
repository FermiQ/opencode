# transport.go (in internal/lsp)

## Overview

The `transport.go` file in the `internal/lsp` package is responsible for the low-level transport of Language Server Protocol (LSP) messages. It implements the JSON-RPC 2.0 Base Protocol, which specifies how messages are framed with `Content-Length` headers. This file provides functions to write outgoing LSP messages and read incoming ones from `io.Writer` and `*bufio.Reader` respectively. It also contains the core message dispatch loop (`handleMessages`) for an `lsp.Client`, routing incoming messages (responses, requests from server, notifications from server) to appropriate handlers. Additionally, it defines the `Call` and `Notify` methods on `lsp.Client` for sending requests and notifications to the server.

## Key Components

### Types
- `NotificationHandler (func(params json.RawMessage))`: A function type for handling server-sent notifications.
- `ServerRequestHandler (func(params json.RawMessage) (any, error))`: A function type for handling server-initiated requests (client must respond).

### Functions
- `WriteMessage(w io.Writer, msg *Message) error`:
    - Marshals an `lsp.Message` struct (from `protocol.go` in the same package) to JSON.
    - Writes the `Content-Length` header followed by `\r\n\r\n`.
    - Writes the JSON payload of the message.
    - Includes debug logging if `config.Get().DebugLSP` is true.
- `ReadMessage(r *bufio.Reader) (*Message, error)`:
    - Reads LSP headers line by line from the `bufio.Reader` until an empty line (end of headers) is encountered.
    - Parses the `Content-Length` header to determine the size of the JSON payload.
    - Reads exactly `contentLength` bytes for the JSON payload using `io.ReadFull()`.
    - Unmarshals the JSON payload into an `lsp.Message` struct.
    - Includes debug logging for received headers and content if `config.Get().DebugLSP` is true.
- `(c *Client) handleMessages()`:
    - This is the main message processing loop for an `lsp.Client`, intended to run in a goroutine.
    - Continuously calls `ReadMessage(c.stdout)` to read incoming messages from the LSP server.
    - **Message Dispatching**:
        - **Server-to-Client Request** (has `Method` and `ID`):
            - Looks up a registered `ServerRequestHandler` based on `msg.Method`.
            - If a handler exists, it's called with `msg.Params`. The handler's result or error is then formatted into a response `Message` and sent back to the server using `WriteMessage(c.stdin, response)`.
            - If no handler is found, a "method not found" error response is sent.
        - **Server-to-Client Notification** (has `Method` but no `ID`):
            - Looks up a registered `NotificationHandler` based on `msg.Method`.
            - If a handler exists, it's called (typically in a new goroutine: `go handler(msg.Params)`) with `msg.Params`.
            - If no handler is found, the notification might be logged if debug LSP is on.
        - **Response to Client's Request** (has `ID` but no `Method`):
            - Looks up the response channel (`chan *Message`) in `c.handlers` using `msg.ID`.
            - If a channel exists, the `msg` is sent on this channel, and the channel is closed. This wakes up the goroutine that made the original request (see `Call` method).
            - The handler entry is then deleted from `c.handlers`.
    - Includes debug logging for various stages if `config.Get().DebugLSP` is true.
- `(c *Client) Call(ctx context.Context, method string, params any, result any) error`:
    - Sends a request to the LSP server and waits for a response.
    - Generates a unique request ID using `c.nextID.Add(1)`.
    - Creates a request `Message` using `NewRequest()` (from `protocol.go`).
    - Registers a response channel in `c.handlers` for the request ID.
    - Sends the request message using `WriteMessage(c.stdin, msg)`.
    - Waits to receive the response `Message` from the channel.
    - If the response contains an error (`resp.Error != nil`), returns it.
    - If `result` is not nil, unmarshals `resp.Result` (which is `json.RawMessage`) into the `result` variable. It handles `*json.RawMessage` as a special case to directly copy bytes.
    - Cleans up the entry in `c.handlers`.
- `(c *Client) Notify(ctx context.Context, method string, params any) error`:
    - Sends a notification to the LSP server (fire-and-forget).
    - Creates a notification `Message` using `NewNotification()` (from `protocol.go`).
    - Sends the message using `WriteMessage(c.stdin, msg)`.

## Important Variables/Constants
This file does not define exported package-level constants or variables beyond the handler type definitions.

## Usage Examples

The functions `WriteMessage` and `ReadMessage`, along with the `handleMessages` loop, form the core of the LSP communication transport layer managed by an `lsp.Client`. The `Call` and `Notify` methods are the primary ways other parts of the `lsp.Client` (like the methods in `methods.go`) interact with the LSP server.

Sending a request (simplified from `methods.go`):
```go
// Assume 'lspClient' is an initialized *lsp.Client
// import "github.com/opencode-ai/opencode/internal/lsp/protocol"

// var hoverResult protocol.Hover
// hoverParams := protocol.HoverParams{ /* ... */ }
// err := lspClient.Call(context.Background(), "textDocument/hover", hoverParams, &hoverResult)
// if err != nil {
//     // Handle error
// }
// // Use hoverResult
```

Sending a notification (simplified from `methods.go`):
```go
// Assume 'lspClient' is an initialized *lsp.Client
// import "github.com/opencode-ai/opencode/internal/lsp/protocol"

// didOpenParams := protocol.DidOpenTextDocumentParams{ /* ... */ }
// err := lspClient.Notify(context.Background(), "textDocument/didOpen", didOpenParams)
// if err != nil {
//     // Handle error
// }
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.Get().DebugLSP`.
    - `github.com/opencode-ai/opencode/internal/logging`: For debug logging.
    - Uses `Message`, `ResponseError`, `NewRequest`, `NewNotification` from `protocol.go` (in the same `lsp` package at the root level).
    - The methods `Call`, `Notify`, and `handleMessages` are integral parts of the `lsp.Client` struct (defined in `client.go`).
- **External Libraries:**
    - `bufio`, `context`, `encoding/json`, `fmt`, `io`, `strings`: Standard Go libraries.
- **Interactions:**
    - Implements the LSP Base Protocol for message framing (Content-Length headers).
    - The `handleMessages` loop is the heart of the client's reactive behavior, processing all incoming messages from the server.
    - `Call` provides a synchronous-like request/response mechanism over the asynchronous JSON-RPC communication by using channels.
    - `Notify` sends messages without expecting a response.
    - This layer is responsible for serializing outgoing messages to JSON and deserializing incoming JSON messages.

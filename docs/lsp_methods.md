# methods.go (in internal/lsp)

## Overview

The `methods.go` file in the `internal/lsp` package is **auto-generated code** and should **not** be manually edited. It consists of a large number of methods attached to the `lsp.Client` struct (defined in `client.go`). Each method corresponds to a specific Language Server Protocol (LSP) request or notification that the client can send to an LSP server. These methods act as typed wrappers around the generic `client.Call()` and `client.Notify()` methods (which are likely defined in `protocol.go` or `client.go`), providing a more convenient and type-safe way to interact with LSP servers.

## Key Components

The file primarily contains method definitions. Each method typically:
- Is named after the LSP method it represents (e.g., `Implementation`, `TypeDefinition`, `Completion`, `DidOpen`).
- Takes a `context.Context` as its first parameter.
- Takes a specific `protocol.*Params` struct (from `internal/lsp/protocol`) as its parameters, corresponding to the LSP method's expected input. For notifications that don't require parameters beyond the method name (like `shutdown` or `exit`), the params argument might be `nil` or a generic empty struct.
- Calls the underlying `c.Call(ctx, "lspMethodName", params, &result)` for requests or `c.Notify(ctx, "lspMethodName", params)` for notifications.
- For requests, it declares a variable of the expected result type (e.g., `var result protocol.InitializeResult`) and passes its address to `c.Call()` to be populated.
- Returns the result and/or an error.

### Examples of Generated Methods:

**Requests (expecting a response):**
- `(c *Client) Initialize(ctx context.Context, params protocol.ParamInitialize) (protocol.InitializeResult, error)`
- `(c *Client) Shutdown(ctx context.Context) error`
- `(c *Client) Completion(ctx context.Context, params protocol.CompletionParams) (protocol.Or_Result_textDocument_completion, error)`
- `(c *Client) Definition(ctx context.Context, params protocol.DefinitionParams) (protocol.Or_Result_textDocument_definition, error)`
- `(c *Client) Hover(ctx context.Context, params protocol.HoverParams) (protocol.Hover, error)`
- `(c *Client) References(ctx context.Context, params protocol.ReferenceParams) ([]protocol.Location, error)`
- `(c *Client) DocumentSymbol(ctx context.Context, params protocol.DocumentSymbolParams) (protocol.Or_Result_textDocument_documentSymbol, error)`
- `(c *Client) CodeAction(ctx context.Context, params protocol.CodeActionParams) ([]protocol.Or_Result_textDocument_codeAction_Item0_Elem, error)`
- `(c *Client) Formatting(ctx context.Context, params protocol.DocumentFormattingParams) ([]protocol.TextEdit, error)`
- `(c *Client) Rename(ctx context.Context, params protocol.RenameParams) (protocol.WorkspaceEdit, error)`
- ... and many more, covering features like call hierarchy, semantic tokens, inlay hints, diagnostics, etc.

**Notifications (fire-and-forget):**
- `(c *Client) Initialized(ctx context.Context, params protocol.InitializedParams) error`
- `(c *Client) Exit(ctx context.Context) error`
- `(c *Client) DidOpen(ctx context.Context, params protocol.DidOpenTextDocumentParams) error`
- `(c *Client) DidChange(ctx context.Context, params protocol.DidChangeTextDocumentParams) error`
- `(c *Client) DidClose(ctx context.Context, params protocol.DidCloseTextDocumentParams) error`
- `(c *Client) DidSave(ctx context.Context, params protocol.DidSaveTextDocumentParams) error`
- `(c *Client) DidChangeWatchedFiles(ctx context.Context, params protocol.DidChangeWatchedFilesParams) error`
- `(c *Client) SetTrace(ctx context.Context, params protocol.SetTraceParams) error`
- ... and others.

## Important Variables/Constants

This file contains no package-level variables or constants. Its content is solely method definitions. The comment "Generated code. Do not edit" at the top is a critical piece of information.

## Usage Examples

These methods are used by other parts of the OpenCode system that need to interact with an LSP server via an `lsp.Client` instance. For example, the `lsp.Client` itself uses some of these during initialization, and tools like `ViewTool` or `EditTool` might use them to send `textDocument/didOpen` or `textDocument/didChange` notifications.

```go
// Assume 'lspClient' is an initialized *lsp.Client instance
// Assume 'ctx' is a valid context.Context

// Example: Sending a textDocument/didOpen notification
didOpenParams := protocol.DidOpenTextDocumentParams{
    TextDocument: protocol.TextDocumentItem{
        URI:        "file:///path/to/document.go",
        LanguageID: "go",
        Version:    1,
        Text:       "package main\n\nfunc main() {}",
    },
}
err := lspClient.DidOpen(ctx, didOpenParams)
if err != nil {
    // Handle error
}

// Example: Requesting code completions
completionParams := protocol.CompletionParams{
    TextDocumentPositionParams: protocol.TextDocumentPositionParams{
        TextDocument: protocol.TextDocumentIdentifier{URI: "file:///path/to/document.go"},
        Position:     protocol.Position{Line: 1, Character: 15}, // Position in func main() {}
    },
}
completionResult, err := lspClient.Completion(ctx, completionParams)
if err != nil {
    // Handle error
}
// Process completionResult (which can be CompletionItem[], CompletionList, or null)
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/lsp/protocol`: This is a primary dependency, as all parameter and result types for the LSP methods are defined in this package (e.g., `protocol.InitializeParams`, `protocol.CompletionItem`).
    - Extends the `lsp.Client` struct (defined in `client.go`).
    - Relies on the underlying `client.Call()` and `client.Notify()` methods of `lsp.Client` to actually send the JSON-RPC messages.
- **External Libraries:**
    - `context`: Used in all method signatures.
- **Interactions:**
    - This file provides a strongly-typed API surface for all standard client-initiated LSP communications.
    - It simplifies making LSP requests and sending notifications by handling the method name string and result type casting internally.
    - The code in this file is typically generated by an LSP bindings generator tool based on the LSP specification, ensuring that the method signatures match the protocol.

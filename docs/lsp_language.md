# language.go (in internal/lsp)

## Overview

The `language.go` file in the `internal/lsp` package provides a utility function, `DetectLanguageID`, which determines a Language Server Protocol (LSP) language identifier (`protocol.LanguageKind`) based on a given file URI's extension. This is essential for LSP clients to correctly inform LSP servers about the language of a document, enabling appropriate language-specific features like syntax highlighting, diagnostics, and completions.

## Key Components

### Functions
- `DetectLanguageID(uri string) protocol.LanguageKind`:
    - Takes a file `uri` (or path) as a string.
    - Extracts the file extension using `filepath.Ext(uri)` and converts it to lowercase.
    - Uses a `switch` statement to map the lowercase extension to a predefined `protocol.LanguageKind` constant.
    - It covers a wide range of common file extensions and their corresponding LSP language kinds (e.g., ".go" maps to `protocol.LangGo`, ".js" to `protocol.LangJavaScript`, ".py" to `protocol.LangPython`).
    - Some extensions have multiple variants mapped to the same language kind (e.g., ".cpp", ".cxx", ".cc", ".c++" all map to `protocol.LangCPP`).
    - If an extension is not recognized in the switch statement, it returns an empty `protocol.LanguageKind("")`, indicating an unknown or unsupported language for LSP purposes within this mapping.

## Important Variables/Constants

This file does not define any exported package-level constants or variables itself, but it extensively uses the `protocol.LanguageKind` constants defined in the `internal/lsp/protocol` package.

## Usage Examples

This function is typically used when an LSP client needs to notify the server about opening or changing a text document, as the `TextDocumentItem` in LSP usually requires a `languageId` field.

```go
// import "github.com/opencode-ai/opencode/internal/lsp"
// import "github.com/opencode-ai/opencode/internal/lsp/protocol"

filePath := "/path/to/myproject/main.go"
fileURI := "file://" + filePath // Example URI

languageID := lsp.DetectLanguageID(fileURI)
// languageID would be protocol.LangGo

// When sending a textDocument/didOpen notification:
// params := protocol.DidOpenTextDocumentParams{
//     TextDocument: protocol.TextDocumentItem{
//         URI:        protocol.DocumentUri(fileURI),
//         LanguageID: languageID, // Use the detected language ID
//         Version:    1,
//         Text:       fileContent,
//     },
// }
// lspClient.Notify(ctx, "textDocument/didOpen", params)
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/lsp/protocol`: Heavily relies on this package for the `protocol.LanguageKind` type and all the language identifier constants (e.g., `protocol.LangGo`, `protocol.LangPython`).
- **External Libraries:**
    - `path/filepath`: For `filepath.Ext()` to extract file extensions.
    - `strings`: For `strings.ToLower()`.
- **Interactions:**
    - Provides a standardized way to map file extensions to LSP language identifiers.
    - This mapping is crucial for the `lsp.Client` (in `client.go`) when constructing `TextDocumentItem` objects for notifications like `textDocument/didOpen`.
    - The accuracy and completeness of this mapping directly impact the LSP server's ability to understand and correctly process documents of different languages.
    - If a language's extension is not listed, it will be treated as unknown by the LSP server, potentially limiting language-specific features for that file.

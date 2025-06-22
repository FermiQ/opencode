# uri.go (in internal/lsp/protocol)

## Overview

The `uri.go` file, part of the `internal/lsp/protocol` package, defines types and functions for handling Language Server Protocol (LSP) URIs, specifically `DocumentUri` (which are file-scheme URIs) and the more general `URI` (an alias for string). A key aspect of this file is its robust parsing and normalization logic for `DocumentUri`, designed to handle various idiosyncrasies and platform differences, particularly those related to VS Code's URI encoding and Windows file paths.

The copyright notice "Copyright 2023 The Go Authors" suggests that this file, or significant portions of it, may be adapted from or inspired by Go's official LSP tooling (e.g., `gopls`).

## Key Components

### Types
- `DocumentUri (string)`:
    - A type alias for string, specifically representing URIs for documents known to the client editor.
    - LSP specification mandates these are typically `file://` scheme URIs.
    - The type has custom unmarshalling logic (`UnmarshalText`) to handle inconsistencies in how clients (especially VS Code) might encode these URIs (e.g., double vs. triple slashes, inconsistent drive letter casing or encoding on Windows).
- `URI (string)`:
    - A type alias for string, representing any generic URI (e.g., `http://`, `https://`, `file://`). `DocumentUri` is a specialized form of `URI`.

### Methods on `DocumentUri`
- `(uri *DocumentUri) UnmarshalText(data []byte) error`:
    - Implements `encoding.TextUnmarshaler`. This method is called when `DocumentUri` values are decoded from JSON (or other text-based formats).
    - It calls `ParseDocumentUri(string(data))` to perform normalization and parsing.
- `(uri DocumentUri) Path() string`:
    - Converts a `DocumentUri` (which must be a valid `file://` URI) into a native OS file path (e.g., `/path/to/file` on Unix, `C:\path\to\file` on Windows).
    - It uses the internal `filename()` helper for the core conversion logic.
    - Panics if the `DocumentUri` is not a valid filename (though this should be rare for URIs that have passed through `ParseDocumentUri`).
- `(uri DocumentUri) Dir() DocumentUri`:
    - Returns a new `DocumentUri` representing the directory containing the file identified by the receiver `uri`.
    - It achieves this by getting the `uri.DirPath()` and then converting that path back to a `DocumentUri` using `URIFromPath()`.
- `(uri DocumentUri) DirPath() string`:
    - Returns the native OS file path of the directory containing the file identified by the `uri`. It calls `uri.Path()` and then `filepath.Dir()` on the result.

### Helper Functions
- `filename(uri DocumentUri) (string, error)`:
    - Internal helper to convert a `DocumentUri` to a "filename" string (which is a slash-separated path, not necessarily an OS-native path yet).
    - Handles empty URIs.
    - Contains an optimization for common POSIX `file:///` URIs to avoid `url.ParseRequestURI` overhead.
    - For other cases or complex URIs, it uses `url.ParseRequestURI`.
    - Enforces that the scheme must be "file".
    - Normalizes Windows drive letters in paths (e.g., `/c:/...` becomes `/C:/...`).
- `ParseDocumentUri(s string) (DocumentUri, error)`:
    - The core parsing and normalization function for `DocumentUri` strings.
    - Returns an error if the scheme is not "file".
    - **VS Code Workaround**: Corrects `file://` URIs (two slashes) to the standard `file:///` (three slashes).
    - **Canonicalization**: Unescapes and then re-encodes the path component of the URI to ensure a canonical form, addressing over-escaping issues from some clients.
    - **Windows Drive Letter Normalization**: Converts lowercase drive letters in `file:///c:/...` style paths to uppercase (e.g., `file:///C:/...`).
    - Returns the normalized `DocumentUri`.
- `URIFromPath(path string) DocumentUri`:
    - Converts a native OS file `path` into a `DocumentUri` (`file://...` scheme).
    - If the path is not absolute (and not a Windows drive path starting with a letter and colon), it attempts to make it absolute.
    - Normalizes Windows drive paths to include a leading slash and uppercase drive letter (e.g., `C:\foo` becomes `/C:/foo` in the URL path part).
    - Converts path separators to slashes using `filepath.ToSlash()`.
    - Constructs and returns a `file://` URI.
- `isWindowsDrivePath(path string) bool`:
    - Checks if a given path string matches the Windows drive path pattern (e.g., "C:\...").
- `isWindowsDriveURIPath(uri string) bool`:
    - Checks if a URI's path component (after `file://`) matches the Windows drive URI path pattern (e.g., "/C:/...").

### Constants
- `fileScheme ("file")`: Constant for the "file" URI scheme.

## Important Variables/Constants
- `fileScheme`: Used consistently for constructing and validating file URIs.

## Usage Examples

Parsing a potentially non-standard DocumentUri string:
```go
// import "github.com/opencode-ai/opencode/internal/lsp/protocol"

uriStr := "file://c%3A/Users/jdoe/file.txt" // Example of an over-escaped VS Code URI
docURI, err := protocol.ParseDocumentUri(uriStr)
if err != nil {
    // Handle error
}
// docURI will be normalized, e.g., "file:///C:/Users/jdoe/file.txt"
// fmt.Println(docURI.Path()) // Output: C:\Users\jdoe\file.txt (on Windows)
```

Converting an OS path to a DocumentUri:
```go
// import "github.com/opencode-ai/opencode/internal/lsp/protocol"

osPath := "/usr/local/bin/mytool"
docURI := protocol.URIFromPath(osPath)
// docURI will be "file:///usr/local/bin/mytool"
```

These functions are critical when the LSP client receives URIs from a server or needs to send URIs to a server, ensuring that they are in a consistent, canonical format that both sides can understand, despite potential client-side quirks or platform differences.

## Dependencies and Interactions

- **Internal Dependencies:** None beyond other types within the `protocol` package that might embed `DocumentUri` or `URI`.
- **External Libraries:**
    - `fmt`, `net/url`, `path/filepath`, `strings`, `unicode`: Standard Go libraries.
- **Interactions:**
    - This file provides robust URI handling tailored to LSP's requirements and common client behaviors (especially VS Code).
    - The `DocumentUri.UnmarshalText` method is automatically invoked by `encoding/json` when decoding LSP messages containing `DocumentUri` fields, ensuring incoming URIs are normalized.
    - `URIFromPath` is used when the client needs to construct a `DocumentUri` from a local file path to send to the server (e.g., in `textDocument/didOpen` notifications).
    - `DocumentUri.Path()` is used when the client receives a `DocumentUri` and needs to operate on the corresponding local file.
    - Correctly handling URI encoding, drive letters, and path separators is essential for interoperability between LSP clients and servers, especially across different operating systems.

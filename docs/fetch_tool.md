# fetch.go (in internal/llm/tools)

## Overview

The `fetch.go` file, part of the `internal/llm/tools` package, implements the "fetch" tool. This tool enables an AI agent to retrieve content from a given URL. It supports fetching content and returning it as plain text, Markdown, or raw HTML. The tool includes features like configurable timeouts, response size limits, and basic content type handling for HTML to text/Markdown conversion.

## Key Components

### Structs
- `FetchParams`: Defines the JSON parameters for the fetch tool.
    - `URL (string)`: The URL to fetch content from.
    - `Format (string)`: Desired output format ("text", "markdown", or "html").
    - `Timeout (int, omitempty)`: Optional timeout in seconds for the HTTP request.
- `FetchPermissionsParams`: Struct used for permission requests, mirroring `FetchParams`.
- `fetchTool`: Implements the `BaseTool` interface.
    - `client (*http.Client)`: An HTTP client used for making requests. Initialized with a default timeout.
    - `permissions (permission.Service)`: Service for handling user permissions before fetching a URL.

### Constants
- `FetchToolName ("fetch")`: The registered name of the tool.
- `fetchToolDescription (string)`: A detailed description for the LLM on when and how to use this tool, its features (output formats, redirects, timeouts, validation), limitations (max response size, HTTP/S only, no auth/cookies, potential blocking), and tips for choosing formats.

### Functions
- `NewFetchTool(permissions permission.Service) BaseTool`: Constructor for `fetchTool`. Initializes an `http.Client` with a default 30-second timeout.
- `(t *fetchTool) Info() ToolInfo`: Returns metadata about the tool, including its name, the detailed `fetchToolDescription`, and parameter schema (URL, format, timeout).
- `(t *fetchTool) Run(ctx context.Context, call ToolCall) (ToolResponse, error)`: The core logic when the fetch tool is invoked.
    1.  Parses `call.Input` into `FetchParams`.
    2.  Validates parameters: `URL` is required, `Format` must be one of "text", "markdown", "html", and `URL` must start with "http://" or "https://".
    3.  Requests permission via `t.permissions.Request()` to fetch the specified URL.
    4.  If permission is denied, returns an error.
    5.  Sets up an `http.Client` with the specified or default timeout (max 120 seconds).
    6.  Creates an HTTP GET request with context and sets a "User-Agent" header.
    7.  Executes the request using `client.Do()`.
    8.  Checks for non-OK HTTP status codes.
    9.  Reads the response body, limiting it to a maximum size (5MB).
    10. Based on the requested `format` and the response's `Content-Type`:
        - **text**: If HTML, extracts text using `extractTextFromHTML`. Otherwise, returns raw body.
        - **markdown**: If HTML, converts to Markdown using `convertHTMLToMarkdown`. Otherwise, wraps raw body in Markdown code fences.
        - **html**: Returns raw body.
    11. Returns the processed content as a `NewTextResponse`.
- `extractTextFromHTML(html string) (string, error)`: Uses `goquery` to parse HTML and extract all text content, then cleans it up by collapsing whitespace.
- `convertHTMLToMarkdown(html string) (string, error)`: Uses `JohannesKaufmann/html-to-markdown` to convert an HTML string to Markdown.

## Important Variables/Constants
- `FetchToolName`: The registered name.
- `fetchToolDescription`: Provides crucial instructions to the LLM.

## Usage Examples

This tool is invoked by an LLM.

LLM wants to fetch the text content of a webpage:
```json
{
  "type": "tool_use",
  "id": "tool_fetch_1",
  "name": "fetch",
  "input": {
    "url": "https://example.com/article.html",
    "format": "text"
  }
}
```
`fetchTool.Run` would:
1. Request permission to fetch `https://example.com/article.html`.
2. If approved, make an HTTP GET request.
3. If the response is HTML, extract plain text from it.
4. Return the extracted text to the LLM.

LLM wants to get Markdown from a documentation page:
```json
{
  "type": "tool_use",
  "id": "tool_fetch_2",
  "name": "fetch",
  "input": {
    "url": "https://pkg.go.dev/net/http",
    "format": "markdown",
    "timeout": 10
  }
}
```
`fetchTool.Run` would:
1. Request permission.
2. Fetch the URL with a 10-second timeout.
3. Convert the HTML content to Markdown.
4. Return the Markdown to the LLM.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.WorkingDirectory()` (used in permission request path, though less direct for a URL fetch).
    - `github.com/opencode-ai/opencode/internal/permission`: For `permission.Service` to request user approval.
    - Relies on types from the parent `tools` package (`ToolCall`, `ToolResponse`, etc.).
- **External Libraries:**
    - `github.com/JohannesKaufmann/html-to-markdown`: For converting HTML to Markdown.
    - `github.com/PuerkitoBio/goquery`: For parsing HTML and extracting text (similar to jQuery).
    - `encoding/json`, `fmt`, `io`, `net/http`, `strings`, `time`, `context`: Standard Go libraries.
- **Interactions:**
    - Makes external HTTP requests to arbitrary URLs specified by the LLM.
    - Requires user permission before making any network request.
    - Performs content transformation (HTML to text/Markdown) based on the requested format and detected content type.
    - Handles basic error conditions like network failures, non-200 status codes, and timeouts.

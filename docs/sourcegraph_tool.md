# sourcegraph.go (in internal/llm/tools)

## Overview

The `sourcegraph.go` file, part of the `internal/llm/tools` package, implements the "sourcegraph" tool. This tool allows an AI agent to search for code across public repositories using Sourcegraph's GraphQL API. It's designed to help find code examples, research solutions, and discover patterns in open-source projects. The tool provides a rich query syntax and formats the results for the agent.

## Key Components

### Structs
- `SourcegraphParams`: Defines the JSON parameters for the Sourcegraph tool.
    - `Query (string)`: The Sourcegraph search query.
    - `Count (int, omitempty)`: Optional number of results to return (default: 10, max: 20).
    - `ContextWindow (int, omitempty)`: Optional number of lines of context to show around a match (default: 10).
    - `Timeout (int, omitempty)`: Optional timeout in seconds for the API request.
- `SourcegraphResponseMetadata`: Struct for metadata about the Sourcegraph search operation.
    - `NumberOfMatches (int)`: Number of matches found (Note: this seems to be based on `resultCount` from API, not individual line matches).
    - `Truncated (bool)`: Indicates if the API reported that the result limit was hit.
- `sourcegraphTool`: Implements the `BaseTool` interface.
    - `client (*http.Client)`: An HTTP client for making requests to the Sourcegraph API.

### Constants
- `SourcegraphToolName ("sourcegraph")`: The registered name of the tool.
- `sourcegraphToolDescription (string)`: An extremely detailed description for the LLM on how and when to use this tool. It includes:
    - Purpose: Searching public code repositories.
    - Usage: Provide a query, optional count, and timeout.
    - **Extensive Query Syntax Guide**: Covers basic search, file filters, repository filters, language filters, boolean operators, regular expressions, quoted strings, exclude filters.
    - **Advanced Filters**: Detailed explanations for repository, file, content, and type filters (symbol, file, path, diff, commit).
    - **Commit/Diff Search**: Filters like `after`, `before`, `author`, `message`.
    - **Result Selection & Control**: `select:*` options, `count`, `timeout`.
    - **Numerous Examples**: Illustrates various complex queries.
    - **Boolean Operators**: `AND`, `OR`, `NOT`, grouping.
    - **Limitations**: Public repos only, rate limits, max 20 results per query via this tool.
    - **Tips**: Use specific extensions, repo filters, `type:symbol`.

### Functions
- `NewSourcegraphTool() BaseTool`: Constructor for `sourcegraphTool`. Initializes an `http.Client` with a default 30-second timeout.
- `(t *sourcegraphTool) Info() ToolInfo`: Returns metadata about the tool.
- `(t *sourcegraphTool) Run(ctx context.Context, call ToolCall) (ToolResponse, error)`: The core logic when the Sourcegraph tool is invoked.
    1.  Parses `call.Input` into `SourcegraphParams`.
    2.  Validates `Query` is provided. Sets default `Count` (10, max 20) and `ContextWindow` (10) if not specified.
    3.  Configures an `http.Client` with the specified or default timeout (max 120 seconds).
    4.  Constructs a GraphQL query string. The query requests fields like `matchCount`, `limitHit`, `results` (including repository name, file path, URL, content, and line matches with preview, line number).
    5.  Marshals the GraphQL request into JSON.
    6.  Makes a POST request to `https://sourcegraph.com/.api/graphql`.
    7.  Handles HTTP errors and reads the response body.
    8.  Unmarshals the JSON response into a `map[string]any`.
    9.  Calls `formatSourcegraphResults()` to process and format the results.
    10. Returns the formatted string as a `NewTextResponse`.
- `formatSourcegraphResults(result map[string]any, contextWindow int) (string, error)`:
    - Parses the complex JSON structure returned by the Sourcegraph GraphQL API.
    - Extracts `matchCount`, `resultCount`, and `limitHit` flags.
    - Formats the output into a human-readable (and LLM-parsable) Markdown-like string:
        - Starts with a header "# Sourcegraph Search Results" and summary counts.
        - Iterates through the top results (up to `maxResults`, currently 10, even if API returned up to 20).
        - For each `FileMatch`:
            - Prints repository name and file path.
            - Prints the file URL.
            - For each `lineMatch`:
                - If full file content is available in the response, it prints the matching line with `contextWindow` lines before and after, prefixed by line numbers.
                - If full content is not available, it prints just the preview line with its line number.
    - Handles API errors reported in the `errors` field of the GraphQL response.
    - Returns the formatted string.

## Important Variables/Constants
- `SourcegraphToolName`: The registered name.
- `sourcegraphToolDescription`: Contains a very rich set of instructions and examples for the LLM.

## Usage Examples

This tool is invoked by an LLM.

LLM wants to find Go code using `context.WithDeadline`:
```json
{
  "type": "tool_use",
  "id": "tool_sg_1",
  "name": "sourcegraph",
  "input": {
    "query": "lang:go context.WithDeadline"
  }
}
```
`sourcegraphTool.Run` would:
1. Construct the appropriate GraphQL query.
2. Call the Sourcegraph API.
3. Parse the JSON response.
4. Format the results, showing file paths, repository names, and snippets of matching lines with context.
5. Return this formatted string to the LLM.

## Dependencies and Interactions

- **Internal Dependencies:**
    - Relies on types from the parent `tools` package.
- **External Libraries:**
    - `bytes`, `context`, `encoding/json`, `fmt`, `io`, `net/http`, `strings`, `time`: Standard Go libraries.
- **Interactions:**
    - Makes external HTTP POST requests to the public Sourcegraph GraphQL API (`https://sourcegraph.com/.api/graphql`).
    - The tool's effectiveness relies on the LLM's ability to construct valid and effective Sourcegraph search queries based on the extensive `sourcegraphToolDescription`.
    - It processes a potentially complex JSON response from the API and formats it into a more structured textual representation.
    - The tool itself limits results displayed to the LLM (e.g., top 10 file matches) even if the API returns more, to manage token usage.

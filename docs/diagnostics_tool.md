# diagnostics.go (in internal/llm/tools)

## Overview

The `diagnostics.go` file, part of the `internal/llm/tools` package, implements the "diagnostics" tool. This tool allows an AI agent to retrieve and display code diagnostics (errors, warnings, hints) for a specific file or for the entire project. It leverages connected Language Server Protocol (LSP) clients to gather this information. The output is structured and formatted for clarity, often using XML-like tags.

## Key Components

### Structs
- `DiagnosticsParams`: Defines the JSON parameters for the diagnostics tool.
    - `FilePath (string)`: The path to the file for which to get diagnostics. If empty, project-wide diagnostics are fetched.
- `diagnosticsTool`: Implements the `BaseTool` interface.
    - `lspClients (map[string]*lsp.Client)`: A map of active LSP clients, keyed by a name or identifier, which are queried for diagnostics.

### Constants
- `DiagnosticsToolName ("diagnostics")`: The registered name of the tool.
- `diagnosticsDescription (string)`: A detailed description for the LLM on when and how to use this tool, its features, limitations, and tips. It highlights using it for checking errors/warnings, debugging, and getting an overview of code quality.

### Functions
- `NewDiagnosticsTool(lspClients map[string]*lsp.Client) BaseTool`: Constructor for `diagnosticsTool`.
- `(b *diagnosticsTool) Info() ToolInfo`: Returns metadata about the tool, including its name, the detailed `diagnosticsDescription`, and parameter schema.
- `(b *diagnosticsTool) Run(ctx context.Context, call ToolCall) (ToolResponse, error)`: The core logic when the diagnostics tool is invoked.
    1.  Parses `call.Input` into `DiagnosticsParams`.
    2.  Checks if any LSP clients are available; if not, returns an error.
    3.  If `params.FilePath` is provided:
        - Calls `notifyLspOpenFile` to ensure all LSP clients are aware of the file (sends `textDocument/didOpen` or `textDocument/didChange`).
        - Calls `waitForLspDiagnostics` to give LSP servers a chance to analyze the file and publish diagnostics.
    4.  Calls `getDiagnostics` to collect and format diagnostics from all LSP clients.
    5.  Returns the formatted diagnostics string as a `NewTextResponse`.
- `notifyLspOpenFile(ctx context.Context, filePath string, lsps map[string]*lsp.Client)`: Iterates through LSP clients and calls `client.OpenFile(ctx, filePath)` for each to ensure they have processed the file.
- `waitForLspDiagnostics(ctx context.Context, filePath string, lsps map[string]*lsp.Client)`:
    - Sets up a mechanism to wait for diagnostic updates from LSP clients, specifically for the given `filePath` or if any project diagnostics change.
    - It registers a temporary notification handler for `textDocument/publishDiagnostics` for each LSP client.
    - If an LSP client already has the file open, it sends a `textDocument/didChange` notification to potentially trigger re-analysis. Otherwise, it ensures the file is opened.
    - It waits on a channel (`diagChan`) that is signaled when new diagnostics arrive, or for a timeout (5 seconds), or if the context is cancelled.
- `hasDiagnosticsChanged(current, original map[protocol.DocumentUri][]protocol.Diagnostic) bool`: Compares two sets of diagnostics to see if there's a change (used by `waitForLspDiagnostics`).
- `getDiagnostics(filePath string, lsps map[string]*lsp.Client) string`:
    - Collects diagnostics from all provided `lspClients`.
    - Separates diagnostics into `fileDiagnostics` (for the specified `filePath`) and `projectDiagnostics` (for all other files).
    - Formats each diagnostic item using `formatDiagnostic`, including severity, location, source, code, and message.
    - Sorts both `fileDiagnostics` and `projectDiagnostics` (errors first, then alphabetically).
    - Constructs an output string:
        - If file-specific diagnostics exist, they are included within `<file_diagnostics>` tags. Output is truncated if more than 10 diagnostics.
        - If project-wide diagnostics exist, they are included within `<project_diagnostics>` tags. Output is truncated if more than 10 diagnostics.
        - A `<diagnostic_summary>` section is added, showing counts of errors and warnings for both the current file and the project.
- `formatDiagnostic(pth string, diagnostic protocol.Diagnostic, source string) string`: Helper to format a single `protocol.Diagnostic` item into a human-readable string.
- `countSeverity(diagnostics []string, severity string) int`: Counts the number of diagnostics of a specific severity within a list of formatted diagnostic strings.

## Important Variables/Constants
- `DiagnosticsToolName`: The registered name for this tool.
- `diagnosticsDescription`: Provides detailed instructions to the LLM.

## Usage Examples

This tool is invoked by an LLM when it needs to assess the correctness or quality of code.

LLM wants to check diagnostics for `src/main.go`:
```json
{
  "type": "tool_use",
  "id": "tool_call_diag_file",
  "name": "diagnostics",
  "input": { "file_path": "src/main.go" }
}
```
`diagnosticsTool.Run` would:
1. Notify LSP clients about `src/main.go`.
2. Wait briefly for diagnostics to be published.
3. Collect diagnostics from all LSP clients.
4. Format them into a string like:
   ```
   <file_diagnostics>
   Error: src/main.go:10:5 [compiler] undefined: someVariable
   Warn: src/main.go:15:2 [linter] unused import: os
   </file_diagnostics>

   <project_diagnostics>
   Error: src/utils.go:5:1 [compiler] package imported but not used: strings
   </project_diagnostics>

   <diagnostic_summary>
   Current file: 1 errors, 1 warnings
   Project: 1 errors, 0 warnings
   </diagnostic_summary>
   ```
5. Return this string to the LLM.

LLM wants project-wide diagnostics:
```json
{
  "type": "tool_use",
  "id": "tool_call_diag_project",
  "name": "diagnostics",
  "input": { "file_path": "" }
}
```
This would skip file-specific notifications and collect all available diagnostics.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/lsp`: For `lsp.Client` and `lsp.HandleDiagnostics`.
    - `github.com/opencode-ai/opencode/internal/lsp/protocol`: For LSP data structures like `protocol.Diagnostic`, `protocol.DocumentUri`, `protocol.PublishDiagnosticsParams`, and severity constants.
    - Relies on `ToolCall`, `ToolResponse`, `BaseTool`, `ToolInfo` types from the parent `tools` package.
- **External Libraries:**
    - `encoding/json`: For parsing input parameters and LSP message parameters.
    - `maps` (standard library): For `maps.Copy`.
    - `sort`, `strings`, `time`, `context`, `fmt`: Standard Go libraries.
- **Interactions:**
    - This tool directly interacts with the registered LSP clients (`lsp.Client`).
    - It relies on LSP servers to publish diagnostics via the `textDocument/publishDiagnostics` notification.
    - The `waitForLspDiagnostics` function attempts to ensure that diagnostics are reasonably up-to-date before being collected, but it uses a timeout to avoid blocking indefinitely.
    - The output format is structured with XML-like tags, which the LLM is expected to parse or understand.
    - Provides a crucial capability for the agent to understand code health and potential issues.

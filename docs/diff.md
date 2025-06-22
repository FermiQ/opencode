# diff.go

## Overview

The `diff.go` file in the `internal/diff` package provides comprehensive functionality for working with text differences. It can generate diffs between two text inputs, parse existing unified diffs, perform intra-line highlighting of changes, and render diffs in a side-by-side format with syntax highlighting. This package is crucial for displaying changes to code or text within the application's TUI.

## Key Components

### Core Types
- `LineType`: An enum (`LineContext`, `LineAdded`, `LineRemoved`) indicating the type of a diff line.
- `Segment`: Represents a portion of a line with a specific `LineType`, used for intra-line highlighting. Fields: `Start`, `End`, `Type`, `Text`.
- `DiffLine`: Represents a single line in a diff. Fields: `OldLineNo`, `NewLineNo`, `Kind (LineType)`, `Content`, `Segments ([]Segment)`.
- `Hunk`: Represents a block of changes in a diff. Fields: `Header` (the `@@ ... @@` line), `Lines ([]DiffLine)`.
- `DiffResult`: Contains the parsed result of a diff. Fields: `OldFile` (name), `NewFile` (name), `Hunks ([]Hunk)`.
- `linePair`: Internal struct to hold a pair of `DiffLine`s for side-by-side rendering.

### Configuration Structs and Options
- `ParseConfig`: Configuration for diff parsing.
    - `ContextSize`: Number of context lines (currently not used by `ParseUnifiedDiff`).
- `ParseOption`: Functional option type for `ParseConfig`.
    - `WithContextSize(size int)`: Sets context size.
- `SideBySideConfig`: Configuration for rendering side-by-side diffs.
    - `TotalWidth`: Total width for the side-by-side view.
- `SideBySideOption`: Functional option type for `SideBySideConfig`.
    - `NewSideBySideConfig(opts ...SideBySideOption)`: Constructor with default values.
    - `WithTotalWidth(width int)`: Sets total width.

### Diff Parsing and Processing
- `ParseUnifiedDiff(diff string) (DiffResult, error)`: Parses a unified diff format string into a `DiffResult` struct. It identifies file headers (`--- a/...`, `+++ b/...`) and hunk headers (`@@ ... @@`) to structure the diff.
- `HighlightIntralineChanges(h *Hunk)`: Modifies a `Hunk`'s lines to include `Segment` data for character-level (intra-line) differences. It compares adjacent removed/added lines using `diffmatchpatch` library.
- `pairLines(lines []DiffLine) []linePair`: Converts a flat list of `DiffLine`s into a slice of `linePair`s, suitable for side-by-side rendering. It groups context lines, removed lines, added lines, and changed lines (removed followed by added).

### Syntax Highlighting
- `SyntaxHighlight(w io.Writer, source, fileName, formatter string, bg lipgloss.TerminalColor) error`: Applies syntax highlighting to the `source` text based on `fileName`'s extension.
    - Uses `chroma` library for lexing and formatting.
    - Dynamically generates a `chroma` XML style based on the application's current `theme.CurrentTheme()`. This involves mapping theme colors to various `chroma` token types.
    - `formatter` specifies the output format (e.g., "terminal16m").
    - `bg` is the background color for the line, used to ensure contrast.
- `getColor(adaptiveColor lipgloss.AdaptiveColor) string`: Helper to get the dark or light variant of a `lipgloss.AdaptiveColor` based on `lipgloss.HasDarkBackground()`.
- `highlightLine(fileName string, line string, bg lipgloss.TerminalColor) string`: A convenience function to syntax highlight a single line.
- `createStyles(t theme.Theme) (removedLineStyle, addedLineStyle, contextLineStyle, lineNumberStyle lipgloss.Style)`: Creates `lipgloss` styles for different diff line types and line numbers based on the provided theme.

### Rendering
- `applyHighlighting(content string, segments []Segment, segmentType LineType, highlightBg lipgloss.AdaptiveColor) string`: Takes a syntax-highlighted line (`content`) and its `segments` to apply further background highlighting for intra-line changes (e.g., highlighting the specific changed words in a different background color). It carefully handles existing ANSI escape codes from syntax highlighting.
- `renderLeftColumn(fileName string, dl *DiffLine, colWidth int) string`: Renders the left column for a side-by-side diff line. Includes line number, diff marker (`-` or space), and syntax/intra-line highlighted content.
- `renderRightColumn(fileName string, dl *DiffLine, colWidth int) string`: Renders the right column for a side-by-side diff line. Includes line number, diff marker (`+` or space), and syntax/intra-line highlighted content.
- `RenderSideBySideHunk(fileName string, h Hunk, opts ...SideBySideOption) string`: Renders a single `Hunk` in side-by-side format. It calls `HighlightIntralineChanges`, `pairLines`, and then iterates through pairs calling `renderLeftColumn` and `renderRightColumn`.
- `FormatDiff(diffText string, opts ...SideBySideOption) (string, error)`: Parses a full `diffText` and formats all its hunks using `RenderSideBySideHunk`.
- `GenerateDiff(beforeContent, afterContent, fileName string) (string, int, int)`: Generates a unified diff string between `beforeContent` and `afterContent`. It also counts and returns the number of added and removed lines. Uses `aymanbagabas/go-udiff` for generating the diff.

## Important Variables/Constants
This file primarily defines types and functions. No significant exported package-level constants or variables are defined, other than the `LineType` enum values.

## Usage Examples

Generating a diff:
```go
before := "Hello world"
after := "Hello Go!"
fileName := "example.txt"
diffString, additions, removals := diff.GenerateDiff(before, after, fileName)
// diffString now contains the unified diff
```

Parsing and formatting a diff for display:
```go
// Assume diffString contains a unified diff
formattedOutput, err := diff.FormatDiff(diffString, diff.WithTotalWidth(120))
if err != nil {
    // Handle error
}
// formattedOutput can now be printed to the terminal
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.WorkingDirectory()` when normalizing file paths in `GenerateDiff`.
    - `github.com/opencode-ai/opencode/internal/tui/theme`: For accessing the current theme colors for syntax highlighting and diff rendering.
- **External Libraries:**
    - `github.com/alecthomas/chroma/v2`: For syntax highlighting.
    - `github.com/aymanbagabas/go-udiff`: For generating unified diffs.
    - `github.com/charmbracelet/lipgloss`: For styling terminal output (colors, backgrounds, layout).
    - `github.com/charmbracelet/x/ansi`: For ANSI string manipulation like truncation.
    - `github.com/sergi/go-diff/diffmatchpatch`: For character-level diffing used in `HighlightIntralineChanges`.
- **Interactions:**
    - Provides a complete pipeline for diff generation, parsing, and rendering.
    - Relies heavily on the current TUI theme for all visual styling.
    - Interacts with string manipulation and regular expressions for parsing diff formats.
    - Produces ANSI-styled strings suitable for display in a terminal.

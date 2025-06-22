# format.go

## Overview

The `format.go` file in the `internal/format` package is responsible for handling different output formats for the application, specifically when running in non-interactive mode (e.g., when a prompt is passed directly via a CLI flag). It defines supported output formats (Text and JSON) and provides functions to parse, validate, and apply these formats to a given content string.

## Key Components

### Types
- `OutputFormat`: A string type representing the output format. Constants `Text` and `JSON` are defined for this type.

### Constants
- `Text (OutputFormat)`: Represents plain text output. This is typically the default.
- `JSON (OutputFormat)`: Represents output formatted as a JSON object.

### Variables
- `SupportedFormats ([]string)`: A slice of strings listing all supported output format names (e.g., `["text", "json"]`). Used for help messages or validation.

### Functions
- `(f OutputFormat) String() string`: A method on `OutputFormat` that returns its string representation.
- `Parse(s string) (OutputFormat, error)`:
    - Converts a raw string `s` (case-insensitive, trimmed whitespace) into an `OutputFormat` type.
    - Returns the corresponding `OutputFormat` (e.g., `Text`, `JSON`) if `s` is a valid format name.
    - Returns an error if `s` is not a recognized format.
- `IsValid(s string) bool`:
    - Checks if the provided format string `s` is a supported output format by attempting to `Parse` it.
    - Returns `true` if valid, `false` otherwise.
- `GetHelpText() string`:
    - Returns a user-friendly string describing the supported output formats and their purpose. This is likely used in CLI help messages.
- `FormatOutput(content string, formatStr string) string`:
    - Takes a raw `content` string (e.g., AI response) and a `formatStr` (e.g., "json", "text").
    - Parses `formatStr` using `Parse()`. If parsing fails or an invalid format is given, it defaults to plain text output (returns `content` as is).
    - If `formatStr` is `JSON`, it calls `formatAsJSON(content)`.
    - If `formatStr` is `Text` (or any other valid format that defaults to text), it returns `content` directly.
- `formatAsJSON(content string) string`:
    - Takes a `content` string and wraps it into a JSON object of the structure `{"response": "content_value"}`.
    - It uses `json.MarshalIndent` for proper JSON formatting and escaping.
    - Includes a fallback to manual string replacement for JSON escaping if `json.MarshalIndent` fails, ensuring some form of JSON output is still provided.

## Important Variables/Constants
- `Text`, `JSON`: Define the recognized output formats.
- `SupportedFormats`: Provides an easy way to list available formats.

## Usage Examples

Parsing a format string from user input:
```go
userInput := "json" // or "text"
format, err := format.Parse(userInput)
if err != nil {
    // Handle invalid format error
    fmt.Println(format.GetHelpText())
}
```

Formatting output content:
```go
aiResponse := "This is a response from the AI."
chosenFormat := "json" // Could be from CLI args

formattedString := format.FormatOutput(aiResponse, chosenFormat)
fmt.Println(formattedString)
// Output for "json" would be:
// {
//   "response": "This is a response from the AI."
// }
// Output for "text" would be:
// This is a response from the AI.
```

Validating a format:
```go
if !format.IsValid("xml") { // "xml" is not a supported format
    fmt.Println("Invalid format specified.")
    fmt.Println(format.GetHelpText())
}
```

## Dependencies and Interactions

- **Internal Dependencies:** None beyond standard Go packages.
- **External Libraries:**
    - `encoding/json`: Standard Go library for JSON marshalling.
    - `strings`: Standard Go library for string manipulation (trimming, lowercasing, replacing).
    - `fmt`: Standard Go library for formatting strings and errors.
- **Interactions:**
    - This package is primarily used by the CLI handling logic (`cmd/root.go`) when the application is run in non-interactive mode with an output format specified.
    - It ensures that the output presented to the user or piped to another process conforms to the requested format.
    - The `GetHelpText` function contributes to the CLI's user-facing documentation.

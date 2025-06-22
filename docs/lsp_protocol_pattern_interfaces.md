# pattern_interfaces.go (in internal/lsp/protocol)

## Overview

The `pattern_interfaces.go` file, part of the `internal/lsp/protocol` package, defines an interface `PatternInfo` and related helper types (`StringPattern`, `RelativePatternInfo`). This abstraction is designed to handle different ways glob patterns can be represented in the Language Server Protocol. Specifically, an LSP `GlobPattern` can be a simple string or a `RelativePattern` struct (which includes a base URI and the pattern string). This file provides a unified way to access the pattern string and an optional base path.

## Key Components

### Interfaces
- `PatternInfo`:
    - An interface for types that represent glob patterns.
    - Methods:
        - `GetPattern() string`: Returns the actual glob pattern string.
        - `GetBasePath() string`: Returns the base path for the pattern, if applicable (primarily for `RelativePattern`). Returns an empty string for simple string patterns.
        - `isPattern()`: A marker method to identify types that satisfy this interface.

### Structs (Helper Types implementing `PatternInfo`)
- `StringPattern`:
    - Implements `PatternInfo` for simple string-based glob patterns.
    - Fields:
        - `Pattern (string)`: The glob pattern string.
    - `GetPattern()` returns `p.Pattern`.
    - `GetBasePath()` returns `""`.
- `RelativePatternInfo`:
    - Implements `PatternInfo` for LSP `RelativePattern` types.
    - Fields:
        - `RP (RelativePattern)`: The underlying `RelativePattern` struct (which contains the pattern string and a `BaseURI` that can be a string or `DocumentUri`).
        - `BasePath (string)`: The processed base path, extracted from `RP.BaseURI` and with "file://" prefix removed.
    - `GetPattern()` returns `string(p.RP.Pattern)`.
    - `GetBasePath()` returns `p.BasePath`.

### Methods on Concrete LSP Types
- `(g *GlobPattern) AsPattern() (PatternInfo, error)`:
    - This is a method on the `GlobPattern` type (which is likely an `Or_...` union type that can hold either a `string` or a `RelativePattern`).
    - It converts a `GlobPattern` into a `PatternInfo` interface object.
    - If `g.Value` is a `string`, it returns a `StringPattern`.
    - If `g.Value` is a `RelativePattern`:
        - It extracts the `BaseURI` from the `RelativePattern`. The `BaseURI` itself can be an `Or_...` type (string or `DocumentUri`).
        - It processes this `BaseURI` to get a `basePath` string (stripping "file://").
        - It returns a `RelativePatternInfo` containing the original `RelativePattern` and the processed `basePath`.
    - Returns an error if the underlying type of `GlobPattern` or `BaseURI` is unknown or nil.

## Important Variables/Constants
This file primarily defines interfaces, helper structs, and a conversion method. It does not export package-level variables or constants.

## Usage Examples

This system is used when parts of the LSP client or server need to work with glob patterns received from the other side, and these patterns might come in different structural forms.

```go
// Assume 'lspGlobPattern' is an instance of protocol.GlobPattern
// received from an LSP message (e.g., in DidChangeWatchedFilesRegistrationOptions).

// patternInfo, err := lspGlobPattern.AsPattern()
// if err != nil {
//     // Handle error, e.g., pattern was of an unexpected type
//     log.Printf("Error converting glob pattern: %v", err)
//     return
// }

// actualGlobString := patternInfo.GetPattern()
// basePath := patternInfo.GetBasePath() // Might be empty if it was a simple string pattern

// fmt.Printf("Glob to watch: %s\n", actualGlobString)
// if basePath != "" {
//     fmt.Printf("Relative to base path: %s\n", basePath)
//     // Use filepath.Join(basePath, actualGlobString) for matching if needed,
//     // or pass them separately to a file watcher that understands relative patterns.
// } else {
//     // Use actualGlobString directly, assuming it's relative to a default workspace root.
// }
```
This abstraction allows, for example, a file watcher registration handler (`lsp.HandleRegisterCapability`) to uniformly extract pattern details regardless of whether the server sent a simple string or a `RelativePattern`.

## Dependencies and Interactions

- **Internal Dependencies:**
    - Relies on LSP types defined elsewhere in the `internal/lsp/protocol` package:
        - `GlobPattern` (likely an `Or_String_RelativePattern` type).
        - `RelativePattern` (struct with `Pattern` and `BaseURI` fields).
        - `DocumentUri` (type alias for string, often representing file URIs).
        - `Location` (not directly used by `PatternInfo` but related to URIs).
- **External Libraries:**
    - `fmt`: For error formatting.
    - `strings`: For `strings.TrimPrefix`.
- **Interactions:**
    - Provides a way to normalize the handling of different representations of glob patterns within the LSP.
    - The `AsPattern()` method is key to converting the potentially complex `GlobPattern` (an `Or_...` type) into a simpler, unified `PatternInfo` interface.
    - This is particularly useful when dealing with `DidChangeWatchedFilesRegistrationOptions`, where `watchers` can contain such glob patterns.
    - The code that consumes `PatternInfo` can then decide how to interpret the pattern and base path (e.g., for setting up filesystem watchers).

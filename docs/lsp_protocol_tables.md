# tables.go (in internal/lsp/protocol)

## Overview

The `tables.go` file in the `internal/lsp/protocol` package defines a single public variable, `TableKindMap`. This map provides a straightforward way to convert Language Server Protocol (LSP) `SymbolKind` integer constants (which represent different types of programming language symbols like classes, functions, variables, etc.) into their human-readable string representations.

## Key Components

### Variables
- `TableKindMap (map[SymbolKind]string)`:
    - A public map where:
        - Keys are of type `SymbolKind` (an integer-based enum defined elsewhere in the `protocol` package, likely in `tsprotocol.go` or a similar generated file). Each `SymbolKind` constant represents a specific type of symbol (e.g., `File`, `Module`, `Class`, `Function`, `Variable`).
        - Values are `string` representations of these kinds (e.g., "File", "Module", "Class", "Function", "Variable").
    - This map covers all standard symbol kinds defined by the LSP specification.

## Important Variables/Constants
- `TableKindMap`: This is the sole and primary component of this file. It acts as a lookup table.

## Usage Examples

This map is useful when displaying symbol information to a user or in logs, where a string representation is more informative than a raw integer kind.

```go
// import "github.com/opencode-ai/opencode/internal/lsp/protocol"
// import "fmt"

// Assume 'symbol' is an instance of protocol.SymbolInformation or protocol.DocumentSymbol
// var symbolKind protocol.SymbolKind = symbol.Kind // .Kind would be of type protocol.SymbolKind

// kindString, ok := protocol.TableKindMap[symbolKind]
// if ok {
//     fmt.Printf("Symbol: %s, Kind: %s\n", symbol.Name, kindString)
// } else {
//     fmt.Printf("Symbol: %s, Kind: Unknown (%d)\n", symbol.Name, symbolKind)
// }

// Example with a specific kind:
// className := protocol.TableKindMap[protocol.Class] // className would be "Class"
```
This allows for consistent string representations of symbol types throughout an application that consumes LSP data.

## Dependencies and Interactions

- **Internal Dependencies:**
    - Relies on the `SymbolKind` type and its associated constants (e.g., `File`, `Module`, `Class`) being defined elsewhere within the `internal/lsp/protocol` package (typically in a file generated from the LSP specification, like `tsprotocol.go`).
- **External Libraries:** None.
- **Interactions:**
    - Provides a utility for converting LSP `SymbolKind` numeric values into meaningful strings.
    - This can be used by any part of the application that processes LSP symbol information and needs to display or log the type of a symbol. For instance, when displaying document outlines or workspace symbol search results.
    - It's a simple lookup table, essentially a static data definition file.

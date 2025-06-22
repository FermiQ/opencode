# layout.go (in internal/tui/layout)

## Overview

The `layout.go` file in the `internal/tui/layout` package defines a set of common interfaces (`Focusable`, `Sizeable`, `Bindings`) that TUI components can implement. These interfaces establish contracts for managing focus, size, and key bindings in a standardized way across different components. The file also includes a utility function `KeyMapToSlice` to convert a struct of `key.Binding` fields into a slice.

## Key Components

### Interfaces

- `Focusable`: Defines methods for components that can gain and lose focus.
    - `Focus() tea.Cmd`: Called when the component gains focus. Can return a `tea.Cmd`.
    - `Blur() tea.Cmd`: Called when the component loses focus. Can return a `tea.Cmd`.
    - `IsFocused() bool`: Returns `true` if the component is currently focused, `false` otherwise.

- `Sizeable`: Defines methods for components whose size can be set and retrieved.
    - `SetSize(width, height int) tea.Cmd`: Sets the component's dimensions. Can return a `tea.Cmd`.
    - `GetSize() (int, int)`: Returns the component's current width and height.

- `Bindings`: Defines a method for components that expose their key bindings.
    - `BindingKeys() []key.Binding`: Returns a slice of `key.Binding`s that the component uses. This is often used for displaying help or managing key conflicts.

### Functions

- `KeyMapToSlice(t any) (bindings []key.Binding)`:
    - A utility function that takes an arbitrary struct `t` as input.
    - It uses reflection (`reflect` package) to iterate over the fields of the struct.
    - If a field is of type `key.Binding` (from `charmbracelet/bubbles/key`), it appends that field's value to the `bindings` slice.
    - Returns a slice of all `key.Binding`s found as fields in the input struct. This is useful for components that define their key maps as struct fields (e.g., `type MyKeyMap struct { Up key.Binding; Down key.Binding }`).

## Important Variables/Constants

This file primarily defines interfaces and a utility function. It does not export package-level variables or constants.

## Usage Examples

Implementing the interfaces in a TUI component:

```go
// import "github.com/charmbracelet/bubbles/key"
// import tea "github.com/charmbracelet/bubbletea"
// import "github.com/opencode-ai/opencode/internal/tui/layout"

type MyCustomComponent struct {
    // ... other fields
    isFocused bool
    width, height int
    keyMap struct { // Example key map
        Enter key.Binding
        Esc   key.Binding
    }
}

// Implement layout.Focusable
func (m *MyCustomComponent) Focus() tea.Cmd { m.isFocused = true; return nil /* or some command */ }
func (m *MyCustomComponent) Blur() tea.Cmd { m.isFocused = false; return nil }
func (m *MyCustomComponent) IsFocused() bool { return m.isFocused }

// Implement layout.Sizeable
func (m *MyCustomComponent) SetSize(w, h int) tea.Cmd { m.width = w; m.height = h; return nil }
func (m *MyCustomComponent) GetSize() (int, int) { return m.width, m.height }

// Implement layout.Bindings
func (m *MyCustomComponent) BindingKeys() []key.Binding {
    // Initialize keyMap if not done
    if m.keyMap.Enter.Help() == nil { // Check if uninitialized
        m.keyMap.Enter = key.NewBinding(key.WithKeys("enter"), key.WithHelp("enter", "submit"))
        m.keyMap.Esc = key.NewBinding(key.WithKeys("esc"), key.WithHelp("esc", "cancel"))
    }
    return layout.KeyMapToSlice(m.keyMap)
}

// ... other tea.Model methods (Init, Update, View) ...
```

A layout manager or a parent component could then use these interfaces:
```go
// var component layout.Focusable // Assume component is of a type that implements Focusable
// component.Focus()

// var sizeableComponent layout.Sizeable
// sizeableComponent.SetSize(100, 20)

// var bindingComponent layout.Bindings
// helpBindings := bindingComponent.BindingKeys() // To display in a help view
```

## Dependencies and Interactions

- **Internal Dependencies:** None from other OpenCode packages beyond what's in the `tui` scope.
- **External Libraries:**
    - `github.com/charmbracelet/bubbles/key`: For the `key.Binding` type.
    - `github.com/charmbracelet/bubbletea`: For `tea.Model` and `tea.Cmd`.
    - `reflect`: Standard Go library, used by `KeyMapToSlice`.
- **Interactions:**
    - These interfaces provide a common contract for TUI components regarding focus management, sizing, and exposing key bindings.
    - Higher-level layout components (like `Split`, `Overlay`, or the main TUI application) can use these interfaces to manage their children without needing to know their concrete types.
    - `KeyMapToSlice` is a reflection-based utility that simplifies collecting `key.Binding`s from a struct, which is a common pattern for organizing key maps in Bubble Tea applications.

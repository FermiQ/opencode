# container.go (in internal/tui/layout)

## Overview

The `container.go` file, part of the `internal/tui/layout` package, defines a `Container` component. This component acts as a wrapper around another `tea.Model` (the `content`). Its primary purpose is to provide layout features like padding and borders to its contained content. It implements `tea.Model` itself, forwarding `Init` and `Update` calls to the content, and uses `lipgloss` for styling its view. It also propagates size changes and key bindings if the content model supports the `Sizeable` and `Bindings` interfaces (defined in `layout.go`).

## Key Components

### Interfaces
- `Container`: The public interface for the container component. It embeds:
    - `tea.Model`: From `charmbracelet/bubbletea`, for standard TUI component behavior.
    - `Sizeable`: (From `layout.go`) For components that can have their size set.
    - `Bindings`: (From `layout.go`) For components that provide key bindings.

### Structs
- `container`: The concrete implementation of the `Container` interface.
    - `width (int)`, `height (int)`: Dimensions of the container.
    - `content (tea.Model)`: The `tea.Model` that this container wraps.
    - `paddingTop (int)`, `paddingRight (int)`, `paddingBottom (int)`, `paddingLeft (int)`: Padding values.
    - `borderTop (bool)`, `borderRight (bool)`, `borderBottom (bool)`, `borderLeft (bool)`: Flags to enable/disable borders on each side.
    - `borderStyle (lipgloss.Border)`: The `lipgloss` border style to use (e.g., `lipgloss.NormalBorder()`, `lipgloss.RoundedBorder()`).

### Methods on `container`
- `Init() tea.Cmd`: Forwards the `Init` call to the `c.content` model.
- `Update(msg tea.Msg) (tea.Model, tea.Cmd)`: Forwards the `Update` call to `c.content` and updates `c.content` with the returned model.
- `View() string`:
    - Gets the current theme using `theme.CurrentTheme()`.
    - Creates a `lipgloss.Style`.
    - Adjusts `width` and `height` based on whether borders are enabled for each side.
    - Applies the border style, background color (from theme), and border foreground color (from theme) if any border is enabled.
    - Sets the style's width, height, and padding.
    - Renders the `c.content.View()` within this styled container.
- `SetSize(width, height int) tea.Cmd`:
    - Sets the container's `width` and `height`.
    - If `c.content` implements the `Sizeable` interface, it calculates the available space for the content (by subtracting padding and border widths/heights) and calls `SetSize` on the content model.
- `GetSize() (int, int)`: Returns the container's current width and height.
- `BindingKeys() []key.Binding`:
    - If `c.content` implements the `Bindings` interface, it returns the content's key bindings.
    - Otherwise, returns an empty slice.

### Constructor and Options
- `ContainerOption func(*container)`: Functional option type for configuring a `container`.
- `NewContainer(content tea.Model, options ...ContainerOption) Container`:
    - Constructor for `container`. Takes the `content` model to wrap and variadic `options`.
    - Initializes with default `borderStyle` (`lipgloss.NormalBorder()`).
    - Applies all provided `ContainerOption`s.
- **Padding Options**:
    - `WithPadding(top, right, bottom, left int) ContainerOption`: Sets padding for all sides.
    - `WithPaddingAll(padding int) ContainerOption`: Sets uniform padding.
    - `WithPaddingHorizontal(padding int) ContainerOption`: Sets left and right padding.
    - `WithPaddingVertical(padding int) ContainerOption`: Sets top and bottom padding.
- **Border Options**:
    - `WithBorder(top, right, bottom, left bool) ContainerOption`: Enables/disables borders for specified sides.
    - `WithBorderAll() ContainerOption`: Enables all borders.
    - `WithBorderHorizontal() ContainerOption`: Enables top and bottom borders.
    - `WithBorderVertical() ContainerOption`: Enables left and right borders.
- **Border Style Options**:
    - `WithBorderStyle(style lipgloss.Border) ContainerOption`: Sets a custom `lipgloss.Border`.
    - `WithRoundedBorder() ContainerOption`: Sets a rounded border style.
    - `WithThickBorder() ContainerOption`: Sets a thick border style.
    - `WithDoubleBorder() ContainerOption`: Sets a double border style.

## Important Variables/Constants
This file does not define exported package-level constants or variables beyond the interface, struct, and option types.

## Usage Examples

Wrapping a simple text model with padding and a border:
```go
// import (
//     tea "github.com/charmbracelet/bubbletea"
//     "github.com/opencode-ai/opencode/internal/tui/layout"
//     "github.com/charmbracelet/lipgloss" // For custom border example if needed
// )

// // Simple content model (example)
// type myContentModel struct { text string; width, height int }
// func (m myContentModel) Init() tea.Cmd { return nil }
// func (m myContentModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) { return m, nil }
// func (m myContentModel) View() string { return lipgloss.NewStyle().Width(m.width).Height(m.height).Render(m.text) }
// func (m *myContentModel) SetSize(w, h int) tea.Cmd { m.width = w; m.height = h; return nil }

// content := &myContentModel{text: "Hello, Container!"}

// // Create a container with padding and a rounded border
// container := layout.NewContainer(
//     content,
//     layout.WithPaddingAll(1),
//     layout.WithBorderAll(),
//     layout.WithRoundedBorder(),
// )

// // In the main application's Update or Init, set the container's size
// // container.SetSize(50, 10) // Example size

// // In the main application's View:
// // viewString := container.View()
// // This viewString would render the "Hello, Container!" text
// // surrounded by a rounded border and 1 character of padding on all sides.
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/tui/theme`: For `theme.CurrentTheme()` to get theme-specific colors (background, border).
    - Uses `Sizeable` and `Bindings` interfaces likely defined in `layout.go` within the same package.
- **External Libraries:**
    - `github.com/charmbracelet/bubbles/key`: For `key.Binding` type (used in `BindingKeys()`).
    - `github.com/charmbracelet/bubbletea`: For `tea.Model`, `tea.Cmd`.
    - `github.com/charmbracelet/lipgloss`: For all styling (padding, borders, colors).
- **Interactions:**
    - Acts as a decorator or wrapper for other `tea.Model` instances, adding layout properties.
    - Delegates core TUI model responsibilities (`Init`, `Update`, `View` for content, `BindingKeys`) to the wrapped content model.
    - Manages its own size and calculates the appropriate size for its content based on its padding and border settings.
    - Uses `lipgloss` extensively for rendering its borders and applying padding.
    - The visual appearance is themeable via `theme.CurrentTheme()`.

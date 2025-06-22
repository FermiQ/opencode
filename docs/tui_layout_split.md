# split.go (in internal/tui/layout)

## Overview

The `split.go` file, within the `internal/tui/layout` package, defines a `SplitPaneLayout` component. This layout manager allows for the creation of a TUI view that can be split into up to three panels: a left panel, a right panel (these two form a horizontal split), and an optional bottom panel that spans the full width beneath the top horizontal split. The relative sizes of these panels are controlled by ratios. The component implements `tea.Model` and the `Sizeable` and `Bindings` interfaces from `layout.go`.

## Key Components

### Interfaces
- `SplitPaneLayout`: The public interface for the split pane layout component. It embeds:
    - `tea.Model`: For standard TUI component behavior.
    - `Sizeable`: For managing its own size and distributing size to its child panels.
    - `Bindings`: For potentially aggregating key bindings from its child panels.
    - And adds methods for managing its panels:
        - `SetLeftPanel(panel Container) tea.Cmd`
        - `SetRightPanel(panel Container) tea.Cmd`
        - `SetBottomPanel(panel Container) tea.Cmd`
        - `ClearLeftPanel() tea.Cmd`
        - `ClearRightPanel() tea.Cmd`
        - `ClearBottomPanel() tea.Cmd`

### Structs
- `splitPaneLayout`: The concrete implementation of `SplitPaneLayout`.
    - `width (int)`, `height (int)`: Dimensions of the entire split layout.
    - `ratio (float64)`: The ratio determining the width of the left panel relative to the total width of the top section (left + right panels). E.g., 0.7 means left panel takes 70% of top width.
    - `verticalRatio (float64)`: The ratio determining the height of the top section (left + right panels) relative to the total height of the layout. E.g., 0.9 means the top section takes 90% of the total height, and the bottom panel takes the remaining 10%.
    - `rightPanel (Container)`: The container for the right panel's content.
    - `leftPanel (Container)`: The container for the left panel's content.
    - `bottomPanel (Container)`: The container for the bottom panel's content.

### Types
- `SplitPaneOption (func(*splitPaneLayout))`: Functional option type for configuring a `splitPaneLayout`.

### Methods on `splitPaneLayout`
- `Init() tea.Cmd`: Initializes all non-nil child panels (`leftPanel`, `rightPanel`, `bottomPanel`) and batches their initial commands.
- `Update(msg tea.Msg) (tea.Model, tea.Cmd)`:
    - Handles `tea.WindowSizeMsg` by calling `SetSize`.
    - Forwards other messages to all non-nil child panels and batches their commands.
- `View() string`:
    - Renders the views of its child panels.
    - If both `leftPanel` and `rightPanel` exist, their views are joined horizontally using `lipgloss.JoinHorizontal`.
    - If only one of them exists, its view is used for the top section.
    - If `bottomPanel` exists and there's a `topSection`, the `topSection` and `bottomPanel.View()` are joined vertically using `lipgloss.JoinVertical`.
    - If only `bottomPanel` exists, its view is used.
    - The final combined view is then rendered within a `lipgloss.Style` that applies the overall width, height, and background color from the current theme.
- `SetSize(width, height int) tea.Cmd`:
    - Updates its own `width` and `height`.
    - Calculates the dimensions for the top section and bottom panel based on `verticalRatio`.
    - Calculates the dimensions for the left and right panels within the top section based on `ratio`.
    - Calls `SetSize` on each non-nil child panel with its calculated dimensions and batches any returned commands.
- `GetSize() (int, int)`: Returns the layout's current width and height.
- `SetLeftPanel(panel Container) tea.Cmd`, `SetRightPanel(panel Container) tea.Cmd`, `SetBottomPanel(panel Container) tea.Cmd`: Set or replace a panel and then call `SetSize` to re-layout.
- `ClearLeftPanel() tea.Cmd`, `ClearRightPanel() tea.Cmd`, `ClearBottomPanel() tea.Cmd`: Set a panel to `nil` and then call `SetSize` to re-layout.
- `BindingKeys() []key.Binding`: Aggregates and returns key bindings from all child panels that implement the `Bindings` interface.

### Constructor and Options
- `NewSplitPane(options ...SplitPaneOption) SplitPaneLayout`:
    - Constructor for `splitPaneLayout`.
    - Initializes with default `ratio` (0.7 for left panel) and `verticalRatio` (0.9 for top section).
    - Applies all provided `SplitPaneOption`s.
- `WithLeftPanel(panel Container) SplitPaneOption`
- `WithRightPanel(panel Container) SplitPaneOption`
- `WithRatio(ratio float64) SplitPaneOption`: Sets the horizontal split ratio.
- `WithBottomPanel(panel Container) SplitPaneOption`
- `WithVerticalRatio(ratio float64) SplitPaneOption`: Sets the vertical split ratio between top and bottom sections.

## Important Variables/Constants
This file does not define exported package-level constants or variables beyond the interface, struct, and option types.

## Usage Examples

Creating a three-pane layout (e.g., a main content area, a sidebar, and a status bar at the bottom):
```go
// import "github.com/opencode-ai/opencode/internal/tui/layout"
// import tea "github.com/charmbracelet/bubbletea"

// Assume leftContent, rightContent, bottomContent are tea.Model implementations
// wrapped in layout.Container (e.g., layout.NewContainer(leftContentModel))

// leftPanel := layout.NewContainer(leftContent, layout.WithBorderAll())
// rightPanel := layout.NewContainer(rightContent, layout.WithBorderAll())
// bottomPanel := layout.NewContainer(bottomContent, layout.WithBorderHorizontal())

// threePaneLayout := layout.NewSplitPane(
//     layout.WithLeftPanel(leftPanel),
//     layout.WithRightPanel(rightPanel),
//     layout.WithBottomPanel(bottomPanel),
//     layout.WithRatio(0.3),         // Left panel takes 30% of top width
//     layout.WithVerticalRatio(0.8), // Top section (left+right) takes 80% of total height
// )

// In main TUI model:
// func (m MainModel) Init() tea.Cmd { return threePaneLayout.Init() }
// func (m MainModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
//     var cmd tea.Cmd
//     m.threePaneLayout, cmd = threePaneLayout.Update(msg)
//     return m, cmd
// }
// func (m MainModel) View() string { return threePaneLayout.View() }
// In response to tea.WindowSizeMsg:
// mainModel.threePaneLayout.SetSize(msg.Width, msg.Height)
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/tui/theme`: For `theme.CurrentTheme()` to get the background color.
    - Uses `Container`, `Sizeable`, and `Bindings` interfaces from the same `layout` package.
- **External Libraries:**
    - `github.com/charmbracelet/bubbles/key`: For `key.Binding`.
    - `github.com/charmbracelet/bubbletea`: For `tea.Model`, `tea.Cmd`, `tea.WindowSizeMsg`.
    - `github.com/charmbracelet/lipgloss`: For `lipgloss.JoinHorizontal`, `lipgloss.JoinVertical`, and styling the overall layout view.
- **Interactions:**
    - Acts as a higher-level layout component that composes other `Container` components.
    - Manages the distribution of available space to its child panels based on configured ratios.
    - Responds to `tea.WindowSizeMsg` to dynamically adjust the layout.
    - Forwards `Init` and `Update` messages to its children and aggregates their `tea.Cmd`s.
    - Aggregates key bindings from children that support the `Bindings` interface.
    - Provides a flexible way to create common TUI layouts like side-by-side views or views with a footer/header.

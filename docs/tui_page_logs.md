# logs.go (in internal/tui/page)

## Overview

The `logs.go` file, part of the `internal/tui/page` package, defines the "Logs" page for the OpenCode TUI. This page is responsible for displaying application logs to the user. It's structured with two main components: a table view for a summary of log entries (`logs.NewLogsTable()`) and a detail view (`logs.NewLogsDetails()`) likely showing the full content of a selected log entry. These components are arranged vertically.

## Key Components

### Variables
- `LogsPage (PageID)`: A constant of type `PageID` (likely defined in `page.go`) with the value "logs", used to identify this page.

### Interfaces
- `LogPage`: The public interface for the logs page. It embeds:
    - `tea.Model`: For standard TUI component behavior.
    - `layout.Sizeable`: For managing its size.
    - `layout.Bindings`: For exposing key bindings (primarily from the logs table).

### Structs
- `logsPage`: The concrete implementation of the `LogPage` interface.
    - `width (int)`, `height (int)`: Dimensions of the entire logs page.
    - `table (layout.Container)`: A container wrapping the `logs.LogsTable` component, which displays a list or table of log entries.
    - `details (layout.Container)`: A container wrapping the `logs.LogsDetails` component, which likely shows the detailed content of a log entry selected in the table.

### Methods on `logsPage`
- `Init() tea.Cmd`: Initializes its child components (`table` and `details`) and batches their initial commands.
- `Update(msg tea.Msg) (tea.Model, tea.Cmd)`: Handles incoming messages.
    - `tea.WindowSizeMsg`: Updates its own `width` and `height` and calls `SetSize` to adjust child component sizes.
    - Forwards other messages to both `table` and `details` components for them to update, and batches their commands.
- `View() string`:
    - Renders the page by vertically joining the views of the `table` and `details` components using `lipgloss.JoinVertical`.
    - The combined view is then rendered within a base style that applies the page's overall width and height.
- `BindingKeys() []key.Binding`: Implements `layout.Bindings`. It delegates this to the `table` component, implying that the primary interactions (like navigation) are handled by the logs table.
- `GetSize() (int, int)`: Implements `layout.Sizeable`. Returns the page's current width and height.
- `SetSize(width int, height int) tea.Cmd`: Implements `layout.Sizeable`.
    - Updates its own `width` and `height`.
    - Divides the available height equally between the `table` and `details` components (each gets `height/2`).
    - Calls `SetSize` on both child components with the full width and their allocated half-height.
- `NewLogsPage() LogPage`: Constructor for `logsPage`.
    - Initializes the `table` container with a new `logs.NewLogsTable()` component, wrapped with `layout.WithBorderAll()`.
    - Initializes the `details` container with a new `logs.NewLogsDetails()` component, also wrapped with `layout.WithBorderAll()`.

## Important Variables/Constants
- `LogsPage (PageID)`: Identifier for this page.

## Usage Examples

The `LogsPage` is another main page managed by the root TUI model. It's instantiated, and its `tea.Model` methods are called by the root model when the logs page is active.

```go
// In the main TUI model (e.g., internal/tui/tui.go)

// logsModel := page.NewLogsPage()

// // This logsModel would be part of a map of pages, and its Init, Update, View
// // methods would be called by the root TUI model based on the active page.
// // When the user navigates to the "logs" view, this page's View() method is called.
```
The user would typically interact with the `logs.LogsTable` component (e.g., scrolling, selecting a log entry), and the `logs.LogsDetails` component would update to show the details of the selected entry. This interaction logic is likely handled within the `logs.LogsTable` and `logs.LogsDetails` components themselves, possibly via messages passed between them or through shared state/events from the logging service.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/tui/components/logs`: For the actual log display components (`logs.NewLogsTable`, `logs.NewLogsDetails`).
    - `github.com/opencode-ai/opencode/internal/tui/layout`: For `layout.Container` to wrap child components and `layout.Sizeable`, `layout.Bindings` interfaces.
    - `github.com/opencode-ai/opencode/internal/tui/styles`: For `styles.BaseStyle()`.
- **External Libraries:**
    - `github.com/charmbracelet/bubbles/key`: For `key.Binding`.
    - `github.com/charmbracelet/bubbletea`: The core TUI framework.
    - `github.com/charmbracelet/lipgloss`: For `lipgloss.JoinVertical` and styling.
- **Interactions:**
    - Organizes the display of application logs into a two-part view (table and details).
    - Delegates most of the actual log handling and display logic to the child components from `internal/tui/components/logs`.
    - Manages the overall size and layout of these two components, splitting the available vertical space between them.
    - Forwards user input and other messages to its child components.
    - Key bindings are primarily sourced from the `table` component, suggesting it's the main focus for user interaction on this page.

# page.go (in internal/tui/page)

## Overview

The `page.go` file in the `internal/tui/page` package defines fundamental types used for page management and navigation within the Terminal User Interface (TUI). It introduces a `PageID` type to uniquely identify different pages or views within the application and a `PageChangeMsg` struct used as a Bubble Tea message to signal requests for page transitions.

## Key Components

### Types
- `PageID (string)`:
    - A type alias for `string`.
    - Used to create distinct identifiers for different pages in the TUI (e.g., `ChatPage`, `LogsPage` as defined in other files in this package).
    - This allows for type-safe references to pages when requesting navigation.

- `PageChangeMsg (struct)`:
    - A struct designed to be used as a `tea.Msg` (Bubble Tea message).
    - It's sent when a part of the application wants to trigger a change to a different page.
    - Fields:
        - `ID (PageID)`: Specifies the `PageID` of the page to navigate to.

## Important Variables/Constants

This file does not define exported package-level constants or variables itself, other than the type definitions. Specific `PageID` constants (like `ChatPage`, `LogsPage`) are defined in their respective page implementation files (e.g., `chat.go`, `logs.go`).

## Usage Examples

Requesting a page change from a component or event handler:
```go
// import tea "github.com/charmbracelet/bubbletea"
// import "github.com/opencode-ai/opencode/internal/tui/page"

// func (m MyComponentModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
//     // ... other logic ...
//     if someCondition {
//         // Request to switch to the Logs page
//         return m, func() tea.Msg {
//             return page.PageChangeMsg{ID: page.LogsPage} // Assuming LogsPage is defined elsewhere
//         }
//     }
//     // ...
//     return m, nil
// }
```

The main TUI model would then handle this `PageChangeMsg` in its `Update` function:
```go
// In the main TUI model's Update method:
// case page.PageChangeMsg:
//     m.currentPageID = msg.ID // Update the active page
//     // Potentially call Init() or Focus() on the new page component
//     return m, nil
```

## Dependencies and Interactions

- **Internal Dependencies:** None from other OpenCode packages within this specific file. However, the types defined here are fundamental for the page management system used by the main TUI application (`internal/tui/tui.go`) and all individual page components (`internal/tui/page/*`).
- **External Libraries:** None directly in this file (though `PageChangeMsg` is intended for use with `github.com/charmbracelet/bubbletea`).
- **Interactions:**
    - Provides the basic building blocks for identifying and navigating between different pages or views in the TUI.
    - `PageID` ensures that page references are somewhat type-checked and consistent.
    - `PageChangeMsg` facilitates a message-passing approach to page navigation, fitting the Elm Architecture / Bubble Tea model.
    - The root TUI model is responsible for listening for `PageChangeMsg` and updating its state to display the requested page.

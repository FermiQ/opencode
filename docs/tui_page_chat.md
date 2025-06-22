# chat.go (in internal/tui/page)

## Overview

The `chat.go` file, part of the `internal/tui/page` package, defines the main chat page for the OpenCode TUI. This page is a primary user interface component, structuring the chat interaction area. It typically consists of a message display area, a text input editor for user prompts, and an optional sidebar for session-related information or history. The page uses a `SplitPaneLayout` to arrange these components and handles user input, message sending, session management, and a completion dialog for "@" mentions.

## Key Components

### Variables
- `ChatPage (PageID)`: A constant of type `PageID` (likely defined in `page.go`) with the value "chat", used to identify this page.
- `keyMap (ChatKeyMap)`: A package-level variable defining the key bindings specific to the chat page.

### Structs
- `chatPage`: Implements `tea.Model` and the layout interfaces (`Sizeable`, `Bindings`). It represents the state and behavior of the chat page.
    - `app (*app.App)`: A reference to the main application object, providing access to services like session management and the AI agent.
    - `editor (layout.Container)`: The container for the chat input editor component.
    - `messages (layout.Container)`: The container for the message display area component.
    - `layout (layout.SplitPaneLayout)`: The main layout manager for the page, organizing the `messages`, `editor`, and potentially a sidebar.
    - `session (session.Session)`: The currently active chat session.
    - `completionDialog (dialog.CompletionDialog)`: A dialog component for "@" mention completions (e.g., for files).
    - `showCompletionDialog (bool)`: Flag to control the visibility of the completion dialog.
- `ChatKeyMap`: Defines the key bindings for the chat page.
    - `ShowCompletionDialog (key.Binding)`: Key to trigger the completion dialog (e.g., "@").
    - `NewSession (key.Binding)`: Key to start a new chat session (e.g., "ctrl+n").
    - `Cancel (key.Binding)`: Key to cancel an ongoing AI agent operation (e.g., "esc").

### Core Functions
- `(p *chatPage) Init() tea.Cmd`: Initializes the main layout and the completion dialog.
- `(p *chatPage) Update(msg tea.Msg) (tea.Model, tea.Cmd)`: Handles incoming messages and updates the page state.
    - `tea.WindowSizeMsg`: Updates the layout size.
    - `dialog.CompletionDialogCloseMsg`: Hides the completion dialog.
    - `chat.SendMsg`: Triggered by the editor component when the user sends a message. Calls `p.sendMessage()`.
    - `dialog.CommandRunCustomMsg`: Triggered by the completion dialog when a custom command (e.g., from an "@" mention) is selected. It formats the command and calls `p.sendMessage()`. Checks if the agent is busy before sending.
    - `chat.SessionSelectedMsg`: Updates the current `p.session` and calls `p.setSidebar()` if it's a new session.
    - `tea.KeyMsg`:
        - "@": Shows the completion dialog.
        - "ctrl+n": Clears the current session and sidebar, signaling a new session.
        - "esc": If a session is active, cancels any ongoing agent processing for that session using `p.app.CoderAgent.Cancel()`.
    - If `p.showCompletionDialog` is true, updates the completion dialog and potentially consumes "enter" key presses.
    - Forwards other messages to the main `p.layout` for child components to handle.
- `(p *chatPage) setSidebar() tea.Cmd`: Creates and sets a new `chat.SidebarCmp` as the right panel in the layout.
- `(p *chatPage) clearSidebar() tea.Cmd`: Clears the right panel from the layout.
- `(p *chatPage) sendMessage(text string, attachments []message.Attachment) tea.Cmd`:
    - If no session is active (`p.session.ID == ""`), it creates a new session using `p.app.Sessions.Create()`, sets the sidebar, and sends a `chat.SessionSelectedMsg`.
    - Calls `p.app.CoderAgent.Run()` to send the user's `text` and `attachments` to the AI agent for the current session.
- `(p *chatPage) SetSize(width, height int) tea.Cmd`: Implements `layout.Sizeable`, passing the size to the main `p.layout`.
- `(p *chatPage) GetSize() (int, int)`: Implements `layout.Sizeable`.
- `(p *chatPage) View() string`:
    - Renders the main `p.layout`.
    - If `p.showCompletionDialog` is true, it renders the `completionDialog` and uses `layout.PlaceOverlay` to display it on top of the main layout, positioned relative to the editor.
- `(p *chatPage) BindingKeys() []key.Binding`: Implements `layout.Bindings`. Aggregates key bindings from `keyMap` and its child components (`messages`, `editor`).
- `NewChatPage(app *app.App) tea.Model`: Constructor for `chatPage`.
    - Initializes a `completions.FileAndFolderContextGroup` for the completion dialog.
    - Creates `layout.Container`s for the messages area (`chat.NewMessagesCmp`) and the editor (`chat.NewEditorCmp`).
    - Sets up the main `layout.SplitPaneLayout` with the messages panel as the left/main panel and the editor as the bottom panel.

## Important Variables/Constants
- `ChatPage (PageID)`: Identifier for this page.
- `keyMap (ChatKeyMap)`: Defines the primary keyboard interactions for the chat page.

## Usage Examples

The `ChatPage` is typically one of the main pages managed by the root TUI model. It's instantiated and its `tea.Model` methods are called as part of the main Bubble Tea application loop.

```go
// In the main TUI model (e.g., internal/tui/tui.go)

// var app *app.App // Initialized application instance
// chatModel := page.NewChatPage(app)

// // This chatModel would be part of a map of pages, and its Init, Update, View
// // methods would be called by the root TUI model based on the active page.
```

When the user types "@" in the chat input, the `completionDialog` becomes active. If the user selects a completion that results in a `dialog.CommandRunCustomMsg`, the chat page processes it to send a formatted command to the agent.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/app`: For `app.App` to access services.
    - `github.com/opencode-ai/opencode/internal/completions`: For `completions.NewFileAndFolderContextGroup` used by the completion dialog.
    - `github.com/opencode-ai/opencode/internal/message`: For `message.Attachment`.
    - `github.com/opencode-ai/opencode/internal/session`: For `session.Session`.
    - `github.com/opencode-ai/opencode/internal/tui/components/chat`: For child components like messages display, editor, and sidebar, and their associated messages (`chat.SendMsg`, `chat.SessionSelectedMsg`).
    - `github.com/opencode-ai/opencode/internal/tui/components/dialog`: For `CompletionDialog` and its messages.
    - `github.com/opencode-ai/opencode/internal/tui/layout`: For `SplitPaneLayout`, `Container`, and layout interfaces.
    - `github.com/opencode-ai/opencode/internal/tui/util`: For utility messages like `util.ReportError`, `util.ReportWarn`, `util.CmdHandler`.
- **External Libraries:**
    - `github.com/charmbracelet/bubbles/key`: For key binding definitions.
    - `github.com/charmbracelet/bubbletea`: The core TUI framework.
    - `github.com/charmbracelet/lipgloss`: Used by `layout.PlaceOverlay` for rendering.
- **Interactions:**
    - Orchestrates the main chat interface, combining message display, input editor, and an optional sidebar.
    - Handles sending user messages (with potential attachments) to the `app.CoderAgent`.
    - Manages session creation and selection.
    - Implements a completion dialog for "@" mentions.
    - Allows cancellation of ongoing agent requests.
    - Delegates rendering and sizing to its layout components.

# chat.go (in internal/tui/components/chat)

## Overview

The `chat.go` file, located in the `internal/tui/components/chat` package, defines several message types used for communication within the chat components of the TUI. It also includes helper functions to render a static header or welcome screen. This screen typically displays the OpenCode logo, version, repository URL, current working directory, and a list of configured LSP servers. This header is likely shown when a new chat session is initiated or before any messages are present.

## Key Components

### Message Types
- `SendMsg`: A struct used as a `tea.Msg` to signal that the user has sent a new message.
    - `Text (string)`: The textual content of the message.
    - `Attachments ([]message.Attachment)`: A slice of attachments (e.g., images) included with the message.
- `SessionSelectedMsg (session.Session)`: A type alias for `session.Session`. Used as a `tea.Msg` to indicate that a specific session has been selected or has become active.
- `SessionClearedMsg (struct{})`: An empty struct used as a `tea.Msg` to signal that the current session has been cleared or a new, empty session is being started.
- `EditorFocusMsg (bool)`: A `tea.Msg` likely used to signal whether the chat input editor should gain or lose focus. The boolean value probably indicates `true` for focus and `false` for blur.

### Helper Functions (for rendering the header/welcome view)
- `header(width int) string`:
    - Composes the main header view by vertically joining the outputs of `logo()`, `repo()`, an empty line, and `cwd()`.
- `lspsConfigured(width int) string`:
    - Renders a section displaying the list of configured Language Server Protocols (LSPs).
    - It retrieves LSP configurations from `config.Get()`, sorts them by name, and formats each with its name and command path.
    - Uses `lipgloss` and theme colors for styling.
- `logo(width int) string`:
    - Renders the OpenCode logo (using `styles.OpenCodeIcon`), name, and current application `version.Version`.
    - Styled using `lipgloss` and theme colors.
- `repo(width int) string`:
    - Renders the application's GitHub repository URL.
- `cwd(width int) string`:
    - Renders the current working directory obtained from `config.WorkingDirectory()`.

## Usage Examples

The message types (`SendMsg`, `SessionSelectedMsg`, etc.) are created and sent by various chat components (like the editor or session list) to the main chat page (`internal/tui/page/chat.go`) or other relevant models to trigger state changes or actions.

The `header()` function (and its sub-functions) are likely called by a chat view component (e.g., `internal/tui/components/chat/list.go` or `messages.go` if that's its new name) when it needs to display an initial welcome screen before a session starts or when a session is cleared.

```go
// Conceptual: If a chat view component needs to display a welcome screen:
// if noActiveSession {
//     welcomeView := chat.header(availableWidth)
//     // ... render welcomeView ...
// }
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For `config.Get()` to access LSP and working directory info.
    - `github.com/opencode-ai/opencode/internal/message`: For `message.Attachment` type.
    - `github.com/opencode-ai/opencode/internal/session`: For `session.Session` type.
    - `github.com/opencode-ai/opencode/internal/tui/styles`: For `styles.OpenCodeIcon` and `styles.BaseStyle()`.
    - `github.com/opencode-ai/opencode/internal/tui/theme`: For `theme.CurrentTheme()` to get theme colors.
    - `github.com/opencode-ai/opencode/internal/version`: For `version.Version`.
- **External Libraries:**
    - `github.com/charmbracelet/lipgloss`: For all styling of the header components.
    - `github.com/charmbracelet/x/ansi`: For string truncation (`ansi.Truncate`).
- **Interactions:**
    - This file provides message structs that facilitate communication between different parts of the chat UI (e.g., editor sending a message, session list selecting a session).
    - The header rendering functions create a static, styled view that is likely used as a placeholder or welcome screen in the chat interface when no conversation is active.

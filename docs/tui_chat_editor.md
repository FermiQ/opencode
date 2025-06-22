# editor.go (in internal/tui/components/chat)

## Overview

The `editor.go` file defines the chat input editor component (`editorCmp`) for the TUI. This component provides a multi-line text input area for users to compose their messages. It supports sending messages, opening an external editor (`$EDITOR`) for longer messages, managing file attachments, and a delete mode for attachments. It integrates with the `app.App` for sending messages to the AI agent and theme changes.

## Key Components

- **`editorCmp` struct**: The main `tea.Model` for the editor.
    - Manages its `width`, `height`, a reference to `app.App`, the current `session.Session`, a `textarea.Model` for text input, a slice of `message.Attachment`s, and a `deleteMode` boolean for attachment management.
- **KeyMaps**:
    - `EditorKeyMaps`: Defines "send" (`Enter`, `Ctrl+S`) and "open editor" (`Ctrl+E`) bindings.
    - `DeleteAttachmentKeyMaps`: Defines bindings for entering attachment delete mode (`Ctrl+R`), deleting all attachments (`Ctrl+R` then `R`), and deleting a specific attachment by index (`Ctrl+R` then a digit).
- **Core Functionality**:
    - `Init()`: Initializes the underlying textarea.
    - `Update(msg tea.Msg)`: Handles various messages:
        - `dialog.ThemeChangedMsg`: Re-styles the textarea.
        - `dialog.CompletionSelectedMsg`: Inserts selected completion text.
        - `SessionSelectedMsg`: Updates the current session.
        - `dialog.AttachmentAddedMsg`: Adds an attachment (up to `maxAttachments`).
        - Key presses for sending, opening external editor, and managing attachments.
        - Forwards other messages to the `textarea.Model`.
    - `View() string`: Renders the attachments display (if any) and the textarea, prefixed by a ">" prompt.
    - `SetSize(width, height int) tea.Cmd`: Adjusts the size of the textarea.
    - `BindingKeys() []key.Binding`: Returns its key bindings.
- **Helper Functions**:
    - `openEditor()`: Opens the system's `$EDITOR` for message composition.
    - `send()`: Packages the textarea content and attachments into a `SendMsg` if the agent is not busy.
    - `attachmentsContent()`: Renders the list of currently added attachments.
    - `CreateTextArea()`: Factory function to create and style a `textarea.Model`.
- `NewEditorCmp(app *app.App) tea.Model`: Constructor for `editorCmp`.

## Dependencies and Interactions

- Interacts with `app.App` to check agent busy status (`app.CoderAgent.IsSessionBusy`) and potentially for other application-level actions.
- Uses `session.Session` to be aware of the current chat context.
- Manages `message.Attachment`s.
- Responds to `dialog.ThemeChangedMsg` for theming and `dialog.CompletionSelectedMsg` for inserting completions.
- Communicates with parent components (likely `page.ChatPage`) by sending `SendMsg`.
- Relies on `layout.Container` (implicitly, as it's wrapped by one in `page.ChatPage`) and `layout.KeyMapToSlice`.
- Uses `textarea.Model` from `charmbracelet/bubbles` for text input.
- Uses `lipgloss` for styling.

## Purpose

This component is the primary means for users to input text and attachments for their chat messages to the AI agent. It provides a rich editing experience including external editor support and basic attachment management.

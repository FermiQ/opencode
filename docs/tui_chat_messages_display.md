# list.go (in internal/tui/components/chat)
## Renamed from `list.go` to `messages_display.md` for clarity, as it's about displaying messages.

## Overview

The `list.go` file (internally referred to as `messagesCmp`) defines the TUI component responsible for rendering the list of messages within a chat session. It uses a `viewport.Model` from `charmbracelet/bubbles` to handle scrolling through potentially long conversations. The component subscribes to message and session events to update its display dynamically. It also includes logic for an initial welcome screen, displaying a "thinking" spinner, and rendering individual messages with appropriate styling based on their role and content.

## Key Components

- **`messagesCmp` struct**: The main `tea.Model` for displaying messages.
    - Manages `width`, `height`, an `app.App` reference, the current `session.Session`, a slice of `message.Message`s, cached UI representations (`uiMessage`), a `viewport.Model` for scrolling, a `spinner.Model` for loading states, and an `attachments` viewport (though its usage isn't fully clear from this file alone).
- **`MessageKeys` struct**: Defines key bindings for navigating the message list (PageUp, PageDown, HalfPageUp, HalfPageDown).
- **Core Functionality**:
    - `Init()`: Initializes the viewport and spinner.
    - `Update(msg tea.Msg)`: Handles various messages:
        - `dialog.ThemeChangedMsg`: Triggers a re-render of messages.
        - `SessionSelectedMsg`, `SessionClearedMsg`: Updates the current session and message list, then re-renders.
        - `tea.KeyMsg`: Handles viewport scrolling keys.
        - `renderFinishedMsg`: Sets `rendering` flag to false and scrolls to bottom.
        - `pubsub.Event[session.Session]`, `pubsub.Event[message.Message]`: Updates session or message data if relevant to the current session and triggers a re-render.
    - `View() string`:
        - If no messages, renders an `initialScreen()` (logo, version, LSP info, etc.).
        - If messages exist, renders the `viewport` content.
        - Displays a `working()` indicator (spinner and status text like "Thinking...", "Generating...") if the agent is busy.
        - Renders a `help()` line with context-dependent key hints.
    - `renderView()`: Converts `message.Message`s into `uiMessage`s (which include pre-rendered content and height, likely defined in `message.go` of this package) and sets the viewport content. Uses caching (`cachedContent`) to avoid re-rendering unchanged messages.
    - `SetSize(width, height int) tea.Cmd`: Adjusts component and viewport sizes and triggers a re-render.
    - `SetSession(session session.Session) tea.Cmd`: Loads messages for the given session and triggers a re-render.
- **Helper Functions**:
    - `initialScreen()`, `header()`, `logo()`, `repo()`, `cwd()`, `lspsConfigured()`: Functions to render the initial welcome/info screen (some of these are defined in `chat.go` of this package but used here).
    - `working()`: Renders the spinner and "Agent is working..." type messages.
    - `help()`: Renders contextual help text.
    - `formatTimeDifference()`: Utility to format time differences.
    - `hasToolsWithoutResponse()`, `hasUnfinishedToolCalls()`: Check the state of tool calls in messages.
- `NewMessagesCmp(app *app.App) tea.Model`: Constructor for `messagesCmp`.

## Dependencies and Interactions

- Relies on `app.App` for accessing services (Messages, CoderAgent, Session).
- Subscribes to `session.Session` and `message.Message` events via `pubsub` to update dynamically.
- Uses `chat.renderUserMessage` and `chat.renderAssistantMessage` (likely from `message.go` in this package) to format individual messages.
- Uses `viewport.Model` and `spinner.Model` from `charmbracelet/bubbles`.
- Uses `lipgloss` for styling.

## Purpose

This component is the main area where users see the conversation history. It's responsible for efficiently rendering potentially many messages, handling scrolling, and providing visual feedback on the agent's status.

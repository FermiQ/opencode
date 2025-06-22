# status.go (in internal/tui/components/core)

## Overview

The `status.go` file defines the `statusCmp` TUI component, which acts as the application's status bar. It displays various pieces of information such as:
- A help widget (`ctrl+? help`).
- Context usage and cost for the current session.
- Project-wide diagnostics summary (errors, warnings, hints, info) from LSP clients, or an "Initializing LSP..." message.
- The name of the currently configured AI model for the Coder agent.
- Temporary informational, warning, or error messages sent from other parts of the application.

## Key Components

- **`StatusCmp` interface**: Public interface for the status component.
- **`statusCmp` struct**: The main `tea.Model` for the status bar.
    - Manages `width`, a temporary `info` message (`util.InfoMsg`), `messageTTL` for how long info messages are displayed, a map of `lspClients`, and the current `session.Session`.
- **Core Functionality**:
    - `Init()`: Returns `nil` (no initial command).
    - `Update(msg tea.Msg)`: Handles messages:
        - `tea.WindowSizeMsg`: Updates the component's width.
        - `chat.SessionSelectedMsg`, `chat.SessionClearedMsg`, `pubsub.Event[session.Session]`: Updates the current session information used for displaying token/cost.
        - `util.InfoMsg`: Displays a temporary message (info, warning, error) and sets a timer (`clearMessageCmd`) to clear it after `messageTTL` or the message's specified TTL.
        - `util.ClearStatusMsg`: Clears the temporary info message.
    - `View() string`: Renders the status bar by horizontally joining several pieces of information:
        - Help widget (`getHelpWidget()`).
        - Session token usage and cost (`formatTokensAndCost()`), if a session is active.
        - Temporary info message, if any, styled by its type (info, warn, error) and truncated to fit.
        - Project diagnostics summary (`projectDiagnostics()`).
        - Current AI model name (`model()`).
        The available width for the temporary info message is calculated to ensure the entire status bar fits within the screen width.
- **Helper Functions**:
    - `clearMessageCmd()`: Returns a `tea.Cmd` to send a `util.ClearStatusMsg` after a delay.
    - `getHelpWidget()`: Renders the "ctrl+? help" text with theming.
    - `formatTokensAndCost()`: Formats token counts (e.g., to K or M) and cost for display, including a warning icon if token usage is high.
    - `projectDiagnostics()`: Aggregates diagnostics from all LSP clients, showing counts of errors, warnings, hints, and info, or an "Initializing LSP..." message.
    - `model()`: Displays the name of the current Coder agent's model.
- `NewStatusCmp(lspClients map[string]*lsp.Client) StatusCmp`: Constructor for `statusCmp`.

## Dependencies and Interactions

- Relies on `config.Get()` for Coder agent model information.
- Uses `models.SupportedModels` to get model names.
- Interacts with `lsp.Client` instances to get diagnostics and server state.
- Responds to `session.Session` updates via `pubsub` and specific messages (`chat.SessionSelectedMsg`, `chat.SessionClearedMsg`).
- Displays messages of type `util.InfoMsg` and clears them using `util.ClearStatusMsg`.
- Uses `styles` and `theme` for all visual styling via `lipgloss`.

## Purpose

This component provides a persistent status bar at the bottom of the TUI, offering users at-a-glance information about the application's state, ongoing session costs, code health (diagnostics), and temporary feedback messages.

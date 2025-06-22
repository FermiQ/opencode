# session.go (in internal/tui/components/dialog)

## Overview

The `session.go` file defines the `SessionDialogCmp` TUI component. This component presents a dialog to the user, listing all available chat sessions and allowing them to select one to switch to. It's a crucial part of managing multiple conversations within the application.

## Key Components

- **Message Types**:
    - `SessionSelectedMsg`: Sent when a user selects a session from the dialog. Contains the chosen `session.Session`.
    - `CloseSessionDialogMsg`: Sent when the dialog is closed without making a selection (e.g., by pressing Escape).
- **`SessionDialog` interface**: Public interface for the session selection dialog component.
    - `SetSessions(sessions []session.Session)`: Method to populate the dialog with a list of sessions.
    - `SetSelectedSession(sessionID string)`: Method to pre-select a session by its ID.
- **`sessionDialogCmp` struct**: The main `tea.Model` for the session dialog.
    - Manages `sessions ([]session.Session)`, `selectedIdx` (for the list), `width`, `height`, and `selectedSessionID` (to track the initially selected/active session).
- **`sessionKeyMap` struct**: Defines key bindings for navigation (Up/Down/J/K), selection (Enter), and closing (Escape).
- **Core Functionality**:
    - `Init()`: Returns `nil`.
    - `Update(msg tea.Msg)`: Handles messages:
        - `tea.KeyMsg`: Processes navigation keys to change `selectedIdx` and selection/close keys to send `SessionSelectedMsg` or `CloseSessionDialogMsg`.
        - `tea.WindowSizeMsg`: Updates component dimensions.
    - `View() string`: Renders the dialog:
        - Displays a title "Switch Session".
        - Lists the titles of available sessions (up to `maxVisibleSessions`, currently 10).
        - Highlights the `selectedIdx` session.
        - Implements basic scrolling logic if the number of sessions exceeds `maxVisibleSessions`.
        - The dialog is styled with a rounded border.
    - `BindingKeys() []key.Binding`: Returns the dialog's key bindings.
    - `SetSessions(sessions []session.Session)`: Populates the dialog with sessions and updates `selectedIdx` if `selectedSessionID` matches one of them.
    - `SetSelectedSession(sessionID string)`: Sets the `selectedSessionID` and updates `selectedIdx` if sessions are already loaded.
- `NewSessionDialogCmp() SessionDialog`: Constructor for `sessionDialogCmp`.

## Dependencies and Interactions

- Uses `session.Session` from `internal/session`.
- Uses `key.Binding` from `charmbracelet/bubbles`.
- Uses `lipgloss` for styling.
- Relies on `theme.CurrentTheme()` and `styles.BaseStyle()` for visual appearance.
- Communicates the user's selection (or cancellation) to a parent model via `SessionSelectedMsg` or `CloseSessionDialogMsg`. The parent model (likely the main TUI or a dialog manager) is then responsible for switching the active chat session.
- The list of sessions is provided externally via the `SetSessions` method, typically by a component that has access to the `session.Service`.

## Purpose

This component provides a user interface for managing and switching between multiple chat sessions. It allows users to easily navigate their conversation history and resume previous interactions.

# arguments.go (in internal/tui/components/dialog)

## Overview

The `arguments.go` file defines a TUI component `MultiArgumentsDialogCmp`. This component presents a dialog to the user, allowing them to input values for multiple named arguments required by a command (often a custom command selected via a completion dialog). It features multiple text input fields, one for each argument, and supports navigation between them.

## Key Components

- **`argumentsDialogKeyMap` struct**: Defines key bindings for the dialog (Enter to confirm/move to next, Escape to cancel).
- **Message Types**:
    - `ShowMultiArgumentsDialogMsg`: A `tea.Msg` to trigger the display of this dialog. Contains `CommandID`, `Content` (the command template), and `ArgNames` (list of argument names to prompt for).
    - `CloseMultiArgumentsDialogMsg`: A `tea.Msg` sent when the dialog is closed. Contains a `Submit` flag (true if confirmed, false if cancelled), and the original `CommandID`, `Content`, along with a map `Args` of argument names to their entered values.
- **`MultiArgumentsDialogCmp` struct**: The main `tea.Model` for the dialog.
    - Manages `width`, `height`, a slice of `textinput.Model`s (one for each argument), `focusIndex` to track the active input, `keys`, and the `commandID`, `content`, and `argNames` passed in via `ShowMultiArgumentsDialogMsg`.
- **Core Functionality**:
    - `Init()`: Initializes the text inputs, focusing the first one.
    - `Update(msg tea.Msg)`: Handles messages:
        - `tea.KeyMsg`:
            - `esc`: Closes the dialog, sending `CloseMultiArgumentsDialogMsg` with `Submit: false`.
            - `enter`: If on the last input, submits the dialog (sends `CloseMultiArgumentsDialogMsg` with `Submit: true` and collected args). Otherwise, moves focus to the next input.
            - `tab`/`shift+tab`: Navigates focus between input fields.
        - Updates the currently focused text input.
    - `View() string`: Renders the dialog with a title, an explanation, and a vertical list of labels (argument names) and their corresponding text input fields. Highlights the focused input's label.
    - `SetSize(width, height int)`: Updates the component's dimensions.
    - `Bindings() []key.Binding`: Returns the dialog's key bindings.
- `NewMultiArgumentsDialogCmp(commandID, content string, argNames []string) MultiArgumentsDialogCmp`: Constructor that initializes the dialog with the necessary command information and creates styled text input fields for each argument name provided.

## Dependencies and Interactions

- Uses `textinput.Model` from `charmbracelet/bubbles` for each argument field.
- Uses `key.Binding` from `charmbracelet/bubbles` for key maps.
- Uses `lipgloss` for styling the dialog, title, explanation, labels, and input fields.
- Relies on `theme.CurrentTheme()` for colors and `styles.BaseStyle()` for base styling.
- Communicates its result (or cancellation) back to the calling model (likely `page.ChatPage` or another dialog manager) by sending a `CloseMultiArgumentsDialogMsg`.
- Triggered by a `ShowMultiArgumentsDialogMsg`.

## Purpose

This component provides a structured way to gather multiple pieces of input from the user when a command requires specific arguments. It's typically used in conjunction with custom commands defined elsewhere (e.g., from `@` mentions processed by `CompletionDialogCmp`) that have placeholders for user-supplied values.

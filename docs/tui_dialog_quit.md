# quit.go (in internal/tui/components/dialog)

## Overview

The `quit.go` file defines the `QuitDialogCmp` TUI component. This component presents a simple confirmation dialog to the user asking "Are you sure you want to quit?". It provides "Yes" and "No" options for the user to confirm or cancel the quit action.

## Key Components

- **`CloseQuitMsg` struct**: An empty struct used as a `tea.Msg` to signal that the quit dialog should be closed (if the user selects "No").
- **`QuitDialog` interface**: Public interface for the quit dialog component.
- **`quitDialogCmp` struct**: The main `tea.Model` for the quit dialog.
    - Manages `selectedNo (bool)`: Tracks if the "No" option is currently selected (true if "No" is selected, false if "Yes" is selected). Defaults to `true` (so "No" is selected initially).
- **`helpMapping` struct**: Defines key bindings for the dialog:
    - `Left`/`Right`/`Tab`: Switch selection between "Yes" and "No".
    - `Enter`/`Space`: Confirm the currently selected option.
    - `Y`/`y`: Select "Yes" (quit).
    - `N`/`n`: Select "No" (close dialog).
- **Core Functionality**:
    - `Init()`: Returns `nil`.
    - `Update(msg tea.Msg)`: Handles messages:
        - `tea.KeyMsg`: Processes key presses according to `helpKeys`.
            - Navigation keys toggle `selectedNo`.
            - Confirmation keys: If "Yes" is selected (`!selectedNo`), sends `tea.Quit`. If "No" is selected, sends `CloseQuitMsg`.
            - 'Y'/'y' keys directly send `tea.Quit`.
            - 'N'/'n' keys directly send `CloseQuitMsg`.
    - `View() string`: Renders the dialog:
        - Displays the question "Are you sure you want to quit?".
        - Renders "Yes" and "No" buttons, with the currently selected button highlighted using theme colors.
        - The dialog is styled with a rounded border.
    - `BindingKeys() []key.Binding`: Returns the dialog's key bindings.
- `NewQuitCmp() QuitDialog`: Constructor for `quitDialogCmp`, initializing `selectedNo` to `true`.

## Dependencies and Interactions

- Uses `key.Binding` from `charmbracelet/bubbles` for key maps.
- Uses `lipgloss` for styling the dialog and buttons.
- Relies on `theme.CurrentTheme()` and `styles.BaseStyle()` for visual appearance.
- If the user confirms "Yes", it sends a `tea.Quit` message, which is handled by the main Bubble Tea application to terminate the TUI.
- If the user selects "No", it sends a `CloseQuitMsg`, which a parent component (likely a dialog manager or the main TUI model) handles to hide the quit dialog.

## Purpose

This component provides a standard confirmation step before the application exits, preventing accidental quits. It's a common UI pattern for ensuring user intent.

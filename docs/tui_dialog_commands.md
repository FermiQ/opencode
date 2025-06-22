# commands.go (in internal/tui/components/dialog)

## Overview

The `commands.go` file defines a reusable TUI component, `CommandDialogCmp`, for presenting a list of executable commands to the user. Each command has a title, an optional description, and an associated handler function. The dialog allows users to navigate the list and select a command for execution. It's built using a generic `SimpleList` component for the list rendering and interaction.

## Key Components

- **`Command` struct**: Represents an individual command that can be displayed and executed.
    - `ID (string)`: A unique identifier for the command.
    - `Title (string)`: The display name of the command.
    - `Description (string)`: An optional longer description.
    - `Handler (func(cmd Command) tea.Cmd)`: A function to be called when the command is selected.
    - `Render(selected bool, width int) string`: Method to render the command item with appropriate styling (highlighting if selected).
- **Message Types**:
    - `CommandSelectedMsg`: Sent when a user selects a command from the dialog. Contains the selected `Command`.
    - `CloseCommandDialogMsg`: Sent when the dialog is closed (e.g., by pressing Escape).
- **`CommandDialog` interface**: Defines the public interface for the command dialog component.
- **`commandDialogCmp` struct**: The main `tea.Model` for the command dialog.
    - Manages `listView (utilComponents.SimpleList[Command])` to display and handle the list of commands.
    - Stores `width` and `height`.
- **`commandKeyMap` struct**: Defines key bindings for the dialog (Enter to select, Escape to close).
- **Core Functionality**:
    - `Init()`: Initializes the underlying list view.
    - `Update(msg tea.Msg)`: Handles messages:
        - `tea.KeyMsg`:
            - `enter`: Sends a `CommandSelectedMsg` with the currently selected command.
            - `esc`: Sends a `CloseCommandDialogMsg`.
        - Forwards other messages (including navigation keys like up/down) to the `listView` component.
    - `View() string`: Renders the dialog, which includes a title ("Commands") and the view of the `listView` component, all within a styled bordered box.
    - `BindingKeys() []key.Binding`: Returns the dialog's key bindings.
    - `SetCommands(commands []Command)`: Allows updating the list of commands displayed in the dialog.
- `NewCommandDialogCmp() CommandDialog`: Constructor for `commandDialogCmp`. Initializes the `SimpleList` with default settings.

## Dependencies and Interactions

- Uses `utilComponents.SimpleList[Command]` (from `internal/tui/components/util`) for the core list management and rendering.
- Uses `key.Binding` from `charmbracelet/bubbles` for key maps.
- Uses `lipgloss` for styling the dialog box, title, and command items.
- Relies on `theme.CurrentTheme()` for colors and `styles.BaseStyle()` for base styling.
- Communicates its actions (command selection, dialog close) to a parent model by sending `CommandSelectedMsg` or `CloseCommandDialogMsg`. The parent model is then responsible for invoking the `Handler` of the selected command.

## Purpose

This component provides a generic and reusable UI element for presenting a list of actions or commands to the user. It abstracts the list navigation and selection logic, allowing different parts of the TUI to offer command palettes or action menus with custom sets of commands and handlers.

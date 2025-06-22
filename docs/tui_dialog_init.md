# init.go (in internal/tui/components/dialog)

## Overview

The `init.go` file defines the `InitDialogCmp` TUI component. This component presents a dialog to the user asking if they want to initialize the current project. Initialization, in this context, means generating a new `OpenCode.md` file. This file is intended to store project-specific information, commands, and style preferences to serve as a persistent memory or context for the AI agents working on that project.

## Key Components

- **`initDialogKeyMap` struct**: Defines key bindings for the dialog:
    - `Tab`/`Left`/`Right`/`h`/`l`: Toggle selection between "Yes" and "No" buttons.
    - `Enter`: Confirm the selected option.
    - `Escape`/`q`: Cancel and close the dialog (defaults to "No").
    - `y`/`n`: Directly select "Yes" or "No".
- **Message Types**:
    - `CloseInitDialogMsg`: Sent when the dialog is closed.
        - `Initialize (bool)`: True if the user chose "Yes", false otherwise.
    - `ShowInitDialogMsg`: (Defined but not directly used as a trigger within this component's `Update` logic; likely sent by a parent component to show this dialog).
        - `Show (bool)`: Controls visibility.
- **`InitDialogCmp` struct**: The main `tea.Model` for the initialization dialog.
    - Manages `width`, `height`, `selected` (0 for "Yes", 1 for "No"), and `keys`.
- **Core Functionality**:
    - `Init()`: Returns `nil`.
    - `Update(msg tea.Msg)`: Handles messages:
        - `tea.KeyMsg`: Processes key presses according to `initDialogKeyMap` to change selection or send a `CloseInitDialogMsg`.
        - `tea.WindowSizeMsg`: Updates component dimensions.
    - `View() string`: Renders the dialog with:
        - A title "Initialize Project".
        - An explanation of what project initialization and `OpenCode.md` entail.
        - The question "Would you like to initialize this project?".
        - "Yes" and "No" buttons, with the currently `selected` one highlighted.
        - All styled using `lipgloss` and theme colors.
    - `SetSize(width, height int)`: Updates component dimensions.
    - `Bindings() []key.Binding`: Returns the dialog's key bindings.
- `NewInitDialogCmp() InitDialogCmp`: Constructor for `InitDialogCmp`.

## Dependencies and Interactions

- Uses `key.Binding` from `charmbracelet/bubbles` for key maps.
- Uses `lipgloss` for styling the dialog.
- Relies on `theme.CurrentTheme()` and `styles.BaseStyle()` for visual appearance.
- Communicates the user's decision (to initialize or not) back to a parent model by sending a `CloseInitDialogMsg`. The parent model is then responsible for initiating the actual project initialization (e.g., calling `config.MarkProjectInitialized()` and creating the `OpenCode.md` file).
- The display of this dialog is likely triggered by a parent component checking `config.ShouldShowInitDialog()`.

## Purpose

This component provides a user-friendly way to prompt the user for a one-time project initialization step. This step aims to enhance the AI's contextual understanding and memory for a specific project by creating a dedicated `OpenCode.md` file.

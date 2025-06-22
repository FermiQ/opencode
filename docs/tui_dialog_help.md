# help.go (in internal/tui/components/dialog)

## Overview

The `help.go` file defines the `helpCmp` TUI component, which is responsible for rendering a help dialog. This dialog displays a list of available keyboard shortcuts (key bindings) to the user. It dynamically formats these bindings into columns for readability and uses the current theme for styling.

## Key Components

- **`helpCmp` struct**: The main `tea.Model` for the help dialog.
    - Manages `width`, `height`, and a slice `keys` of `key.Binding` to display.
- **Core Functionality**:
    - `Init()`: Returns `nil`.
    - `SetBindings(k []key.Binding)`: Sets the list of key bindings to be displayed by the help dialog.
    - `Update(msg tea.Msg)`: Handles `tea.WindowSizeMsg` to update its dimensions. Other messages are typically not processed by this static display component.
    - `View() string`: Renders the help dialog.
        - It calls `render()` to get the formatted list of key bindings.
        - Prepends a "Keyboard Shortcuts" title.
        - Wraps the entire content in a styled, bordered box.
- **Helper Functions**:
    - `removeDuplicateBindings(bindings []key.Binding) []key.Binding`: Processes a list of key bindings to remove duplicates based on the key sequence (e.g., if "ctrl+c" is bound multiple times, only the last one encountered in reverse order is kept). This helps in overriding default bindings with more specific ones.
    - `render() string`: The core rendering logic for the key bindings.
        - Takes the `h.keys` (after duplicate removal).
        - Arranges them into groups (columns) to fit within the available width. It aims for a layout with a fixed number of rows per column group (`rows = 12 - 2`).
        - For each group, it creates two columns: one for the key sequences (e.g., "ctrl+c") and one for their descriptions (e.g., "copy").
        - Styles keys and descriptions differently using `lipgloss` and theme colors.
        - Handles alignment and padding within columns.
        - Joins the column groups horizontally.
        - Includes a workaround for a `lipgloss` issue (#209) when joining multiple pairs if the last pair is shorter, by using `lipgloss.Place` for the last pair.
- **`HelpCmp` interface & `NewHelpCmp()` constructor**: Standard component setup.

## Dependencies and Interactions

- Uses `key.Binding` from `charmbracelet/bubbles` to represent keyboard shortcuts.
- Uses `lipgloss` extensively for styling the dialog box, title, keys, and descriptions.
- Relies on `theme.CurrentTheme()` for colors and `styles` for base styling and common styles like `Bold()`.
- The key bindings to be displayed are provided externally via the `SetBindings()` method, typically by a parent component that aggregates bindings from various active TUI elements.

## Purpose

This component provides a user-friendly way to display all relevant keyboard shortcuts in a dynamically formatted layout. It helps users discover and learn how to interact with the TUI efficiently. The duplicate removal logic ensures that the help view isn't cluttered and reflects the effective key bindings when overrides are present.

# theme.go (in internal/tui/components/dialog)

## Overview

The `theme.go` file defines the `ThemeDialogCmp` TUI component. This component presents a dialog that lists all available application themes, allowing the user to select and apply a new theme. It interacts with the `internal/tui/theme` package to get available themes and set the chosen one.

## Key Components

- **Message Types**:
    - `ThemeChangedMsg`: Sent when a user selects and successfully applies a new theme. Contains `ThemeName (string)`.
    - `CloseThemeDialogMsg`: Sent when the dialog is closed without changing the theme.
- **`ThemeDialog` interface**: Public interface for the theme selection dialog component.
- **`themeDialogCmp` struct**: The main `tea.Model` for the theme dialog.
    - Manages `themes ([]string)` (list of available theme names), `selectedIdx` (for the list), `width`, `height`, and `currentTheme` (name of the currently active theme).
- **`themeKeyMap` struct**: Defines key bindings for navigation (Up/Down/J/K), selection (Enter), and closing (Escape).
- **Core Functionality**:
    - `Init()`: Loads available themes from `theme.AvailableThemes()` and sets `selectedIdx` to the `theme.CurrentThemeName()`.
    - `Update(msg tea.Msg)`: Handles messages:
        - `tea.KeyMsg`: Processes navigation keys to change `selectedIdx`.
            - `enter`: If the selected theme is different from the current one, attempts to set it using `theme.SetTheme()`. If successful, sends `ThemeChangedMsg`. If same or error, or if no themes, may send `CloseThemeDialogMsg` or `util.ReportError`.
            - `escape`: Sends `CloseThemeDialogMsg`.
        - `tea.WindowSizeMsg`: Updates component dimensions.
    - `View() string`: Renders the dialog:
        - Displays a title "Select Theme".
        - Lists the names of available themes.
        - Highlights the `selectedIdx` theme.
        - The dialog is styled with a rounded border using the current theme's colors.
    - `BindingKeys() []key.Binding`: Returns the dialog's key bindings.
- `NewThemeDialogCmp() ThemeDialog`: Constructor for `themeDialogCmp`.

## Dependencies and Interactions

- Uses `theme.AvailableThemes()`, `theme.CurrentThemeName()`, and `theme.SetTheme()` from `internal/tui/theme` to manage theme data and application.
- Uses `key.Binding` from `charmbracelet/bubbles`.
- Uses `lipgloss` for styling.
- Relies on `styles.BaseStyle()` for base styling.
- Communicates theme changes or closure to a parent model via `ThemeChangedMsg` or `CloseThemeDialogMsg`. The parent (likely the main TUI model) would then typically trigger a re-render of the entire UI to apply the new theme.

## Purpose

This component provides a user interface for theme selection, allowing users to customize the visual appearance of the OpenCode TUI by choosing from a predefined list of themes.

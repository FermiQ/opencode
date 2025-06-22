# complete.go (in internal/tui/components/dialog)

## Overview

The `complete.go` file defines the `CompletionDialogCmp` TUI component. This dialog is designed to provide completion suggestions to the user, typically triggered by a specific character (like "@") in an input field (e.g., the chat editor). It uses a `CompletionProvider` interface to fetch completion items based on the user's query and displays them in a list for selection.

## Key Components

- **`CompletionItem` struct & `CompletionItemI` interface**:
    - `CompletionItem` stores the `Title` (display string) and `Value` (actual insertion string) for a completion.
    - `CompletionItemI` is an interface (embedding `utilComponents.SimpleListItem`) that `CompletionItem` implements, defining methods like `GetValue()` and `DisplayValue()`.
    - `NewCompletionItem()` is a constructor for `CompletionItemI`.
- **`CompletionProvider` interface**: Defines the contract for a source of completion items.
    - `GetId() string`: Returns an ID for the provider.
    - `GetEntry() CompletionItemI`: Returns a default or entry-point completion item.
    - `GetChildEntries(query string) ([]CompletionItemI, error)`: Fetches completion items based on a query.
- **Message Types**:
    - `CompletionSelectedMsg`: Sent when a user selects a completion. Contains `SearchString` (what the user typed, e.g., "@some/path") and `CompletionValue` (the value to insert).
    - `CompletionDialogCloseMsg`: Sent when the dialog is closed without a selection.
- **`CompletionDialog` interface**: Public interface for the completion dialog component.
- **`completionDialogCmp` struct**: The main `tea.Model` for the completion dialog.
    - Manages `query` (the text after the trigger char), a `completionProvider`, `width`, `height`, a `pseudoSearchTextArea` (a `textarea.Model` used internally to capture input for the completion query, not directly visible as a full editor), and a `listView (utilComponents.SimpleList[CompletionItemI])` to display suggestions.
- **`completionDialogKeyMap` struct**: Defines key bindings (Tab/Enter to complete, Space/Esc/Backspace to cancel/close).
- **Core Functionality**:
    - `Init()`: Returns `nil`.
    - `Update(msg tea.Msg)`: Handles messages:
        - `tea.KeyMsg`:
            - If the `pseudoSearchTextArea` is focused:
                - Updates the text area with typed characters (excluding completion keys).
                - Extracts the query (text after the trigger like "@").
                - If the query changes, fetches new child entries from the `completionProvider` and updates the `listView`.
                - Forwards other keys to `listView` for navigation.
                - `Tab`/`Enter`: Calls `complete()` with the selected item from `listView`.
                - `Space`/`Esc`/`Backspace` (on empty input): Calls `close()`.
            - If not focused (likely the first key press that triggered the dialog, e.g., "@"):
                - Fetches initial entries, sets the typed key into `pseudoSearchTextArea`, and focuses it.
    - `View() string`: Renders the `listView` of completion items within a styled bordered box.
    - `SetWidth(width int)`: Sets the dialog width.
    - `BindingKeys() []key.Binding`: Returns its key bindings.
- **Helper Functions**:
    - `complete(item CompletionItemI) tea.Cmd`: Sends `CompletionSelectedMsg` and closes the dialog.
    - `close() tea.Cmd`: Resets state and sends `CompletionDialogCloseMsg`.
- `NewCompletionDialogCmp(completionProvider CompletionProvider) CompletionDialog`: Constructor.

## Dependencies and Interactions

- Uses `textarea.Model` from `charmbracelet/bubbles` for internal query input.
- Uses `utilComponents.SimpleList[CompletionItemI]` for displaying suggestions.
- Relies on a `CompletionProvider` (e.g., `completions.FileAndFolderContextGroup`) to supply completion items.
- Communicates selection or closure to parent components (e.g., `page.ChatPage`) via `CompletionSelectedMsg` or `CompletionDialogCloseMsg`.
- Uses `lipgloss`, `styles`, and `theme` for styling.

## Purpose

This component provides an interactive way for users to get suggestions and complete input, particularly for "@" mentions in the chat, by interfacing with pluggable completion providers. It enhances user input efficiency by offering contextually relevant options.

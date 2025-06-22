# permission.go (in internal/tui/components/dialog)

## Overview

The `permission.go` file defines the `PermissionDialogCmp` TUI component. This dialog is displayed to the user when an AI agent's tool (like "bash", "edit", "write", "fetch") requests permission to perform an action. The dialog presents information about the requested action, including the tool name, relevant path, and often a diff or command preview. The user can then "Allow" the action once, "Allow for session" (persisting the grant for similar requests in the current session), or "Deny" it.

## Key Components

- **`PermissionAction` (string enum)**: Defines user responses: `PermissionAllow`, `PermissionAllowForSession`, `PermissionDeny`.
- **`PermissionResponseMsg` struct**: A `tea.Msg` sent when the user responds to the dialog. Contains the original `permission.PermissionRequest` and the chosen `PermissionAction`.
- **`PermissionDialogCmp` interface**: Public interface for the permission dialog.
- **`permissionDialogCmp` struct**: The main `tea.Model` for the dialog.
    - Manages `width`, `height`, the current `permission.PermissionRequest` being displayed, a `viewport.Model` (`contentViewPort`) for scrollable content (like diffs or long commands), `selectedOption` (for the Allow/AllowSession/Deny buttons), and caches for rendered diffs (`diffCache`) and markdown (`markdownCache`).
- **`permissionsMapping` struct**: Defines key bindings (Left/Right/Tab to switch options, Enter/Space to confirm, A/S/D for direct actions).
- **Core Functionality**:
    - `Init()`: Initializes the content viewport.
    - `Update(msg tea.Msg)`: Handles messages:
        - `tea.WindowSizeMsg`: Updates dimensions and triggers `SetSize`.
        - `tea.KeyMsg`: Processes key presses according to `permissionsKeys` to change `selectedOption` or call `selectCurrentOption()`. Forwards other keys to the `contentViewPort` if it's active (e.g., for scrolling).
    - `View() string`: Renders the dialog:
        - A title "Permission Required".
        - A header section (`renderHeader()`) showing tool name and path.
        - Content specific to the tool (`renderBashContent()`, `renderEditContent()`, etc.), often displayed within the `contentViewPort`. This content might be a command preview, a diff of changes, or the URL to be fetched.
        - Action buttons ("Allow (a)", "Allow for session (s)", "Deny (d)") with the current `selectedOption` highlighted (`renderButtons()`).
    - `SetPermissions(permission permission.PermissionRequest) tea.Cmd`: Sets the permission request to be displayed and calls `SetSize`.
    - `SetSize()`: Dynamically adjusts the dialog's width and height based on the content of the permission request and the overall window size.
- **Helper Functions**:
    - `selectCurrentOption()`: Sends a `PermissionResponseMsg` based on `selectedOption`.
    - `renderButtons()`: Renders the Allow/AllowSession/Deny buttons with selection highlighting.
    - `renderHeader()`: Renders common permission request details (tool, path) and tool-specific headers.
    - `renderXContent()` (e.g., `renderBashContent`, `renderEditContent`): Renders the specific details of the permission request (command, diff, URL) often using markdown or diff formatting, utilizing the caches.
    - `GetOrSetDiff()`, `GetOrSetMarkdown()`: Cache helpers to avoid re-rendering expensive content.
- `NewPermissionDialogCmp() PermissionDialogCmp`: Constructor.

## Dependencies and Interactions

- Uses `permission.PermissionRequest` from `internal/permission`.
- Uses tool parameter types (e.g., `tools.BashPermissionsParams`, `tools.EditPermissionsParams`) from `internal/llm/tools`.
- Uses `diff.FormatDiff` from `internal/diff` to render diffs.
- Uses `viewport.Model` from `charmbracelet/bubbles` for scrollable content.
- Uses `lipgloss`, `styles`, and `theme` for TUI styling.
- Communicates the user's decision to a parent model (likely the main TUI or a dialog manager) by sending a `PermissionResponseMsg`. The parent is then responsible for calling the appropriate methods on the `permission.Service`.
- Display of this dialog is typically triggered when the `permission.Service` publishes a `PermissionRequest` event.

## Purpose

This component is a critical part of the application's security and user control model. It ensures that potentially impactful actions taken by the AI agent are explicitly approved by the user, providing transparency and preventing unintended changes. The detailed display of what the tool intends to do (e.g., showing the exact command or diff) allows the user to make an informed decision.

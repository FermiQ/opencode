# sidebar.go (in internal/tui/components/chat)

## Overview

The `sidebar.go` file defines the `sidebarCmp` TUI component. This component is responsible for displaying contextual information related to the current chat session. This includes the session title, a static header (logo, repo, cwd), LSP configuration, and a list of files modified within the current session along with their additions and removals (diff stats). It subscribes to file history events to dynamically update the list of modified files.

## Key Components

- **`sidebarCmp` struct**: The main `tea.Model` for the sidebar.
    - Manages `width`, `height`, the current `session.Session`, a `history.Service` instance, and a map `modFiles` to store paths and diff stats (additions, removals) of modified files.
- **Core Functionality**:
    - `Init()`: If `history.Service` is available, it subscribes to file events from the history service, loads initial modified files for the current session by comparing latest versions against their initial versions, and returns a command to listen for these file events.
    - `Update(msg tea.Msg)`: Handles messages:
        - `SessionSelectedMsg`: Updates the current session and reloads modified files.
        - `pubsub.Event[session.Session]`: Updates the displayed session if the current session is updated.
        - `pubsub.Event[history.File]`: If a file in the current session is updated, it calls `processFileChanges` to update its diff stats in `modFiles`. It then re-subscribes to file events.
    - `View() string`: Renders the sidebar content by vertically joining:
        - A static header (logo, repo, cwd - likely from `chat.go` in the same package).
        - The current session title (`sessionSection()`).
        - LSP configuration details (`lspsConfigured()` - from `chat.go`).
        - The list of modified files with their diff stats (`modifiedFiles()`).
    - `SetSize(width, height int) tea.Cmd`: Updates the component's dimensions.
- **Helper Functions**:
    - `sessionSection()`: Renders the current session title.
    - `modifiedFile(...)`: Renders a single line for a modified file, showing its path and diff stats (+additions, -removals).
    - `modifiedFiles()`: Renders the "Modified Files:" header and the list of all modified files, sorted alphabetically. Shows "No modified files" if none.
    - `loadModifiedFiles(ctx context.Context)`: Fetches all latest and initial file versions for the current session from `history.Service`, calculates diffs between them, and populates `modFiles`.
    - `processFileChanges(ctx context.Context, file history.File)`: Updates or removes a specific file's entry in `modFiles` when a new version is received.
    - `findInitialVersion(ctx context.Context, path string) (history.File, error)`: Helper to find the "initial" version of a file in the history.
    - `getDisplayPath(path string)`: Helper to make file paths relative to the working directory for display.
- `NewSidebarCmp(session session.Session, history history.Service) tea.Model`: Constructor for `sidebarCmp`.

## Dependencies and Interactions

- Uses `session.Session` and `history.Service` to get session data and file versions.
- Subscribes to `pubsub` events from `history.Service` to react to file changes.
- Uses `diff.GenerateDiff` to calculate additions/removals between file versions.
- Uses helper functions from `chat.go` (in the same package) like `header()` and `lspsConfigured()` for parts of its view.
- Relies on `config.WorkingDirectory()` for path manipulation.
- Uses `lipgloss` and `styles` for TUI styling.

## Purpose

This component provides a persistent contextual sidebar within the chat page, offering the user an at-a-glance view of the current session's details and a summary of file modifications made during that session. This helps track changes and understand the scope of work being done by the AI agent.

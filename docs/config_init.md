# init.go

## Overview

The `init.go` file, part of the `config` package, handles a specific aspect of project initialization status. It provides functionality to check if a project (specifically, its OpenCode configuration within the `.opencode` data directory) has been "initialized," which seems to be a one-time setup marker. This is used to determine if an initial setup dialog or process should be shown to the user.

## Key Components

### Constants
- `InitFlagFilename`: The name of the flag file ("init") that is created in the application's data directory (`.opencode/init`) to signify that the initialization process has occurred.

### Structs
- `ProjectInitFlag`: A struct likely intended for future use or a different approach, as it's defined (`Initialized bool`) but not directly used in the current logic for reading/writing the flag. The presence of the `InitFlagFilename` file itself acts as the boolean flag.

### Functions
- `ShouldShowInitDialog() (bool, error)`:
    - Checks if the global configuration `cfg` is loaded.
    - Constructs the path to the initialization flag file (e.g., `~/.opencode/init`).
    - If the flag file exists, it returns `false` (dialog should not be shown).
    - If the flag file does not exist, it returns `true` (dialog should be shown).
    - Returns an error if there's an issue checking the file status, other than it not existing.
- `MarkProjectInitialized() error`:
    - Checks if the global configuration `cfg` is loaded.
    - Constructs the path to the initialization flag file.
    - Creates an empty file at that path. This action "marks" the project/configuration as initialized.
    - Returns an error if the file creation fails.

## Important Variables/Constants
- `InitFlagFilename`: Defines the simple mechanism (presence of a file) for tracking initialization.

## Usage Examples

This functionality is likely used early in the application startup, possibly in the TUI setup, to guide the user through an initial configuration if it's their first time or if the init flag is missing.

```go
// Conceptual usage in TUI or main app setup:
import "github.com/opencode-ai/opencode/internal/config"

// ... after config.Load() has been called ...

showDialog, err := config.ShouldShowInitDialog()
if err != nil {
    // Handle error
    log.Fatalf("Failed to check init status: %v", err)
}

if showDialog {
    // Display some initial setup dialog to the user.
    // If the user completes the setup:
    err := config.MarkProjectInitialized()
    if err != nil {
        // Handle error
        log.Printf("Failed to mark project as initialized: %v", err)
    }
}
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - Relies on the global `cfg` variable (from `config.go` in the same package) being loaded, primarily to get `cfg.Data.Directory`.
- **External Libraries:**
    - `os`: For file system operations (checking file status `os.Stat`, creating files `os.Create`).
    - `path/filepath`: For joining path components.
    - `fmt`: For error formatting.
- **Interactions:**
    - Interacts with the file system by checking for and creating a specific flag file within the application's data directory (e.g., `~/.opencode/init`).
    - The functions `ShouldShowInitDialog` and `MarkProjectInitialized` provide a simple mechanism for other parts of the application (likely the TUI) to manage a one-time initialization state.

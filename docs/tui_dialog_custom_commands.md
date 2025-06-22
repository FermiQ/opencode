# custom_commands.go (in internal/tui/components/dialog)

## Overview

The `custom_commands.go` file defines the logic for loading custom user-defined commands. These commands are sourced from Markdown (`.md`) files located in specific directories:
1.  `$XDG_CONFIG_HOME/opencode/commands/` (user-global commands)
2.  `$HOME/.opencode/commands/` (user-global commands, fallback)
3.  `<project_root>/.opencode/commands/` (project-specific commands, using `cfg.Data.Directory`)

Each `.md` file's content is treated as the command template. The system can identify named arguments within these templates (e.g., `$ARG_NAME`) and will prompt the user for their values using the `MultiArgumentsDialogCmp` if such arguments are found.

## Key Components

- **Constants**:
    - `UserCommandPrefix ("user:")`: Prepended to IDs of commands loaded from user-global directories.
    - `ProjectCommandPrefix ("project:")`: Prepended to IDs of commands loaded from the project's command directory.
- **`namedArgPattern` (regexp.Regexp)**: A regular expression (`\$([A-Z][A-Z0-9_]*)`) used to find named arguments (e.g., `$MY_VARIABLE`, `$ANOTHER_ARG`) in command templates.
- **Functions**:
    - `LoadCustomCommands() ([]Command, error)`:
        - The main function to load all custom commands.
        - It determines paths for user-global (XDG and home fallback) and project-specific command directories.
        - Calls `loadCommandsFromDir()` for each of these locations and aggregates the results.
        - Handles errors during loading from one location by printing a warning but continuing to load from others.
    - `loadCommandsFromDir(commandsDir string, prefix string) ([]Command, error)`:
        - Reads all `.md` files from the specified `commandsDir`.
        - If `commandsDir` doesn't exist, it creates it and returns an empty list.
        - For each `.md` file:
            - The command ID is derived from its relative path within `commandsDir` (e.g., `subdir/mycommand` becomes `prefix:subdir:mycommand`).
            - A `Command` struct (defined in `commands.go`) is created:
                - `ID` and `Title` are set using the derived ID and prefix.
                - `Description` indicates the source file.
                - `Handler`: A function that, when called:
                    - Reads the content of the Markdown file (the command template).
                    - Uses `namedArgPattern` to find all named arguments (e.g., `$NAME`).
                    - If arguments are found, it sends a `ShowMultiArgumentsDialogMsg` to prompt the user for these argument values.
                    - If no arguments are found, it sends a `CommandRunCustomMsg` directly with the command content.
- **`CommandRunCustomMsg` struct**: A `tea.Msg` sent when a custom command (with its arguments filled, if any) is ready to be executed.
    - `Content (string)`: The command template with arguments substituted.
    - `Args (map[string]string)`: The map of argument names to their user-provided values (nil if no arguments).

## Dependencies and Interactions

- Uses `Command` struct and `ShowMultiArgumentsDialogMsg` from the same `dialog` package.
- Relies on `config.Get()` for the project's data directory path.
- Interacts with the filesystem (`os.Stat`, `os.MkdirAll`, `os.ReadFile`, `filepath.Walk`) to find and read command files.
- The `Handler` for custom commands ultimately triggers either a `ShowMultiArgumentsDialogMsg` (if arguments are needed) or a `CommandRunCustomMsg`. These messages are then handled by other parts of the TUI (likely `page.ChatPage` or a dialog manager) to either show the arguments dialog or process the command for execution by the AI agent.

## Purpose

This system allows users to create reusable, templated prompts or commands that can be easily invoked within the OpenCode TUI. By supporting named arguments, these custom commands can be made flexible and interactive, prompting users for necessary inputs before the command content is finalized and sent to the AI agent. This promotes a more efficient and personalized workflow.

# root.go

## Overview

The `root.go` file is the entry point for the `opencode` command-line interface (CLI). It defines the root command, handles command-line arguments, initializes the application configuration, database connection, and starts either the interactive TUI (Terminal User Interface) or a non-interactive mode based on the provided flags.

## Key Components

### Variables
- `rootCmd`: This is the main command for the CLI, configured using the `cobra` library. It defines the usage, short and long descriptions, examples, and the main execution logic (`RunE`).

### Functions
- `RunE(cmd *cobra.Command, args []string) error`: The core execution logic for the `opencode` command. It handles:
    - Help and version flags.
    - Loading configuration (`config.Load`).
    - Setting the current working directory.
    - Validating output format for non-interactive mode.
    - Connecting to the database (`db.Connect`).
    - Initializing the main application (`app.New`).
    - Initializing MCP (Multi-Context Prompt) tools (`initMCPTools`).
    - Running in non-interactive mode if a prompt is provided.
    - Setting up and running the TUI in interactive mode, including managing event subscriptions.
    - Graceful shutdown of application components.
- `attemptTUIRecovery(program *tea.Program)`: A function intended to handle panics within the TUI, currently it quits the program.
- `initMCPTools(ctx context.Context, app *app.App)`: Initializes the MCP tools by fetching them in a separate goroutine.
- `setupSubscriber[T any](ctx context.Context, wg *sync.WaitGroup, name string, subscriber func(context.Context) <-chan pubsub.Event[T], outputCh chan<- tea.Msg)`: A generic function to set up a subscription to a pub/sub channel and forward events to the TUI's message channel. It handles graceful shutdown and error recovery.
- `setupSubscriptions(app *app.App, parentCtx context.Context) (chan tea.Msg, func())`: Sets up all necessary event subscriptions (logging, sessions, messages, permissions, coderAgent) for the TUI and returns a channel for TUI messages and a cleanup function.
- `Execute()`: This function is called by `main.go` to execute the `rootCmd`. It handles errors from command execution and exits with status 1 if an error occurs.
- `init()`: This Go initialization function is used to define and register flags for the `rootCmd` using `cobra`. Flags include `--help`, `--version`, `--debug`, `--cwd`, `--prompt`, `--output-format`, and `--quiet`. It also registers a custom completion function for the `output-format` flag.

## Important Variables/Constants

This file primarily uses local variables within functions or the `rootCmd` variable. There are no package-level constants or globally critical variables defined directly for export, other than `rootCmd` itself which is central to the CLI's operation.

## Usage Examples

The `rootCmd` definition includes examples of how to use the `opencode` command:

```
# Run in interactive mode
opencode

# Run with debug logging
opencode -d

# Run with debug logging in a specific directory
opencode -d -c /path/to/project

# Print version
opencode -v

# Run a single non-interactive prompt
opencode -p "Explain the use of context in Go"

# Run a single non-interactive prompt with JSON output format
opencode -p "Explain the use of context in Go" -f json
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/app`: For the main application logic.
    - `github.com/opencode-ai/opencode/internal/config`: For loading application configuration.
    - `github.com/opencode-ai/opencode/internal/db`: For database connection and operations.
    - `github.com/opencode-ai/opencode/internal/format`: For output formatting options.
    - `github.com/opencode-ai/opencode/internal/llm/agent`: For MCP tools.
    - `github.com/opencode-ai/opencode/internal/logging`: For application-wide logging.
    - `github.com/opencode-ai/opencode/internal/pubsub`: For event subscription mechanisms.
    - `github.com/opencode-ai/opencode/internal/tui`: For the terminal user interface.
    - `github.com/opencode-ai/opencode/internal/version`: For accessing the application version.
- **External Libraries:**
    - `github.com/charmbracelet/bubbletea`: For building the TUI.
    - `github.com/lrstanley/bubblezone`: For mouse support in the TUI.
    - `github.com/spf13/cobra`: For creating the CLI structure and handling commands/flags.
- **Interactions:**
    - Initializes and orchestrates major components of the application (`app`, `db`, `config`, `tui`).
    - Parses command-line arguments to determine application behavior (interactive vs. non-interactive, debug mode, etc.).
    - Forwards system events (logs, session updates, AI messages) to the TUI via pub/sub channels.
    - Manages the lifecycle of the application, including startup and graceful shutdown.

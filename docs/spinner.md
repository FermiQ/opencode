# spinner.go

## Overview

The `spinner.go` file, also within the `internal/format` package, provides a terminal spinner utility. This spinner is intended for use in non-interactive command-line operations to indicate that a process is ongoing (e.g., "Thinking..."). It wraps the `spinner` component from the `charmbracelet/bubbles` library and manages its lifecycle using `bubbletea`.

## Key Components

### Structs
- `Spinner`: The main struct for managing the spinner.
    - `model (spinner.Model)`: The underlying `bubbles/spinner` model.
    - `done (chan struct{})`: A channel used to signal that the spinner's goroutine has completed.
    - `prog (*tea.Program)`: The `bubbletea` program instance running the spinner.
    - `ctx (context.Context)`: A context used to manage the spinner's lifecycle.
    - `cancel (context.CancelFunc)`: The cancel function for `ctx`, used to stop the spinner.
- `spinnerModel`: A `bubbletea.Model` implementation that wraps the `bubbles/spinner.Model` and handles its updates and view logic.
    - `spinner (spinner.Model)`: The actual spinner component.
    - `message (string)`: The message to display next to the spinner (e.g., "Thinking...").
    - `quitting (bool)`: A flag to indicate if the spinner is in the process of quitting, to avoid rendering after a quit signal.

### Types
- `quitMsg (struct{})`: An empty struct used as a `tea.Msg` to signal the `spinnerModel` to quit.

### Functions
- `(m spinnerModel) Init() tea.Cmd`: Implements `tea.Model.Init`. Returns the initial command to start the spinner's tick.
- `(m spinnerModel) Update(msg tea.Msg) (tea.Model, tea.Cmd)`: Implements `tea.Model.Update`. Handles:
    - `tea.KeyMsg`: Sets `quitting` to true and returns `tea.Quit` command (though key presses are unlikely in typical non-interactive use).
    - `spinner.TickMsg`: Updates the spinner animation and returns the next tick command.
    - `quitMsg`: Sets `quitting` to true and returns `tea.Quit` command.
- `(m spinnerModel) View() string`: Implements `tea.Model.View`. Renders the current spinner animation frame and the associated message. Returns an empty string if `quitting` is true.
- `NewSpinner(message string) *Spinner`: Constructor for the `Spinner`.
    - Initializes a `bubbles/spinner.Model` (using `spinner.Dot` style).
    - Creates a `context.Context` and its `cancel` function for lifecycle management.
    - Initializes the `spinnerModel` with the spinner and the provided `message`.
    - Creates a new `tea.Program` with the `spinnerModel`, configured to output to `os.Stderr` and to operate `WithoutCatchPanics` (as it's a simple, controlled component).
    - Returns the configured `*Spinner` instance.
- `(s *Spinner) Start()`: Starts the spinner animation.
    - It launches a goroutine that runs the `tea.Program` (`s.prog.Run()`).
    - Inside this goroutine, another goroutine is launched to listen for context cancellation (`s.ctx.Done()`). When the context is cancelled (by `Stop()`), it sends a `quitMsg` to the `tea.Program` to gracefully shut it down.
    - The outer goroutine waits for `s.prog.Run()` to complete and then closes the `s.done` channel.
- `(s *Spinner) Stop()`: Stops the spinner animation.
    - Calls `s.cancel()` to cancel the context, which signals the spinner's `tea.Program` to quit.
    - Waits for the `s.done` channel to be closed, ensuring the spinner's goroutine has finished before returning.

## Important Variables/Constants
This file does not define exported package-level constants or variables beyond the types.

## Usage Examples

Using the spinner for a long-running operation:
```go
import "github.com/opencode-ai/opencode/internal/format"
import "time"
import "fmt"

func main() {
    spinner := format.NewSpinner("Processing your request...")
    spinner.Start()

    // Simulate a long-running task
    time.Sleep(3 * time.Second)

    spinner.Stop()
    fmt.Println("Done!")
}
```
When run, this would show "⠋ Processing your request..." (with the dot spinning) on `stderr` for 3 seconds, then clear it (implicitly, as the Bubble Tea program exits and the line is overwritten or scrolled away), and finally print "Done!" to `stdout`.

## Dependencies and Interactions

- **Internal Dependencies:** None beyond standard Go packages.
- **External Libraries:**
    - `github.com/charmbracelet/bubbles/spinner`: The core spinner component.
    - `github.com/charmbracelet/bubbletea`: The TUI framework used to run and manage the spinner.
- **Interactions:**
    - Creates a `bubbletea` program that runs in a separate goroutine to render the spinner to `os.Stderr`.
    - Uses Go's `context` package for managing the lifecycle and graceful shutdown of the spinner.
    - Designed for non-interactive scenarios where visual feedback for ongoing processes is desired without a full TUI.

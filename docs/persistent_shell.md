# shell.go (in internal/llm/tools/shell)

## Overview

The `shell.go` file, located in the `internal/llm/tools/shell` package, implements a `PersistentShell`. This component provides a way to execute shell commands within a single, long-lived shell session. This is crucial for tools like the "bash" tool, as it allows commands to maintain state (e.g., current working directory, environment variables, activated virtual environments) across multiple invocations. The implementation uses temporary files to capture stdout, stderr, exit status, and the current working directory after each command.

## Key Components

### Structs
- `PersistentShell`: Manages a persistent shell process.
    - `cmd (*exec.Cmd)`: The underlying `exec.Cmd` for the shell process.
    - `stdin (*os.File)`: The stdin pipe to the shell process, used to send commands.
    - `isAlive (bool)`: Flag indicating if the shell process is currently running.
    - `cwd (string)`: The current working directory of the persistent shell. This is updated after commands like `cd`.
    - `mu (sync.Mutex)`: A mutex to protect access to the shell, especially when executing commands.
    - `commandQueue (chan *commandExecution)`: A channel to serialize command execution requests to the single shell process.
- `commandExecution`: An unexported struct to bundle a command, its timeout, a result channel, and a context for a single execution request.
- `commandResult`: An unexported struct to return the results of a command execution (stdout, stderr, exit code, interruption status, error).

### Package Variables
- `shellInstance (*PersistentShell)`: A package-level singleton instance of the persistent shell.
- `shellInstanceOnce (sync.Once)`: Ensures that `shellInstance` is initialized only once.

### Functions
- `GetPersistentShell(workingDir string) *PersistentShell`:
    - The public accessor for the singleton `PersistentShell` instance.
    - Uses `shellInstanceOnce.Do` for thread-safe initialization on the first call.
    - If the existing `shellInstance` is not alive, it re-initializes it with its last known `cwd`.
    - This ensures that tools across the application share the same persistent shell session for a given initial `workingDir`.
- `newPersistentShell(cwd string) *PersistentShell`:
    - The actual constructor for `PersistentShell`.
    - Determines the shell path (from config `cfg.Shell.Path`, then `SHELL` env var, then `/bin/bash`) and arguments (from config `cfg.Shell.Args`, then default `["-l"]`).
    - Creates an `exec.Cmd` for the shell.
    - Sets the command's working directory (`cmd.Dir`) to the provided `cwd`.
    - Gets a pipe to the shell's stdin.
    - Appends `GIT_EDITOR=true` to the shell's environment (likely to prevent git commands from opening an interactive editor).
    - Starts the shell command (`cmd.Start()`).
    - Initializes the `PersistentShell` struct, including the `commandQueue`.
    - Launches two goroutines:
        - One to run `shell.processCommands()` which listens on `commandQueue` for commands to execute.
        - One to wait for the shell process (`cmd.Wait()`) to exit, which then sets `isAlive` to false and closes the `commandQueue`.
- `(s *PersistentShell) processCommands()`:
    - Runs in a goroutine, continuously reading `commandExecution` requests from `s.commandQueue`.
    - For each request, it calls `s.execCommand()` and sends the `commandResult` back on the request's `resultChan`.
    - Includes a `defer recover()` to catch panics within this processing loop, mark the shell as not alive, and close the queue.
- `(s *PersistentShell) execCommand(command string, timeout time.Duration, ctx context.Context) commandResult`:
    - The core internal method for executing a single command. It's protected by `s.mu.Lock()`.
    - Creates temporary files for stdout, stderr, exit status, and the new current working directory.
    - Constructs a wrapper command: `eval <quoted_command> < /dev/null > <stdout_file> 2> <stderr_file>; EXEC_EXIT_CODE=$?; pwd > <cwd_file>; echo $EXEC_EXIT_CODE > <status_file>`. This structure captures all necessary outputs and status into files.
    - Writes this `fullCommand` to the shell's `s.stdin`.
    - Monitors for completion by periodically checking if the `statusFile` exists and has content.
    - Also handles command `timeout` and `ctx.Done()` for cancellation. If timeout or cancellation occurs, it calls `s.killChildren()` to terminate any processes spawned by the command.
    - Reads the content of the temporary files to get stdout, stderr, exit code, and the new `cwd`.
    - Updates `s.cwd` if `cd` was used.
    - Returns a `commandResult`.
- `(s *PersistentShell) killChildren()`: Attempts to find and terminate child processes of the main shell process using `pgrep -P <shell_pid>` and `kill`.
- `(s *PersistentShell) Exec(ctx context.Context, command string, timeoutMs int) (string, string, int, bool, error)`:
    - The public method for executing a command.
    - Sends a `commandExecution` request to the `s.commandQueue` and waits for the result on the `resultChan`.
    - Returns the stdout, stderr, exit code, interruption status, and any error from the `commandResult`.
- `(s *PersistentShell) Close()`:
    - Attempts to gracefully exit the shell by writing "exit\n" to its stdin.
    - Kills the shell process.
    - Marks the shell as not alive.
- `shellQuote(s string) string`: Quotes a string for safe use in a shell command.
- `readFileOrEmpty(path string) string`, `fileExists(path string) bool`, `fileSize(path string) int64`: Utility functions for interacting with the temporary files.

## Important Variables/Constants
- `shellInstance` and `shellInstanceOnce`: Manage the singleton nature of the persistent shell.

## Usage Examples

The `PersistentShell` is primarily used by the "bash" tool (`internal/llm/tools/bash.go`).

```go
// Conceptual usage within the bash tool:
// import "github.com/opencode-ai/opencode/internal/llm/tools/shell"
// import "github.com/opencode-ai/opencode/internal/config"

// currentWorkDir := config.WorkingDirectory()
// ps := shell.GetPersistentShell(currentWorkDir)

// userCommand := "echo 'Hello from persistent shell'"
// timeoutMilliseconds := 5000
// stdout, stderr, exitCode, interrupted, err := ps.Exec(context.Background(), userCommand, timeoutMilliseconds)

// if err != nil { /* handle error */ }
// fmt.Printf("Stdout: %s\nStderr: %s\nExit Code: %d\nInterrupted: %v\n", stdout, stderr, exitCode, interrupted)

// Another command (state like CWD would persist if 'cd' was used previously)
// stdout, stderr, _, _, _ = ps.Exec(context.Background(), "pwd", 1000)
// fmt.Printf("Current CWD in shell: %s\n", stdout)
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: For getting the initial working directory and shell configuration (path, args).
- **External Libraries:**
    - `sync`, `os`, `os/exec`, `path/filepath`, `strings`, `syscall`, `time`, `context`, `errors`, `fmt`: Standard Go libraries.
- **Interactions:**
    - Manages a long-running external shell process (e.g., bash, zsh).
    - Uses a command queue to serialize execution, ensuring only one command runs at a time in the persistent shell.
    - Relies on writing commands to the shell's stdin and reading outputs/status from temporary files. This is a common pattern for interacting with interactive shells non-interactively.
    - Provides a mechanism to update its internal tracking of the shell's current working directory (`s.cwd`) if a command changes it (e.g., `cd`).
    - Attempts to clean up child processes on timeout or cancellation.
    - The `GIT_EDITOR=true` environment variable is set to prevent interactive prompts from git.

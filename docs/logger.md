# logger.go (in internal/logging)

## Overview

The `logger.go` file, part of the `internal/logging` package, provides a set of convenient wrapper functions around Go's standard structured logging library, `slog`. It offers leveled logging methods (Info, Debug, Warn, Error) and corresponding "Persist" versions that likely add a special attribute to log entries to indicate they should always be persisted, regardless of TUI display filters. Additionally, it includes a robust `RecoverPanic` utility function for graceful panic handling, logging, and optional cleanup.

## Key Components

### Functions

- **Standard Logging Wrappers**:
    - `Info(msg string, args ...any)`: Logs a message at the Info level using `slog.Info()`.
    - `Debug(msg string, args ...any)`: Logs a message at the Debug level using `slog.Debug()`.
    - `Warn(msg string, args ...any)`: Logs a message at the Warn level using `slog.Warn()`.
    - `Error(msg string, args ...any)`: Logs a message at the Error level using `slog.Error()`.
    Each of these functions takes a message string and a variadic `args` parameter, which `slog` interprets as key-value pairs for structured logging.

- **Persistent Logging Wrappers**:
    - `InfoPersist(msg string, args ...any)`: Similar to `Info`, but appends a special key-value pair (`persistKeyArg`, `true`) to the arguments. This suggests that logs made with `Persist` functions are intended to be always recorded or shown, potentially bypassing some dynamic filtering in a UI log view. (Note: `persistKeyArg` is not defined in this file, implying it's defined elsewhere in the `logging` package, likely in `message.go` or `writer.go`).
    - `DebugPersist(msg string, args ...any)`: Persistent version of `Debug`.
    - `WarnPersist(msg string, args ...any)`: Persistent version of `Warn`.
    - `ErrorPersist(msg string, args ...any)`: Persistent version of `Error`.

- `RecoverPanic(name string, cleanup func())`:
    - A utility function designed to be used with `defer` to handle panics gracefully within goroutines or critical sections of code.
    - It takes a `name` string (to identify the context where the panic occurred) and an optional `cleanup` function.
    - If `recover()` captures a panic:
        1.  It logs the panic using `ErrorPersist()`, including the `name` and the panic value.
        2.  It creates a timestamped panic log file (e.g., `opencode-panic-<name>-<timestamp>.log`).
        3.  It writes the panic information, current time, and a full stack trace (from `debug.Stack()`) to this file.
        4.  It logs an informational message (using `InfoPersist()`) indicating where the panic details were saved.
        5.  If a `cleanup` function was provided, it executes it. This allows for resource release or state resetting after a panic.

## Important Variables/Constants

This file does not define any exported package-level constants or variables itself. It relies on the globally configured `slog.Logger` instance (which is set up in `internal/config/config.go`). The unexported `persistKeyArg` constant is implicitly used by the `*Persist` functions.

## Usage Examples

Standard logging:
```go
import "github.com/opencode-ai/opencode/internal/logging"

func DoSomething() {
    logging.Info("Starting operation", "user_id", 123)
    // ... do work ...
    if err != nil {
        logging.Error("Operation failed", "error", err, "input_data", someData)
        return
    }
    logging.Debug("Intermediate step successful", "details", "...")
    logging.InfoPersist("Important milestone reached and recorded")
}
```

Panic recovery in a goroutine:
```go
import "github.com/opencode-ai/opencode/internal/logging"

func MyGoroutineTask() {
    defer logging.RecoverPanic("MyGoroutineTask", func() {
        // Optional cleanup logic, e.g., close resources
        fmt.Println("Cleanup after panic in MyGoroutineTask")
    })

    // ... code that might panic ...
    panic("something went terribly wrong")
}
```
If `MyGoroutineTask` panics, `RecoverPanic` will log the error, save a detailed panic log file (e.g., `opencode-panic-MyGoroutineTask-20231027-103000.log`), and then print "Cleanup after panic in MyGoroutineTask".

## Dependencies and Interactions

- **Internal Dependencies:**
    - Implicitly depends on the `logging` package's global `slog` logger setup (configured in `internal/config/config.go`).
    - The `*Persist` functions likely interact with a filtering mechanism in the TUI's log viewer, where the `persistKeyArg` signals that the log entry should not be filtered out by default.
- **External Libraries:**
    - `log/slog`: The standard Go structured logging library. All logging calls are ultimately routed through `slog`.
    - `fmt`, `os`, `runtime/debug`, `time`: Standard Go libraries used for formatting, file operations, stack traces, and time.
- **Interactions:**
    - Provides a simplified and consistent logging interface for the entire application.
    - Enhances `slog` by adding "Persist" variants for important logs.
    - Offers a robust panic recovery mechanism that ensures panics are logged with detailed stack traces to separate files, aiding in debugging, and allows for custom cleanup actions.

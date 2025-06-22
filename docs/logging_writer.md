# writer.go (in internal/logging)

## Overview

The `writer.go` file, within the `internal/logging` package, defines a custom `io.Writer` implementation. This writer is intended to be used as the output for Go's standard `slog.Logger`. Its primary role is to parse log lines formatted in `logfmt` (which `slog.TextHandler` can produce), transform these parsed lines into structured `LogMessage` objects (defined in `logging/message.go`), and then publish these `LogMessage` objects via a package-level pub/sub broker. This allows other parts of the application, like the TUI, to subscribe to and display structured log messages.

## Key Components

### Constants
- `persistKeyArg ("$_persist")`: A special key used in log attributes. If this key is present (typically with a boolean true value), it signals that the log message should be marked with `Persist = true` in the `LogMessage`.
- `PersistTimeArg ("$_persist_time")`: Another special key. If present, its value (parsed as a `time.Duration`) is used to set the `PersistTime` field in the `LogMessage`.

### Structs
- `LogData`: An unexported struct that acts as a store and publisher for `LogMessage` objects.
    - `messages ([]LogMessage)`: A slice to store all log messages (though its primary use might be for immediate publishing rather than long-term storage here).
    - `Broker (*pubsub.Broker[LogMessage])`: A pub/sub broker (from `internal/pubsub`) that broadcasts `LogMessage` events.
    - `lock (sync.Mutex)`: A mutex to protect concurrent access to the `messages` slice.
- `writer`: An unexported struct that implements the `io.Writer` interface. It's an empty struct as its behavior is defined by its `Write` method.

### Package Variables
- `defaultLogData (*LogData)`: A package-level singleton instance of `LogData`. This is the central broker for all log messages processed by the `writer`.

### Functions
- `(l *LogData) Add(msg LogMessage)`:
    - Appends a `LogMessage` to the `l.messages` slice (with locking).
    - Publishes the `msg` as a `pubsub.CreatedEvent` via `l.Broker`.
- `(l *LogData) List() []LogMessage`:
    - Returns a copy of the `l.messages` slice (with locking). (Its utility might be limited if messages are primarily for real-time pub/sub).
- `(w *writer) Write(p []byte) (int, error)`:
    - Implements the `io.Writer` interface. This method is called by `slog.TextHandler` with log data formatted as `logfmt`.
    - Uses `logfmt.NewDecoder` to parse the input byte slice `p`.
    - For each key-value pair scanned by the decoder:
        - It populates a `LogMessage` struct.
        - It handles special keys:
            - "time": Parses the timestamp.
            - "level": Sets the log level (converted to lowercase).
            - "msg": Sets the main message content.
            - `persistKeyArg ("$_persist")`: Sets `LogMessage.Persist` to `true`.
            - `PersistTimeArg ("$_persist_time")`: Parses the duration and sets `LogMessage.PersistTime`.
            - Other keys are added to `LogMessage.Attributes`.
        - Calls `defaultLogData.Add(msg)` to store (briefly) and publish the structured `LogMessage`.
    - Returns the number of bytes written and any parsing error.
- `NewWriter() *writer`: Constructor for the custom `writer`.
- `Subscribe(ctx context.Context) <-chan pubsub.Event[LogMessage]`:
    - A public function that allows other packages to subscribe to log message events from the `defaultLogData.Broker`.
- `List() []LogMessage`:
    - A public function that returns all log messages currently held by `defaultLogData`.

## Important Variables/Constants
- `defaultLogData`: The central instance managing the storage and pub/sub broadcasting of parsed log messages.
- `persistKeyArg`, `PersistTimeArg`: Special keys that enable the `*Persist` logging functions in `logger.go` to pass persistence hints through the `slog` system to this writer.

## Usage Examples

The `writer` is primarily intended to be used when configuring the global `slog.Logger` instance, typically during application startup.

```go
// In internal/config/config.go (conceptual, as shown in the actual file):
// import "github.com/opencode-ai/opencode/internal/logging"
// import "log/slog"
// import "os"

// ...
// defaultLevel := slog.LevelInfo
// if cfg.Debug {
//     defaultLevel = slog.LevelDebug
// }
// logger := slog.New(slog.NewTextHandler(logging.NewWriter(), &slog.HandlerOptions{
//     Level: defaultLevel,
// }))
// slog.SetDefault(logger)
```
When `slog.Info("message", "key", "value", logging.persistKeyArg, true)` is called anywhere:
1. `slog`'s `TextHandler` formats this into a `logfmt` string (e.g., `time=... level=INFO msg="message" key=value $_persist=true`).
2. This string is passed to the `Write` method of the `logging.writer` instance.
3. The `Write` method parses this, creates a `LogMessage` with `Persist=true`.
4. `defaultLogData.Add()` publishes this `LogMessage` via its broker.
5. Any subscribed components (e.g., TUI log panel) receive the `LogMessage` event.

Subscribing to logs:
```go
// In a TUI component:
// import "github.com/opencode-ai/opencode/internal/logging"
// import "github.com/opencode-ai/opencode/internal/pubsub"

// logEventChan := logging.Subscribe(context.Background())
// go func() {
//     for event := range logEventChan {
//         if event.Type == pubsub.CreatedEvent {
//             displayLog(event.Data) // event.Data is a logging.LogMessage
//         }
//     }
// }()
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/pubsub`: For `pubsub.Broker` to broadcast `LogMessage` events.
    - Uses `LogMessage` and `Attr` structs defined in `logging/message.go`.
- **External Libraries:**
    - `github.com/go-logfmt/logfmt`: For parsing `logfmt` formatted log lines.
    - `bytes`, `context`, `fmt`, `strings`, `sync`, `time`: Standard Go libraries.
- **Interactions:**
    - Acts as the sink for `slog` output.
    - Parses `logfmt` data, which is the default text format for `slog.TextHandler` when writing to an `io.Writer` (instead of a terminal which might get colors).
    - Converts raw log lines into structured `LogMessage` objects.
    - Decouples log generation (`slog`) from log consumption by using a pub/sub model. This allows multiple parts of the application (e.g., TUI, potentially network log shippers in the future) to react to log events.
    - The special handling of `persistKeyArg` and `PersistTimeArg` allows metadata to be passed from high-level log calls (e.g., `logging.InfoPersist`) through `slog` to this custom processing logic.

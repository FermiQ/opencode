# broker.go (in internal/pubsub)

## Overview

The `broker.go` file, part of the `internal/pubsub` package, defines a generic, in-memory publish-subscribe (pub/sub) broker. This `Broker[T]` allows different parts of the application to communicate asynchronously by publishing events of a generic type `T` and subscribing to receive these events. It's a fundamental component for decoupling services and enabling event-driven architecture within the application.

## Key Components

### Constants
- `bufferSize (64)`: Default buffer size for subscriber channels.

### Structs
- `Broker[T any]`: The generic pub/sub broker.
    - `subs (map[chan Event[T]]struct{})`: A map where keys are subscriber channels and values are empty structs (used as a set).
    - `mu (sync.RWMutex)`: A read-write mutex to protect concurrent access to the `subs` map and `subCount`.
    - `done (chan struct{})`: A channel that is closed when the broker is shut down, signaling subscribers and publishers to stop.
    - `subCount (int)`: The current number of active subscribers.
    - `maxEvents (int)`: (Initialized in `NewBrokerWithOptions` but not directly used in the provided code, perhaps intended for future use like limiting stored events or a backlog).

### Functions
- `NewBroker[T any]() *Broker[T]`:
    - Constructor for `Broker`. Calls `NewBrokerWithOptions` with default `bufferSize` (64) and `maxEvents` (1000).
- `NewBrokerWithOptions[T any](channelBufferSize, maxEvents int) *Broker[T]`:
    - Constructor that allows specifying `channelBufferSize` and `maxEvents`.
    - Initializes the `subs` map and `done` channel.
- `(b *Broker[T]) Shutdown()`:
    - Gracefully shuts down the broker.
    - Closes the `b.done` channel to signal termination.
    - Acquires a write lock, iterates through all subscriber channels in `b.subs`, deletes them from the map, and closes each channel. This ensures subscribers stop receiving events and their listening goroutines can terminate.
    - Resets `b.subCount` to 0.
- `(b *Broker[T]) Subscribe(ctx context.Context) <-chan Event[T]`:
    - Allows a component to subscribe to events of type `T`.
    - Takes a `context.Context` which, when cancelled, will automatically unsubscribe the listener.
    - If the broker is already shut down (`b.done` is closed), it returns an immediately closed channel.
    - Otherwise, it creates a new buffered channel (`sub`) of type `Event[T]`.
    - Adds `sub` to the `b.subs` map and increments `b.subCount`.
    - Launches a goroutine that listens for `ctx.Done()`. When the context is cancelled:
        - It removes `sub` from `b.subs`, closes `sub`, and decrements `b.subCount`.
    - Returns the read-only subscriber channel `<-chan Event[T]`.
- `(b *Broker[T]) GetSubscriberCount() int`:
    - Returns the current number of active subscribers.
- `(b *Broker[T]) Publish(t EventType, payload T)`:
    - Publishes an event to all current subscribers.
    - Takes an `EventType` (e.g., `CreatedEvent`, `UpdatedEvent` from `events.go`) and a `payload` of the generic type `T`.
    - If the broker is shut down, it returns immediately.
    - It creates a snapshot of the current subscriber channels (to avoid holding the lock while sending).
    - Creates an `Event[T]` struct with the type and payload.
    - Iterates through the snapshot of subscriber channels and attempts to send the event to each.
    - Uses a non-blocking send (`select { case sub <- event: default: }`) to prevent a slow or blocked subscriber from holding up the publisher or other subscribers. If a subscriber's channel buffer is full, the event for that subscriber might be dropped.

## Important Variables/Constants
- `bufferSize`: Default capacity for subscriber channels, helping to absorb short bursts of events if a subscriber is temporarily slow.

## Usage Examples

Creating a broker for `string` events:
```go
// import "github.com/opencode-ai/opencode/internal/pubsub"
// import "context"
// import "fmt"
// import "time"

// stringBroker := pubsub.NewBroker[string]()
// defer stringBroker.Shutdown()
```

Subscribing to events:
```go
// ctx, cancel := context.WithCancel(context.Background())
// defer cancel() // Important to cancel context when subscriber is done

// eventChan := stringBroker.Subscribe(ctx)
// go func() {
//     for event := range eventChan {
//         fmt.Printf("Received event: Type=%s, Payload=%s\n", event.Type, event.Payload)
//     }
//     fmt.Println("Subscriber done.")
// }()
```

Publishing an event:
```go
// stringBroker.Publish(pubsub.CreatedEvent, "Hello, world!")
// stringBroker.Publish(pubsub.UpdatedEvent, "Updated message!")
```

This broker is used by various services in OpenCode (e.g., `session.Service`, `message.Service`, `logging.LogData`, `permission.Service`) to announce changes to their respective data types.

## Dependencies and Interactions

- **Internal Dependencies:**
    - Uses `EventType` and `Event[T]` from `events.go` in the same `pubsub` package.
- **External Libraries:**
    - `sync`: For `sync.RWMutex`.
    - `context`: For managing subscriber lifecycles.
- **Interactions:**
    - Provides a generic, type-safe pub/sub mechanism.
    - `Subscribe` returns a channel that respects context cancellation for automatic cleanup of subscriptions.
    - `Publish` uses non-blocking sends to subscriber channels, meaning that if a subscriber is not actively processing events and its buffer fills up, it might miss events. This is a common trade-off for decoupling and preventing publishers from being blocked by slow consumers.
    - The `Shutdown` method ensures that all subscriber channels are closed, allowing subscriber goroutines to terminate cleanly.

# events.go (in internal/pubsub)

## Overview

The `events.go` file, part of the `internal/pubsub` package, defines the core types and constants used by the publish-subscribe system. This includes the `EventType` for categorizing events (Created, Updated, Deleted), the generic `Event[T]` struct that wraps a payload of type `T` with an `EventType`, and interfaces for `Suscriber[T]` and `Publisher[T]`.

## Key Components

### Constants
- `CreatedEvent (EventType)`: Represents an event where a resource or data item was created. Value: "created".
- `UpdatedEvent (EventType)`: Represents an event where a resource or data item was updated. Value: "updated".
- `DeletedEvent (EventType)`: Represents an event where a resource or data item was deleted. Value: "deleted".

### Interfaces
- `Suscriber[T any]`: A generic interface for types that allow subscription to events of type `T`.
    - `Subscribe(context.Context) <-chan Event[T]`: Method to subscribe, returning a read-only channel of `Event[T]`. The `context.Context` is typically used to manage the lifecycle of the subscription.
- `Publisher[T any]`: A generic interface for types that can publish events of type `T`.
    - `Publish(EventType, T)`: Method to publish an event with a given `EventType` and payload `T`.

### Types
- `EventType (string)`: A type alias for string, used to define the named event type constants (CreatedEvent, UpdatedEvent, DeletedEvent).
- `Event[T any]`: A generic struct representing an event within the pub/sub system.
    - `Type (EventType)`: The type of the event (e.g., `CreatedEvent`).
    - `Payload (T)`: The actual data associated with the event, of generic type `T`.

## Important Variables/Constants
- `CreatedEvent`, `UpdatedEvent`, `DeletedEvent`: Standardized event types used throughout the application where pub/sub is employed.

## Usage Examples

These types and constants are fundamental to the operation of the `Broker[T]` defined in `broker.go` and are used by services that publish or subscribe to events.

Defining a service that publishes events:
```go
// import "github.com/opencode-ai/opencode/internal/pubsub"

// type MyData struct { /* ... */ }
// type MyService struct {
//     broker pubsub.Publisher[MyData] // Could also be *pubsub.Broker[MyData]
// }

// func (s *MyService) CreateData(data MyData) {
//     // ... save data ...
//     s.broker.Publish(pubsub.CreatedEvent, data)
// }
```

A component subscribing to these events:
```go
// import "github.com/opencode-ai/opencode/internal/pubsub"
// import "context"

// func WatchMyData(ctx context.Context, subscriberService pubsub.Suscriber[MyData]) {
//     eventChan := subscriberService.Subscribe(ctx)
//     for event := range eventChan {
//         switch event.Type {
//         case pubsub.CreatedEvent:
//             fmt.Printf("Data created: %+v\n", event.Payload)
//         case pubsub.UpdatedEvent:
//             fmt.Printf("Data updated: %+v\n", event.Payload)
//         // Handle other event types
//         }
//     }
// }
```

The `pubsub.Broker[T]` itself implements both `Suscriber[T]` (misspelled as "Suscriber" in the code) and `Publisher[T]`.

## Dependencies and Interactions

- **Internal Dependencies:** None beyond types within the same package (e.g., `Broker` uses `Event` and `EventType`).
- **External Libraries:**
    - `context`: Used in the `Suscriber` interface.
- **Interactions:**
    - This file provides the basic vocabulary (event types) and data structures (`Event[T]`) for the pub/sub system.
    - The `Suscriber` and `Publisher` interfaces define the contracts for interacting with a pub/sub mechanism like the `Broker`.
    - The generic nature of `Event[T]`, `Suscriber[T]`, and `Publisher[T]` allows the pub/sub system to be used for various data types throughout the application without code duplication.
    - The misspelling "Suscriber" for the interface name is a minor typo.

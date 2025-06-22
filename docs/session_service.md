# session.go (in internal/session)

## Overview

The `session.go` file, part of the `internal/session` package, defines the `Session` data structure and the `Service` for managing user conversation sessions. A session typically represents a single, coherent interaction or task context with the AI. The service handles creating, retrieving, updating (saving), and deleting sessions, interacting with the database via `sqlc`-generated queries. It also integrates with a pub/sub mechanism to broadcast session-related events.

## Key Components

### Structs
- `Session`: Represents a user conversation session.
    - `ID (string)`: Unique identifier for the session.
    - `ParentSessionID (string)`: Identifier for a parent session, if this session is a sub-task or a derivative (e.g., for title generation or agent-delegated tasks).
    - `Title (string)`: A user-friendly title for the session.
    - `MessageCount (int64)`: The number of messages associated with this session.
    - `PromptTokens (int64)`: Total prompt tokens consumed in this session.
    - `CompletionTokens (int64)`: Total completion tokens generated in this session.
    - `SummaryMessageID (string)`: ID of a message that might act as a summary for this session.
    - `Cost (float64)`: Estimated monetary cost of the LLM interactions in this session.
    - `CreatedAt (int64)`: Unix timestamp of session creation.
    - `UpdatedAt (int64)`: Unix timestamp of the last update to the session.
- `service`: The concrete implementation of the `Service` interface.
    - `Broker (*pubsub.Broker[Session])`: Pub/sub broker for session events.
    - `q (db.Querier)`: The `sqlc`-generated database querier.

### Interfaces
- `Service`: Defines the contract for session management operations.
    - `pubsub.Suscriber[Session]`: Embeds pub/sub subscription capability for session events.
    - `Create(ctx context.Context, title string) (Session, error)`: Creates a new top-level session with a given title.
    - `CreateTitleSession(ctx context.Context, parentSessionID string) (Session, error)`: Creates a special session (ID prefixed with "title-") for generating a title, linked to a `parentSessionID`.
    - `CreateTaskSession(ctx context.Context, toolCallID, parentSessionID, title string) (Session, error)`: Creates a session for a sub-task delegated by a tool, using the `toolCallID` as its own ID and linking to a `parentSessionID`.
    - `Get(ctx context.Context, id string) (Session, error)`: Retrieves a session by its ID.
    - `List(ctx context.Context) ([]Session, error)`: Lists all sessions.
    - `Save(ctx context.Context, session Session) (Session, error)`: Updates an existing session's mutable fields (Title, token counts, summary ID, cost).
    - `Delete(ctx context.Context, id string) error`: Deletes a session by its ID.

### Functions
- `NewService(q db.Querier) Service`: Constructor for the `service`. Initializes the pub/sub broker and stores the database querier.
- `(s *service) Create(...)`: Implements `Service.Create`. Calls `s.q.CreateSession` and publishes a `pubsub.CreatedEvent`.
- `(s *service) CreateTaskSession(...)`: Implements `Service.CreateTaskSession`. Calls `s.q.CreateSession` with specific parameters for task sessions and publishes a `pubsub.CreatedEvent`.
- `(s *service) CreateTitleSession(...)`: Implements `Service.CreateTitleSession`. Calls `s.q.CreateSession` with specific parameters for title generation sessions and publishes a `pubsub.CreatedEvent`.
- `(s *service) Delete(...)`: Implements `Service.Delete`. Retrieves the session first (for the event payload), then calls `s.q.DeleteSession`, and publishes a `pubsub.DeletedEvent`.
- `(s *service) Get(...)`: Implements `Service.Get`. Calls `s.q.GetSessionByID` and maps the result.
- `(s *service) Save(...)`: Implements `Service.Save`. Calls `s.q.UpdateSession` and publishes a `pubsub.UpdatedEvent`.
- `(s *service) List(...)`: Implements `Service.List`. Calls `s.q.ListSessions` and maps the results.
- `(s *service) fromDBItem(item db.Session) Session`: A helper function to convert a `db.Session` (sqlc generated type) to a `session.Session` (service's domain type). It handles nullable fields like `ParentSessionID` and `SummaryMessageID`.

## Important Variables/Constants
This file does not define exported package-level constants or variables beyond the type and interface definitions.

## Usage Examples

Creating a new session service:
```go
// Assume 'queries' is an initialized *db.Queries
// sessionService := session.NewService(queries)
```

Creating a new user session:
```go
// newSession, err := sessionService.Create(context.Background(), "My New Chat Topic")
// if err != nil { /* handle error */ }
```

Retrieving and updating a session:
```go
// currentSession, err := sessionService.Get(context.Background(), "some-session-id")
// if err != nil { /* handle error */ }
//
// currentSession.Title = "Updated Chat Topic"
// currentSession.PromptTokens += 100
// currentSession.CompletionTokens += 50
// currentSession.Cost += 0.002
//
// updatedSession, err := sessionService.Save(context.Background(), currentSession)
// if err != nil { /* handle error */ }
```

Subscribing to session events (e.g., in the TUI):
```go
// eventChan := sessionService.Subscribe(context.Background())
// go func() {
//     for event := range eventChan {
//         fmt.Printf("Session event: Type=%s, SessionID=%s, Title=%s\n",
//             event.Type, event.Payload.ID, event.Payload.Title)
//     }
// }()
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/db`: For `db.Querier` and `db.Session` (sqlc generated types).
    - `github.com/opencode-ai/opencode/internal/pubsub`: For `pubsub.Broker` and event types.
- **External Libraries:**
    - `github.com/google/uuid`: For generating new session IDs in `Create()`.
    - `database/sql`: For `sql.NullString` to handle nullable foreign keys.
    - `context`: Standard Go library.
- **Interactions:**
    - This service is central to managing the lifecycle and state of user conversations.
    - It provides an abstraction layer over the database for all session-related operations.
    - The `CreateTaskSession` and `CreateTitleSession` methods suggest specialized session types used internally by the agent system (e.g., for agent-delegated tasks or automatic title generation).
    - Integration with the pub/sub system allows other components (like the TUI) to react dynamically to changes in sessions (creation, updates, deletion).

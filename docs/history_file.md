# file.go (in internal/history)

## Overview

The `file.go` file in the `internal/history` package defines a `Service` for managing records of files, which can be thought of as file snapshots or versions, associated with user sessions. This service allows creating, retrieving, updating, and deleting these file records. It interacts with the database via `sqlc`-generated queries and incorporates a publish-subscribe mechanism (`pubsub.Broker`) to announce changes to file records. A key feature is its versioning logic for new file records.

## Key Components

### Constants
- `InitialVersion ("initial")`: A string constant representing the version name for the first snapshot of a file.

### Structs
- `File`: Represents a file record managed by the service.
    - `ID (string)`: Unique identifier for the file record.
    - `SessionID (string)`: Identifier of the session this file record belongs to.
    - `Path (string)`: The original path of the file.
    - `Content (string)`: The content of the file at the time of this record.
    - `Version (string)`: A version string for this snapshot (e.g., "initial", "v1", "v2").
    - `CreatedAt (int64)`: Timestamp of creation.
    - `UpdatedAt (int64)`: Timestamp of last update.
- `service`: The concrete implementation of the `Service` interface.
    - `Broker (*pubsub.Broker[File])`: A pub/sub broker to publish events about `File` creations, updates, and deletions.
    - `db (*sql.DB)`: The raw database connection, used for beginning transactions.
    - `q (*db.Queries)`: The `sqlc`-generated queries object for database interactions.

### Interfaces
- `Service`: Defines the contract for file history operations.
    - `pubsub.Suscriber[File]`: Embeds the subscriber interface, allowing other components to listen for `File` events.
    - `Create(ctx context.Context, sessionID, path, content string) (File, error)`: Creates the first version (`InitialVersion`) of a file record.
    - `CreateVersion(ctx context.Context, sessionID, path, content string) (File, error)`: Creates a new version of an existing file record. It determines the next version string (e.g., "v1", "v2") based on the latest existing version for that path.
    - `Get(ctx context.Context, id string) (File, error)`: Retrieves a file record by its ID.
    - `GetByPathAndSession(ctx context.Context, path, sessionID string) (File, error)`: Retrieves a file record by its path and session ID.
    - `ListBySession(ctx context.Context, sessionID string) ([]File, error)`: Lists all file records for a given session.
    - `ListLatestSessionFiles(ctx context.Context, sessionID string) ([]File, error)`: Lists the latest version of each file associated with a given session.
    - `Update(ctx context.Context, file File) (File, error)`: Updates an existing file record (content and version).
    - `Delete(ctx context.Context, id string) error`: Deletes a file record by its ID.
    - `DeleteSessionFiles(ctx context.Context, sessionID string) error`: Deletes all file records associated with a specific session.

### Functions
- `NewService(q *db.Queries, db *sql.DB) Service`: Constructor for the `service`. Initializes the pub/sub broker and stores the database querier and connection.
- `(s *service) Create(...)`: Implements `Service.Create`. Calls `createWithVersion` with `InitialVersion`.
- `(s *service) CreateVersion(...)`: Implements `Service.CreateVersion`. Fetches existing versions for the path to determine the next sequential version number (e.g., "v1" -> "v2"). If parsing fails or the format is unexpected, it falls back to a timestamp-based version.
- `(s *service) createWithVersion(ctx context.Context, sessionID, path, content, version string) (File, error)`: Internal helper for creating file records.
    - It uses a retry loop (up to `maxRetries`) to handle potential `UNIQUE constraint failed` errors when inserting into the database, which might occur if two processes try to create the same version simultaneously or if the version generation logic leads to a collision. If a collision occurs, it attempts to increment the version number (if 'vX' format) or uses a timestamp for the next attempt.
    - Operations are performed within a database transaction.
    - Publishes a `pubsub.CreatedEvent` on success.
- `(s *service) Get(...)`, `GetByPathAndSession(...)`, `ListBySession(...)`, `ListLatestSessionFiles(...)`: Implementations that call corresponding methods on `s.q` (the `sqlc` querier) and then map the `db.File` results to `history.File` using `fromDBItem`.
- `(s *service) Update(...)`: Implements `Service.Update`. Calls `s.q.UpdateFile` and publishes an `pubsub.UpdatedEvent`.
- `(s *service) Delete(...)`: Implements `Service.Delete`. Retrieves the file first (to have its data for the event), then calls `s.q.DeleteFile`, and publishes a `pubsub.DeletedEvent`.
- `(s *service) DeleteSessionFiles(...)`: Implements `Service.DeleteSessionFiles`. Lists all files for the session and calls `Delete` for each.
- `(s *service) fromDBItem(item db.File) File`: A helper function to convert a `db.File` (sqlc generated type) to a `history.File` (service's domain type).

## Important Variables/Constants
- `InitialVersion`: The default version string for the first time a file is recorded.

## Usage Examples

Creating a new file history service:
```go
// Assume 'sqlDB' is an *sql.DB and 'queries' is a *db.Queries
historyService := history.NewService(queries, sqlDB)
```

Creating an initial file record:
```go
fileRecord, err := historyService.Create(context.Background(), "session123", "main.go", "package main\nfunc main() {}")
// fileRecord.Version will be "initial"
```

Creating a new version of a file:
```go
updatedContent := "package main\nfunc main() {\n\tprintln(\"hello\")\n}"
newVersionRecord, err := historyService.CreateVersion(context.Background(), "session123", "main.go", updatedContent)
// newVersionRecord.Version might be "v1" (if previous was "initial") or "v2", etc.
```

Subscribing to file changes:
```go
eventChan := historyService.Subscribe(context.Background())
go func() {
    for event := range eventChan {
        fmt.Printf("File event: Type=%s, FileID=%s\n", event.Type, event.Data.ID)
    }
}()
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/db`: For the `db.Queries` interface and `db.File` struct (sqlc generated types).
    - `github.com/opencode-ai/opencode/internal/pubsub`: For the `pubsub.Broker` and event types.
- **External Libraries:**
    - `github.com/google/uuid`: For generating unique IDs for file records.
    - `database/sql`: For `*sql.DB` and transaction handling.
- **Interactions:**
    - This service is central to tracking changes or snapshots of files within different sessions.
    - It performs CRUD operations on the `files` table in the database.
    - Its versioning logic for `CreateVersion` tries to maintain a simple "vX" sequence.
    - It uses database transactions in `createWithVersion` to ensure atomicity and handles potential unique constraint violations with retries.
    - Other parts of the application can subscribe to its `pubsub.Broker` to react to changes in file history.

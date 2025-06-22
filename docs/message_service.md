# message.go (in internal/message)

## Overview

The `message.go` file, part of the `internal/message` package, defines the `Service` for managing conversational messages. This service handles CRUD (Create, Read, Update, Delete) operations for messages, interacting with the database via `sqlc`-generated queries. A key feature is its handling of the `Message.Parts` field, which is a slice of `ContentPart` interfaces. Because `ContentPart` can be one of several concrete types (text, image, tool call, etc.), this file implements custom JSON marshalling and unmarshalling logic (`marshallParts`, `unmarshallParts`) to correctly serialize and deserialize this polymorphic slice for database storage. The service also integrates with a pub/sub mechanism to broadcast message events.

## Key Components

### Structs
- `CreateMessageParams`: Parameters for creating a new message.
    - `Role (MessageRole)`: The role of the message sender.
    - `Parts ([]ContentPart)`: The content parts of the message.
    - `Model (models.ModelID)`: The ID of the LLM model (if an assistant message).
- `service`: The concrete implementation of the `Service` interface.
    - `Broker (*pubsub.Broker[Message])`: Pub/sub broker for message events.
    - `q (db.Querier)`: The `sqlc`-generated database querier.
- `partWrapper`: An unexported helper struct used for custom JSON marshalling/unmarshalling of `ContentPart`.
    - `Type (partType)`: An enum (`reasoningType`, `textType`, etc.) to identify the concrete type of `Data`.
    - `Data (ContentPart)`: The actual `ContentPart` data (during marshalling) or `json.RawMessage` (during unmarshalling before specific type assertion).
- `partType (string)`: An unexported enum-like string type for identifying concrete `ContentPart` types during JSON processing.

### Interfaces
- `Service`: Defines the contract for message management operations.
    - `pubsub.Suscriber[Message]`: Embeds pub/sub subscription capability.
    - `Create(ctx context.Context, sessionID string, params CreateMessageParams) (Message, error)`: Creates a new message.
    - `Update(ctx context.Context, message Message) error`: Updates an existing message (primarily its parts and finished_at time).
    - `Get(ctx context.Context, id string) (Message, error)`: Retrieves a message by ID.
    - `List(ctx context.Context, sessionID string) ([]Message, error)`: Lists all messages for a session.
    - `Delete(ctx context.Context, id string) error`: Deletes a message by ID.
    - `DeleteSessionMessages(ctx context.Context, sessionID string) error`: Deletes all messages for a session.

### Functions
- `NewService(q db.Querier) Service`: Constructor for the `service`.
- `(s *service) Create(...)`: Implements `Service.Create`.
    - Automatically adds a `Finish` part with reason "stop" to non-assistant messages.
    - Calls `marshallParts` to serialize `params.Parts`.
    - Calls `s.q.CreateMessage` to store the message in the database.
    - Publishes a `pubsub.CreatedEvent`.
- `(s *service) Update(...)`: Implements `Service.Update`.
    - Calls `marshallParts` to serialize `message.Parts`.
    - Updates `FinishedAt` in the database if a `Finish` part is present.
    - Calls `s.q.UpdateMessage`.
    - Publishes a `pubsub.UpdatedEvent`.
- `(s *service) Get(...)`, `(s *service) List(...)`, `(s *service) Delete(...)`, `(s *service) DeleteSessionMessages(...)`: Implementations that call corresponding methods on `s.q` and handle conversions/events.
- `(s *service) fromDBItem(item db.Message) (Message, error)`: Converts a `db.Message` (from sqlc) to a `message.Message`, calling `unmarshallParts`.
- `marshallParts(parts []ContentPart) ([]byte, error)`:
    - Custom marshaller for `[]ContentPart`.
    - Wraps each `ContentPart` in a `partWrapper` which includes a `Type` field indicating the concrete type.
    - Marshals the slice of `partWrapper`s to a JSON byte array. This allows storing structured, polymorphic content in a single JSON database column.
- `unmarshallParts(data []byte) ([]ContentPart, error)`:
    - Custom unmarshaller for `[]ContentPart`.
    - Unmarshals the JSON `data` (from the database) into a slice of `json.RawMessage`.
    - For each `rawPart`, it first unmarshals it into a temporary struct to read the `Type` field.
    - Based on the `Type`, it then unmarshals the `Data` field into the correct concrete `ContentPart` struct (e.g., `TextContent`, `ToolCall`).
    - Returns the reconstructed `[]ContentPart`.

## Important Variables/Constants
- `partType` enum constants (e.g., `textType`, `toolCallType`) are crucial for the custom JSON marshalling/unmarshalling logic.

## Usage Examples

This service is primarily used by the AI agent (`internal/llm/agent/agent.go`) and potentially other components that need to interact with the conversation history.

Creating a message:
```go
// Assume 'messageService' is an initialized message.Service
// import "github.com/opencode-ai/opencode/internal/message"

params := message.CreateMessageParams{
    Role: message.User,
    Parts: []message.ContentPart{
        message.TextContent{Text: "Hello AI!"},
    },
}
userMessage, err := messageService.Create(context.Background(), "session-123", params)
// userMessage will have an auto-added FinishPart
```

Updating an assistant's streaming message:
```go
// assistantMessage is an existing message.Message with Role message.Assistant
// assistantMessage.AppendContent("More text...")
// assistantMessage.AddFinish(message.FinishReasonEndTurn)
// err := messageService.Update(context.Background(), assistantMessage)
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/db`: For `db.Querier` and `db.Message` (sqlc types).
    - `github.com/opencode-ai/opencode/internal/llm/models`: For `models.ModelID`.
    - `github.com/opencode-ai/opencode/internal/pubsub`: For `pubsub.Broker` and event types.
    - Uses types defined in `content.go` from the same package (e.g., `Message`, `ContentPart`, `MessageRole`).
- **External Libraries:**
    - `github.com/google/uuid`: For generating new message IDs.
    - `database/sql`: For `sql.NullString`.
    - `encoding/json`, `fmt`, `time`, `context`: Standard Go libraries.
- **Interactions:**
    - This service acts as an abstraction layer over the database for message storage and retrieval.
    - The custom JSON marshalling/unmarshalling (`marshallParts`, `unmarshallParts`) is critical for handling the flexible `Message.Parts` slice, which can contain different types of content for multimodal messages and tool interactions. This allows complex message structures to be stored as a single JSON string in the database's "parts" column.
    - It integrates with a pub/sub system to notify other parts of the application (like the TUI) about message creation, updates, and deletions.

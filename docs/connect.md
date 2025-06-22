# connect.go

## Overview

The `connect.go` file in the `db` package is responsible for establishing and configuring the connection to the SQLite database used by the OpenCode application. It handles database file creation, directory setup, applying necessary PRAGMA settings for performance and integrity, and running database migrations using the `goose` library.

## Key Components

### Functions
- `Connect() (*sql.DB, error)`: This is the primary function of the file. It performs the following steps:
    1.  Retrieves the data directory path from the application configuration (`config.Get().Data.Directory`).
    2.  Ensures the data directory exists, creating it if necessary.
    3.  Constructs the full path to the SQLite database file (`opencode.db`) within the data directory.
    4.  Opens a connection to the SQLite database using `sql.Open("sqlite3", dbPath)`. It utilizes the `ncruces/go-sqlite3` driver.
    5.  Verifies the database connection using `db.Ping()`.
    6.  Executes a series of PRAGMA statements to configure the SQLite connection. These include enabling foreign keys, setting journal mode to WAL (Write-Ahead Logging), adjusting page size and cache size, and setting synchronous mode to NORMAL for a balance of safety and performance.
    7.  Sets up the `goose` migration tool:
        - Sets the base filesystem for migrations to `FS` (an embedded filesystem defined in `embed.go` in the same package, containing SQL migration files).
        - Sets the SQL dialect to "sqlite3".
    8.  Applies all pending database migrations by calling `goose.Up(db, "migrations")`.
    9.  Returns the `*sql.DB` connection object or an error if any step fails.

## Important Variables/Constants

This file does not define its own exported package-level constants or variables. It relies on constants and variables from other packages (like `config`) and the `FS` variable from `embed.go` for migrations.

## Usage Examples

The `Connect` function is typically called once during application startup to obtain a database connection pool, which is then used by other parts of the application for database operations.

```go
// Conceptual usage (e.g., in cmd/root.go or app.New)
import "github.com/opencode-ai/opencode/internal/db"
// ...

databaseConnection, err := db.Connect()
if err != nil {
    log.Fatalf("Failed to connect to database: %v", err)
}
// defer databaseConnection.Close() // Important to close when app exits

// Pass databaseConnection to services that need it,
// e.g., for creating a db.New(databaseConnection) querier.
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/config`: To get the data directory path (`config.Get().Data.Directory`).
    - `github.com/opencode-ai/opencode/internal/logging`: For logging errors and debug messages related to PRAGMA settings and migrations.
    - Relies on `FS` (from `embed.go` in the same `db` package) for accessing embedded SQL migration files.
- **External Libraries:**
    - `database/sql`: Standard Go library for database interaction.
    - `github.com/ncruces/go-sqlite3/driver`: SQLite driver. The blank import `_ "github.com/ncruces/go-sqlite3/driver"` registers the driver.
    - `github.com/ncruces/go-sqlite3/embed`: Embeds the SQLite library. The blank import `_ "github.com/ncruces/go-sqlite3/embed"` is for this purpose.
    - `github.com/pressly/goose/v3`: Database migration tool. Used to apply schema changes to the database.
- **Interactions:**
    - Interacts with the file system to create the data directory and the `opencode.db` SQLite file if they don't exist.
    - Modifies the state of the SQLite database by applying PRAGMA settings and running schema migrations.
    - Provides a configured `*sql.DB` object that other parts of the application use to query and modify the database.

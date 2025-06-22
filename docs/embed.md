# embed.go

## Overview

The `embed.go` file in the `db` package serves a singular, critical purpose: to embed SQL migration files into the compiled Go binary. This is achieved using Go's `embed` package. By embedding the migration files, the application can manage database schema changes without needing loose SQL files present in the file system at runtime, making deployment simpler and more self-contained.

## Key Components

### Variables
- `FS (embed.FS)`: This is a package-level variable of type `embed.FS`. The `//go:embed migrations/*.sql` directive above its declaration instructs the Go compiler to embed all files matching the pattern `migrations/*.sql` (i.e., all SQL files within the `migrations` subdirectory relative to this source file) into this `FS` variable.

## Important Variables/Constants
- `FS`: This is the most important component. It holds the content of all SQL migration files, making them accessible at runtime for database migration tools like `goose`.

## Usage Examples

The `FS` variable is primarily used by the database migration logic, typically within the `db.Connect()` function (or a similar setup function) when configuring the `goose` migration library.

```go
// Conceptual usage within db/connect.go or similar:
// import "github.com/pressly/goose/v3"
// import "github.com/opencode-ai/opencode/internal/db" // To access db.FS

// ... inside a function like db.Connect() ...
// goose.SetBaseFS(db.FS) // db.FS refers to the var FS in embed.go

// if err := goose.SetDialect("sqlite3"); err != nil {
//     // handle error
// }

// if err := goose.Up(sqlDB, "migrations"); err != nil { // "migrations" is the path within the embed.FS
//     // handle error
// }
// ...
```
The `goose` library will then read the migration files directly from this embedded filesystem.

## Dependencies and Interactions

- **Internal Dependencies:**
    - None directly, but it provides the `FS` variable which is crucial for `db/connect.go`'s migration logic.
- **External Libraries:**
    - `embed`: This is a standard Go package used for embedding files.
- **Interactions:**
    - This file directly interacts with the Go build process due to the `//go:embed` directive. The compiler reads the specified files and includes their content in the executable.
    - The `FS` variable it exports is consumed by the database migration system (e.g., `goose` as configured in `db/connect.go`) to apply schema changes to the database.
    - It depends on the presence and structure of the `migrations/` directory (relative to `embed.go`) at compile time.

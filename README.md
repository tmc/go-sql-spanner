# go-sql-spanner

[![go.dev reference](https://img.shields.io/badge/go.dev-reference-007d9c?logo=go&logoColor=white&style=flat-square)](https://pkg.go.dev/github.com/googleapis/go-sql-spanner)

[Google Cloud Spanner](https://cloud.google.com/spanner) driver for
Go's [database/sql](https://golang.org/pkg/database/sql/) package.


This support is currently in the __Preview__ release status.

``` go
import _ "github.com/googleapis/go-sql-spanner"

db, err := sql.Open("spanner", "projects/PROJECT/instances/INSTANCE/databases/DATABASE")
if err != nil {
    log.Fatal(err)
}

// Print tweets with more than 500 likes.
rows, err := db.QueryContext(ctx, "SELECT id, text FROM tweets WHERE likes > @likes", 500)
if err != nil {
    log.Fatal(err)
}
defer rows.Close()

var (
    id   int64
    text string
)
for rows.Next() {
    if err := rows.Scan(&id, &text); err != nil {
        log.Fatal(err)
    }
    fmt.Println(id, text)
}
```

## Statements

Statements support follows the official [Google Cloud Spanner Go](https://pkg.go.dev/cloud.google.com/go/spanner) client style arguments.

```go
db.QueryContext(ctx, "SELECT id, text FROM tweets WHERE likes > @likes", 500)

db.ExecContext(ctx, "INSERT INTO tweets (id, text, rts) VALUES (@id, @text, @rts)", id, text, 10000)

db.ExecContext(ctx, "DELETE FROM tweets WHERE id = @id", 14544498215374)
```

## Transactions

- Read-write transactions always uses the strongest isolation level and ignore the user-specified level.
- Read-only transactions do strong-reads by default. Read-only transactions must be ended by calling
either Commit or Rollback. Calling either of these methods will end the current read-only
transaction and return the session that is used to the session pool.

``` go
tx, err := db.BeginTx(ctx, &sql.TxOptions{}) // Read-write transaction.

tx, err := db.BeginTx(ctx, &sql.TxOptions{
    ReadOnly: true, // Read-only transaction using strong reads.
})

conn, _ := db.Conn(ctx)
_, _ := conn.ExecContext(ctx, "SET READ_ONLY_STALENESS='EXACT_STALENESS 10s'")
tx, err := conn.BeginTx(ctx, &sql.TxOptions{
    ReadOnly: true, // Read-only transaction using a 10 second exact staleness.
})
```

## DDL Statements

[DDL statements](https://cloud.google.com/spanner/docs/data-definition-language)
are not supported in transactions per Cloud Spanner restriction.
Instead, run them on a connection without an active transaction:

```go
db.ExecContext(ctx, "CREATE TABLE ...")
```

Multiple DDL statements can be sent in one batch to Cloud Spanner by defining a DDL batch:

```go
conn, _ := db.Conn(ctx)
_, _ := conn.ExecContext(ctx, "START BATCH DDL")
_, _ = conn.ExecContext(ctx, "CREATE TABLE Singers (SingerId INT64, Name STRING(MAX)) PRIMARY KEY (SingerId)")
_, _ = conn.ExecContext(ctx, "CREATE INDEX Idx_Singers_Name ON Singers (Name)")
// Executing `RUN BATCH` will run the previous DDL statements as one batch.
_, _ := conn.ExecContext(ctx, "RUN BATCH")
```

See also [the batch DDL example](/examples/ddl-batches).

## Examples

The [`examples`](/examples) directory contains standalone code samples that show how to use common
features of Cloud Spanner and/or the database/sql package. Each standalone code sample can be
executed without any prior setup, as long as Docker is installed on your local system.

## Raw Connection / Specific Cloud Spanner Features

Use the `Conn.Raw` method to get access to a Cloud Spanner specific connection instance. This
instance can be used to access Cloud Spanner specific features and settings, such as mutations,
read-only staleness settings and commit timestamps.

```go
conn, _ := db.Conn(ctx)
_ = conn.Raw(func(driverConn interface{}) error {
    spannerConn, ok := driverConn.(spannerdriver.SpannerConn)
    if !ok {
        return fmt.Errorf("unexpected driver connection %v, expected SpannerConn", driverConn)
    }
    // Use the `SpannerConn` interface to set specific Cloud Spanner settings or access
    // specific Cloud Spanner features.

    // Example: Set and get the current read-only staleness of the connection.
    _ = spannerConn.SetReadOnlyStaleness(spanner.ExactStaleness(10 * time.Second))
    _ = spannerConn.ReadOnlyStaleness()

    return nil
})
```

See also the [examples](/examples) directory for further code samples.

## Emulator

See the [Google Cloud Spanner Emulator](https://cloud.google.com/spanner/docs/emulator) support to learn how to start the emulator.
Once the emulator is started and the host environmental flag is set, the driver should just work.

```
$ gcloud beta emulators spanner start
$ export SPANNER_EMULATOR_HOST=localhost:9010
```

## Troubleshooting

The driver will retry any Aborted error that is returned by Cloud Spanner
during a read/write transaction. If the driver detects that the data that
was used by the transaction was changed by another transaction between the
initial attempt and the retry attempt, the Aborted error will be propagated
to the client application as an `spannerdriver.ErrAbortedDueToConcurrentModification`
error.


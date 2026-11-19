# sqlite

SQLite databases for [Quirk](https://github.com/AlexVachon/quirk): a file
(or memory), SQL with `?` / `:name` parameters, rows as Maps, transactions
with `with`.

SQLite itself (3.54.1) is built into Quirk: nothing to install, on Linux,
macOS and Windows.

```quirk
use sqlite

db := sqlite.open("app.db")                       // made if it isn't there
db.script("""
    CREATE TABLE IF NOT EXISTS todos (
        id   INTEGER PRIMARY KEY,
        text TEXT NOT NULL,
        done INTEGER NOT NULL DEFAULT 0
    )
""")

id := db.insert("INSERT INTO todos (text) VALUES (?)", ["milk"])
db.execute("UPDATE todos SET done = ? WHERE id = ?", [true, id])

for row in db.query("SELECT id, text FROM todos WHERE done = :done", {"done": false}) {
    print(row["text"])
}
left := db.value("SELECT count(*) FROM todos WHERE done = 0")
db.close()
```

## Opening

| | |
|---|---|
| `sqlite.open("app.db")` | the database in that file, made if it isn't there |
| `sqlite.open(":memory:")` | a new database in memory, gone when it's closed |
| `sqlite.open(path, read_only=true)` | no writes |
| `sqlite.open(path, must_exist=true)` | an error instead of a new file |
| `sqlite.open(path, busy_ms=5000)` | how long to wait while another connection writes (default 5 s) |

`db.close()` closes it (twice is fine), or `with sqlite.open(path) as db { … }`.

## Running SQL

| | returns |
|---|---|
| `db.query(sql, params)` | every row, a List of Maps (column name → value) |
| `db.query_one(sql, params)` | the first row, or `null` |
| `db.value(sql, params, default=null)` | the first column of the first row (`SELECT count(*) …`), or `default` |
| `db.execute(sql, params)` | how many rows an INSERT / UPDATE / DELETE changed |
| `db.insert(sql, params)` | the new row's id |
| `db.execute_many(sql, rows)` | the statement once per item of `rows`, in one transaction |
| `db.script(sql)` | several statements separated by `;`, no parameters (a schema, a migration) |
| `db.columns(sql)`, `db.tables()` | column names of a SELECT, the database's tables |
| `db.changes()`, `db.last_id()` | after the last statement |

### Parameters

`?` in the SQL with a List, in order — or `:name` with a Map:

```quirk
db.query("SELECT * FROM users WHERE age > ? AND city = ?", [30, "Montréal"])
db.query("SELECT * FROM users WHERE age > :age AND city = :city", {"age": 30, "city": "Montréal"})
```

The values are sent to SQLite as they are, never pasted into the SQL — so a
value can't change what the statement does (no SQL injection). **Always use
parameters for values that come from outside**; never build SQL with `+`.

Values: `Int`, `Double`, `String`, `Bool` (stored as `1` / `0`) and `null`.

### Rows

Each row is a Map, column name → value: `Int`, `Double`, `String`, or
`null` for NULL. SQLite has no Bool type: a stored `true` reads back as `1`.

```quirk
row := db.query_one("SELECT name, age FROM users WHERE id = ?", [id])
if row != null {
    print(row["name"], Int(row["age"]) + 1)
}
```

### Rows as structs

```quirk
struct Todo {
    id: Int
    text: String
    done: Bool
    tags: List
}

for t in db.query_as(Todo, "SELECT * FROM todos WHERE done = ?", [false]) {
    todo: Todo = t
    print(todo.text, todo.tags.length())
}
first := db.query_one_as(Todo, "SELECT * FROM todos WHERE id = ?", [id])   // or null
```

Each column goes in the field of the same name, as that field's type: an
INTEGER `0` / `1` in a `Bool`, TEXT holding JSON in a `List` or `Map` field, a
variant's name in an enum field, NULL as the field's zero. Columns with no field
are left out; fields with no column keep what the struct starts with.

## Schema changes: `migrate`

```quirk
db.migrate([
    "CREATE TABLE todos (id INTEGER PRIMARY KEY, text TEXT NOT NULL)",
    "ALTER TABLE todos ADD COLUMN done INTEGER NOT NULL DEFAULT 0",
    "CREATE INDEX todos_done ON todos (done)",
])
```

Each step is applied once, in order, in its own transaction (all of it or none
of it). SQLite remembers how many ran (its `user_version`, also
`db.schema_version()`), so on the next start only new steps run — call it every
time the app starts. **Only add steps at the end**; never change one that has
already run somewhere. A database that's had more steps than the list (an older
program on a newer database) is refused.

## Transactions

```quirk
with db.transaction() {
    db.execute("UPDATE accounts SET balance = balance - ? WHERE id = ?", [100, from])
    db.execute("UPDATE accounts SET balance = balance + ? WHERE id = ?", [100, to])
}
```

The statements inside are done all together, or not at all: committed at
the end of the block, **rolled back if anything in it raises** (and the
exception goes on). A `return` from inside commits. A transaction inside
another is a savepoint: if it fails, only its own part is undone.

Writing many rows is also much faster inside one transaction.

## Errors

Everything that goes wrong raises `sqlite.Error` (a `RuntimeError`), with
SQLite's message, its `code` and the `sql`:

```quirk
try {
    db.insert("INSERT INTO users (email) VALUES (?)", [email])
} catch (e: sqlite.Error) {
    if e.is_constraint() { print("that email is taken") } else { throw }
}
```

- `e.is_constraint()` — a UNIQUE / NOT NULL / CHECK / FOREIGN KEY constraint
- `e.is_busy()` — another connection kept the database locked past `busy_ms`
- `e.code` — SQLite's extended result code (`2067` UNIQUE, `1299` NOT NULL …;
  `code % 256` is the primary one)

## Async tasks

While SQLite reads or writes the disk, other `async` tasks run. A
connection can be shared by tasks; SQLite is built thread-safe.

## Requirements

Quirk 7.54.0 or later (SQLite is in its runtime).

## License

MIT. SQLite is in the public domain.

# PostgreSQL Client - Ecko Std Lib Package

A PostgreSQL client for [Ecko](https://ecko.sh), written in Ecko. It speaks the v3 wire protocol over `std.net`'s raw sockets and does SCRAM-SHA-256 authentication (Postgres's modern default), built from
`std.hash`, `bytes`, and the bitwise operators. The PBKDF2 step (4096 rounds
with Postgres's default settings) runs in the interpreter and takes a few
milliseconds.

## Install

```bash
ecko get github.com/ecko-lang/postgresql-client --as postgres
```

`ecko get` vendors the package under
`./vendor/github.com/ecko-lang/postgresql-client/` and pins a file-tree hash in
`ecko.sum`.

Without `--as`, `ecko get` records this dependency under the alias
`postgresql_client` (the repo name with `-` mapped to `_`); `--as postgres` is what makes
`import postgres` work. It writes the entry as
`"postgres": "github.com/ecko-lang/postgresql-client@0.38.0"`. Expand it to the map form in
your `ecko.json` to grant the network capability the client needs:

```json
{
  "dependencies": {
    "postgres": {
      "path": "github.com/ecko-lang/postgresql-client",
      "version": "0.38.0",
      "grant": ["net"]
    }
  }
}
```

```ecko
import postgres
```

## Usage

```ecko
import postgres

db = postgres.connect({
    host: "127.0.0.1", port: 5432,
    user: "me", password: "secret", database: "app"
})                                      # or postgres.connect_tls({ ... })

rows = postgres.query(db, "select id, name from users order by id")
for r in rows {                         # rows are maps keyed by column name
    print("{r.id}: {r.name}")
}

# Parameterized query: $1..$N bind out-of-band - injection-safe.
postgres.query(db, "insert into users (name) values ($1)", ["Ada"])
me = postgres.query(db, "select * from users where id = $1", [42])

# Or a sql { } block: every {hole} is a bound parameter.
name = "Ada"
postgres.query(db, sql { SELECT * FROM users WHERE name = {name} })

# Commits when the function returns; rolls back if it raises.
postgres.transaction(db, fn(tx) {
    postgres.query(tx, "update accounts set balance = balance - $1 where id = $2", [100, 1])
    postgres.query(tx, "update accounts set balance = balance + $1 where id = $2", [100, 2])
})

postgres.close(db)
```

## API

| function | notes |
|----------|-------|
| `connect({host, port, user, password, database})` | open + authenticate (SCRAM-SHA-256) |
| `connect_tls({...})` | same, over TLS |
| `query(db, sql)` | simple text query; returns a list of row maps (empty for non-SELECT) |
| `query(db, sql, params)` | **parameterized** query with `$1..$N` placeholders - injection-safe |
| `query(db, sql { ... })` | a `sql { }` block; its holes are bound parameters |
| `transaction(db, fn)` | run `fn(db)` in a transaction: commit on return, roll back and re-raise on error; returns what `fn` returned |
| `close(db)` | close the connection |

Values come back in text format: integers, text, etc. as strings; SQL
`NULL` as `null`. Column names are the map keys.

### Parameterized queries

Pass a params list and reference them as `$1`, `$2`, … in the SQL. Each value
is sent to the server **out-of-band** via the extended query protocol (Parse /
Bind / Execute), so it is never spliced into the SQL text and can't break out
of its slot - the same injection-safety guarantee as a `?` placeholder:

```ecko
name = "Robert'); DROP TABLE students;--"
postgres.query(db, "insert into students (name) values ($1)", [name])
# stored as a literal string; no tables were harmed
```

Params bind in text format (integers/floats/strings verbatim, `true`/`false`
as `t`/`f`, `bytes` as a `\x` bytea literal, `null` as SQL `NULL`); the server
coerces to each column's type. Without a params list, `query` uses the simple
protocol as before.

### `sql { }` blocks

A `sql { }` block is the same `{ query, params }` value `std.sql` takes, with a
`?` for each hole. `query` numbers those for Postgres (`$1`, `$2`, ...), skipping
any `?` inside a string, a quoted name, a comment or a dollar-quoted body. Pass
the block on its own: it carries its params, so a separate list is refused.

**A block cannot use the jsonb `?` operators** (`?`, `?|`, `?&`): a `?` there
looks like one more placeholder. The count is checked, so this fails with an
error rather than binding the wrong values. Use `jsonb_exists()` and its
siblings, or the string form with `$n`, where a `?` is never rewritten.

### Transactions

`transaction(db, fn)` has the same shape as `std.sql`'s. It commits when `fn`
returns and gives back its value. If `fn` raises, it rolls back and re-raises
the same error. If the commit fails it rolls back and raises that error.

Postgres aborts a transaction when a statement in it fails, even if `fn`
caught the error, and then answers `COMMIT` with a rollback. `transaction`
reports that as an error rather than as success, because nothing was saved.
Transactions do not nest: starting one inside another on the same connection
raises.

## Errors

Every failure is `{ kind: "sql", driver: "postgres", message }`, the kind
`std.sql` raises, so one `catch` arm covers both. An error from the server also
carries `code`, its SQLSTATE: `23505` is a unique violation, `40001` a
serialization failure.

```ecko
try {
    postgres.query(db, "insert into users (email) values ($1)", [email])
} catch (e) {
    match e.kind {
        "sql" => if e.code == "23505" { print("already registered") } else { error(e) }
        _ => error(e)
    }
}
```

A broken connection reports the same kind. A failed query leaves the connection
usable for the next one.

**Migration from 0.37:** errors used to be `kind: "postgres"`. Match `"sql"`
instead, and check `e.driver` if you need to tell databases apart.

## Testing

```bash
ecko test          # offline: SCRAM vectors, bind encoding, placeholders and
                   # transactions, partly against a scripted loopback server
ecko example.ecko  # live round trip against a PostgreSQL server
```

## License

MIT

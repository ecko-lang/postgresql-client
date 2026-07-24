# PostgreSQL Client - Ecko Std Lib Package

A PostgreSQL client for [Ecko](https://ecko.sh), written in Ecko. It speaks the v3 wire protocol over `std.net`'s raw sockets and does SCRAM-SHA-256 authentication (Postgres's modern default), built from
`std.hash`, `bytes`, and the bitwise operators. The PBKDF2 step (4096 rounds
with Postgres's default settings) runs in the interpreter and is still
sub-millisecond.

## Install

```bash
ecko get github.com/ecko-sh/postgresql-client
```

`ecko get` vendors the package under
`./vendor/github.com/ecko-sh/postgresql-client/` and pins a file-tree hash in
`ecko.sum`.

`ecko get` records this dependency under the alias `postgresql-client`,
which isn't a valid import name (hyphens aren't allowed in Ecko
identifiers). Alias it to `postgres` in your `ecko.json` - this also grants
the network capability the client needs:

```json
{
  "dependencies": {
    "postgres": {
      "path": "github.com/ecko-sh/postgresql-client",
      "version": "v0.9.1",
      "grant": ["net"]
    }
  }
}
```

```ecko
import postgres
```

## Use

```ecko
import postgres

db = postgres.connect({
    host: "127.0.0.1", port: 5432,
    user: "me", password: "secret", database: "app"
})                                      # or postgres.connect_tls({ ... })

rows = postgres.query(db, "select id, name from users order by id")
for r in rows {                         # rows are maps keyed by column name
    print(get(r, "id") + ": " + get(r, "name"))
}

# Parameterized query: $1..$N bind out-of-band - injection-safe.
postgres.query(db, "insert into users (name) values ($1)", ["Ada"])
me = postgres.query(db, "select * from users where id = $1", [42])

postgres.close(db)
```

## API

| function | notes |
|----------|-------|
| `connect({host, port, user, password, database})` | open + authenticate (SCRAM-SHA-256) |
| `connect_tls({...})` | same, over TLS |
| `query(db, sql)` | simple text query; returns a list of row maps (empty for non-SELECT) |
| `query(db, sql, params)` | **parameterized** query with `$1..$N` placeholders - injection-safe |
| `close(db)` | close the connection |
| `pbkdf2(password, salt, iters)` | the PBKDF2-HMAC-SHA256 primitive (bonus utility) |

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

## Testing

```bash
ecko test          # offline: SCRAM PBKDF2 against known vectors, no server
ecko example.ecko  # live round trip against a PostgreSQL server
```

## License

MIT

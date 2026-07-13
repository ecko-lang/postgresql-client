# PostgreSQL Client

A PostgreSQL client for [Ecko](https://ecko.sh), written in Ecko. It speaks the v3 wire protocol over `std.net`'s raw sockets and does SCRAM-SHA-256 authentication (Postgres's modern default), built from
`std.hash`, `bytes`, and the bitwise operators. The PBKDF2 step (4096 rounds
with Postgres's default settings) runs in the interpreter and is still
sub-millisecond.

## Install

```bash
ecko add https://github.com/ecko-sh/postgresql-client
```

`ecko add` vendors the package into `./vendor/postgres/` and pins it by SHA-256
in `ecko.lock`. Grant it the network capability in your `ecko.json`:

```json
{
  "dependencies": {
    "postgres": {
      "source": "https://github.com/ecko-sh/postgresql-client",
      "grant": ["net"]
    }
  }
}
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

postgres.close(db)
```

## API

| function | notes |
|----------|-------|
| `connect({host, port, user, password, database})` | open + authenticate (SCRAM-SHA-256) |
| `connect_tls({...})` | same, over TLS |
| `query(db, sql)` | run a statement; returns a list of row maps (empty for non-SELECT) |
| `close(db)` | close the connection |
| `pbkdf2(password, salt, iters)` | the PBKDF2-HMAC-SHA256 primitive (bonus utility) |

Values come back in text format: integers, text, etc. as strings; SQL
`NULL` as `null`. Column names are the map keys.

This client implements the simple query protocol only, so statements are sent
to the server as text. The extended (parameterized) protocol is a planned
addition. Until then, quote/escape untrusted input yourself or use server-side
validation.

## Testing

```bash
ecko test          # offline: SCRAM PBKDF2 against known vectors, no server
ecko example.ecko  # live round trip against a PostgreSQL server
```

## License

MIT

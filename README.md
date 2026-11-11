# postgresql-client

A PostgreSQL client written in Ecko: the v3 wire protocol and SCRAM-SHA-256
authentication over `std.net`, with parameterized queries, `sql { }` blocks,
transactions and affected-row counts. Needs `net`; grant it when you import
the package.

## Install

```bash
ecko get github.com/ecko-lang/postgresql-client --as postgres
```

Then grant the network capability in the dependency entry of your `ecko.json`:

```json
{
  "dependencies": {
    "postgres": {
      "path": "github.com/ecko-lang/postgresql-client",
      "version": "0.39.0",
      "grant": ["net"]
    }
  }
}
```

```ecko norun
import postgres   # granted net in ecko.json
```

## Usage

```ecko norun
import postgres   # granted net in ecko.json

db = postgres.connect("127.0.0.1", "app", password: secret("pw"), database: "shop")

# Parameterized: $1..$N bind out-of-band, so a value never changes the SQL.
n = postgres.exec(db, "insert into users (name) values ($1), ($2)", ["Ada", "Grace"])
print(n)                                 # 2

rows = postgres.query(db, "select id, name from users where name = $1", ["Ada"])
for r in rows {
    print("{r.id}: {r.name}")            # 1: Ada  (every value is text)
}

# A sql { } block: each {hole} is a bound parameter.
name = "Grace"
print(postgres.query(db, sql { SELECT count(*) AS n FROM users WHERE name = {name} }))
# [{n: 1}]

# Commits when the function returns; rolls back and re-raises if it raises.
postgres.transaction(db, fn(tx) {
    postgres.exec(tx, "update accounts set balance = balance - $1 where id = $2", [100, 1])
    postgres.exec(tx, "update accounts set balance = balance + $1 where id = $2", [100, 2])
})

postgres.close(db)
```

### Handling errors

```ecko norun
import postgres   # granted net in ecko.json

db = postgres.connect("127.0.0.1", "app", password: secret("pw"), database: "shop")
email = "ada@example.com"
try {
    postgres.exec(db, "insert into users (email) values ($1)", [email])
} catch (e) {
    if get(e, "kind") == "sql" and get(e, "code") == "23505" {
        print("already registered")
    } else {
        error(e)
    }
}
```

## API

| function | returns |
|---|---|
| `connect(host, user, password: null, database: null, port: 5432)` | a connection handle, authenticated |
| `connect_tls(host, user, password: null, database: null, port: 5432)` | the same, over TLS |
| `query(db, sql, params = [])` | the rows, as a list of maps keyed by column name |
| `exec(db, sql, params = [])` | the number of rows the statement changed, as an Int |
| `transaction(db, body)` | what `body(db)` returned, committed |
| `close(db)` | `null`; the connection is closed |

## Notes

### Errors

A failure caused by the server or the network is `kind: "sql"`, the kind
`std.sql` raises, with `driver: "postgres"`; one `catch` arm covers every
database. An error the server reported also carries `code`, its SQLSTATE
(`23505` is a unique violation, `40001` a serialization failure). Read fields
with `get(e, "code")`: a connection error has no `code`.

| kind | reason | when |
|---|---|---|
| `net` | | the server cannot be reached |
| `sql` | `server_error` | the server rejected the login or the statement; `code` is the SQLSTATE |
| `sql` | `auth_failed` | the server asks for a password and none was given |
| `sql` | `auth_unsupported` | the server asks for an auth method other than SCRAM-SHA-256 or trust (MD5, cleartext) |
| `sql` | `duplicate_column` | two result columns share a name; give one an alias with `AS` |
| `sql` | `rolled_back` | `transaction`'s commit became a rollback because a statement inside failed |
| `sql` | `tls_refused` | `connect_tls` asked for TLS and the server does not accept it |
| `sql` | `closed` | the server closed the connection mid-reply |
| `sql` | `protocol` | the server sent a message this client cannot parse |
| `bug` | `bad_argument` | a wrongly typed argument; a list, map or other unsupported parameter; a malformed `sql { }` call; `connect` given a map |
| `bug` | `nested_transaction` | `transaction` started on a connection already inside one |

A failed query, including a `duplicate_column` one, leaves the connection
usable for the next one. A failed login closes the socket it opened.

### Parameters

`$1..$N` parameters are sent out-of-band through the extended query protocol
(Parse, Bind, Describe, Execute, Sync, in one write), never spliced into the
SQL, so a value cannot break out of its slot. They bind in text format and the
server converts each to its column's type:

- strings, Ints, Floats and decimals as written; `true`/`false` as `t`/`f`
- `bytes` as a `\x` bytea literal
- a `secret` is revealed, so the real value is stored
- `null` as SQL NULL
- a list or map is refused: it has no single text form for an array, json or
  composite column. Encode it yourself, such as `json.encode(v)` for `jsonb`.

Without params, `query` and `exec` use the simple protocol.

### Results

Values come back in text format: numbers and booleans as strings (`"42"`,
`"t"`), SQL NULL as `null`. The client asks the server for
`client_encoding=UTF8`, so text arrives as UTF-8 whatever the database's
encoding; a value that still is not valid UTF-8 (a `SQL_ASCII` database stores
bytes unchecked) comes back as `bytes` rather than failing the query.

Rows are maps keyed by column name, so a result with two columns of one name
(`select a.id, b.id ...`) is refused with `duplicate_column` instead of one
value silently replacing the other.

`exec` reads the count from the server's command tag: `INSERT 0 3` is 3,
`UPDATE 2` is 2, and a statement whose tag has no count (`CREATE TABLE`) is 0.

### `sql { }` blocks

A `sql { }` block is the same `{ query, params }` value `std.sql` takes, with a
`?` for each hole. `query` and `exec` number those for Postgres (`$1`, `$2`,
...), skipping any `?` inside a string, a quoted name, a comment or a
dollar-quoted body. Pass the block on its own: it carries its params, so a
separate list is refused.

A block cannot use the jsonb `?` operators (`?`, `?|`, `?&`): a `?` there looks
like one more placeholder. The count is checked, so this fails rather than
binding the wrong values. Use `jsonb_exists()` and its siblings, or the string
form with `$n`, where a `?` is never rewritten.

### Transactions

`transaction(db, body)` has the same shape as `std.sql`'s. Postgres aborts a
transaction when a statement in it fails, even if `body` caught the error, and
then answers `COMMIT` with a rollback; `transaction` raises `rolled_back` then,
because nothing was saved. Transactions do not nest.

### Limits

- Authentication is SCRAM-SHA-256 or trust. MD5 and cleartext password auth
  are not supported.
- The server's SCRAM signature (its proof that it knows the password) is not
  verified. Use `connect_tls` to a server you trust.
- Values are not converted to Ecko types; `COPY` is not supported.
- There are no connect or query timeouts.

## Testing

```bash
ecko test          # offline, against known vectors and a scripted loopback server
ecko example.ecko  # a live round trip; needs a running PostgreSQL server
```

## License

[MIT](LICENSE)

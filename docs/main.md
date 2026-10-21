# postgresql-client

## `connect(cfg)`

connect({ host, port, user, password, database }) -> a connection handle.

## `connect_tls(cfg)`

Connect to PostgreSQL over TLS and complete authentication. Same `cfg` as
`connect`. Needs the `net` capability.

## `close(sock)`

Close the connection.

## `query(sock, sql, params = [])`

query(sock, sql) runs a simple text query. query(sock, sql, params) runs the
extended protocol with $1..$N placeholders bound out-of-band (injection-safe).
A `sql { }` block works in place of the string and the list: its `?`
placeholders are numbered for Postgres.

## `transaction(sock, body)`

transaction(db, fn) runs `fn(db)` inside a transaction: it commits when `fn`
returns, and rolls back and re-raises if `fn` raises. A commit that fails is
rolled back too, and so is one the server turns into a rollback because a
statement inside failed; both raise. Returns what `fn` returned.

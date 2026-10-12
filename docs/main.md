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

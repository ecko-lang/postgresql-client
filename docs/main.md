# postgresql-client

## `pbkdf2(password, salt, iters)`

PBKDF2-HMAC-SHA256, single 32-byte block (SCRAM's dkLen).

## `connect(cfg)`

connect({ host, port, user, password, database }) -> a connection handle.

## `connect_tls(cfg)`

Connect to PostgreSQL over TLS and complete authentication. Same `cfg` as
`connect`. Needs the `net` capability.

## `close(sock)`

Close the connection.

## `pg_text(v)`

Text format for every param: the value goes out-of-band as a length-prefixed
field, NEVER spliced into the SQL, so it can't break out of its slot. That's
injection safety by construction - the same guarantee as a `?` placeholder.

## `build_bind(params)`

build_bind(params) -> the Bind ('B') message payload: unnamed portal + unnamed
statement + 0 param format codes (all text) + Int16 param count + each param
(Int32 length, then bytes; length -1 = SQL NULL) + 0 result format codes.

## `query(sock, sql, params = [])`

query(sock, sql) runs a simple text query. query(sock, sql, params) runs the
extended protocol with $1..$N placeholders bound out-of-band (injection-safe).

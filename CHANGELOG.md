# Changelog

## 0.39.0

### Breaking

- `connect` and `connect_tls` take named arguments instead of a map:
  `connect(host, user, password: null, database: null, port: 5432)`. A map
  passed as the first argument is refused with a `bug` error that shows the
  new form.
  Migration: `postgres.connect({ host: h, port: p, user: u, password: pw, database: d })`
  becomes `postgres.connect(h, u, password: pw, database: d, port: p)`.
- A list or map query parameter is refused (`bug`, `bad_argument`) instead of
  being sent as its display text, such as `[1, 2]`, which stored a wrong value
  or failed on the server.
  Migration: encode the value yourself, for example `json.encode(v)` for a
  `json` or `jsonb` column.
- A result with two columns of the same name raises `sql` with reason
  `duplicate_column` instead of keeping only the last value.
  Migration: give one of the columns an alias with `AS`.
- A mistake in the calling code (a malformed `sql { }` call, a nested
  `transaction`) is now kind `bug` instead of `sql`.
  Migration: do not catch these as `sql`; fix the call.

### Added

- `exec(db, sql, params = [])` returns the number of rows a statement changed.
- Every `sql` error carries a `reason` (`server_error`, `auth_failed`,
  `auth_unsupported`, `duplicate_column`, `rolled_back`, `tls_refused`,
  `closed`, `protocol`), so a caller can match on it.
- `password` may be a `secret`.

### Fixed

- A `secret` query parameter was stored as the text `[secret]`; its value is
  now sent.
- `connect_tls` now sends Postgres's TLS request before the handshake, so it
  works with an ordinary PostgreSQL server.
- `password` and `database` are optional: trust authentication and the
  server's default database no longer fail with "map has no key".
- The client asks the server for UTF-8 text, and a value that is still not
  valid UTF-8 comes back as `bytes` instead of crashing the query.
- A failed login closes the socket it opened.
- A truncated or malformed reply from the server raises `sql` with reason
  `protocol` instead of an index error.
- An unsupported auth method (MD5, cleartext) says so and names its code.

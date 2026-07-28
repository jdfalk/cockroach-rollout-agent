### Fixed

#### SQL coordination could not connect to any secure CockroachDB cluster

`db_client` built a bare `native-tls` connector with no root certificate and no
client identity, so every SQL-coordinated command — `init-db`, `discover`,
`daemon` — failed against a normal cluster. `sslmode=require` died at
`error performing TLS handshake`, because only the system trust store was
consulted and CockroachDB deployments use a private CA. Supplying libpq's
`sslrootcert`/`sslcert`/`sslkey` in the connection string failed differently,
with `invalid connection string`, because the `postgres` crate's parser
understands `sslmode` and nothing else. There was no combination that worked,
which made the recommended SQL-coordinated deployment unreachable in practice.

TLS material is now configured outside the URL, through `--ssl-root-cert`,
`--ssl-client-cert` and `--ssl-client-key` (and the matching
`CROACH_ROLLOUT_SSL_*` environment variables), and wired into the connector via
`add_root_certificate` and `Identity`.

One sharp edge is called out rather than papered over: `native-tls` accepts only
PKCS#8 keys, while `cockroach cert create-client` emits PKCS#1. That case is
detected and the error names the `openssl pkcs8 -topk8` conversion instead of
surfacing a bare parse failure.

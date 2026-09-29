# Standalone PostgreSQL Databases

Databases belong to the signed-in user, not to a deployment. No repository,
template, app linkage, or automatic environment injection is involved.

- `create_postgres_database(name)` accepts only a display name (1-63 characters).
  The server generates a unique physical database name, username, and password,
  configures database-level access, and verifies a login before returning.
- `list_postgres_databases()` returns this user's IDs, names, status, and the
  usable connection strings for ready databases. These values are secrets.
- `delete_postgres_database(database_id)` requires explicit user intent. It
  terminates connections, drops all data in that database, and removes its login.
  Other databases and the shared PostgreSQL container are preserved.

Creation completes within the tool call; it does not return a deployment run ID.
Do not poll deployment tools for a database. After an ambiguous create result,
list databases before deciding whether another create is needed. Failed setup
retains an owned record for cleanup. Do not silently delete or recreate it.

## Choose the Connection String

| Returned field | Use |
|---|---|
| `connection_string` | Public TLS on port 443, for Node `pg` 8.23+ and current Go `pgx` supporting `sslnegotiation=direct`. |
| `libpq_connection_string` | Public TLS for Python/psql using libpq 17+; includes `sslrootcert=system` for certificate verification. Use system libpq 18.6+ for Python on this infrastructure; older bundled libpq previously stalled TLS reads. |
| `private_connection_string` | Apps inside the PaperOS subnet. Port 15432, authenticated plaintext on the trusted private network. Not reachable from a developer's laptop over the internet. |

The public router requires direct TLS, not the traditional PostgreSQL SSL
negotiation. Inspect the actual driver/version; do not assume every database
client or ORM accepts the same URL parameters. Keep certificate validation on.
Do not pass the libpq-specific `sslrootcert=system` URL to Node `pg`, which treats
that value as a filename. Upgrade an unsupported driver with user approval, or
explain its supported connection/tunnel configuration; do not invent a URL.

Use the selected returned URL in the backend's normal database env variable.
Never commit it, log it, or place it in browser-exposed env. Multiple apps may
use the same database. App destruction/redeploy does not delete a database;
database deletion does not update any app's env automatically. The dashboard
Databases page provides the same create, list/reveal/copy, and delete operations.

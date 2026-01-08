# DB-1 Validation: PostgreSQL binding on port 5001

Date: 2026-01-08

## Expected
PostgreSQL should be listening on **port 5001** (per `database/startup.sh` and `db_connection.txt`).

## Observed
- **No listener on 5001**
  - `ss -ltnp | grep :5001` → no output
  - `pg_isready -p 5001` → `/var/run/postgresql:5001 - no response`
  - `psql -p 5001 -d postgres -c "SELECT 1"` → socket not found

- **Postgres is running on port 5000 and holds the data dir lock**
  - Startup output:
    - `FATAL: lock file "postmaster.pid" already exists`
    - `HINT: ... (PID 292) ... "/var/lib/postgresql/data"?`
  - PID 292:
    - `/usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/data -p 5000`
  - `pg_isready -p 5000` → accepting connections
  - `SHOW port;` (on port 5000) → `5000`
  - `/var/lib/postgresql/data/postmaster.pid` shows port `5000`

## Notes
- `database/startup.sh` is configured with `DB_PORT=5001` but cannot start a second instance on the same data directory while the existing port 5000 instance is running.
- The script currently continues after failing to start Postgres on 5001 and writes `db_connection.txt`/`db_visualizer/postgres.env` pointing to 5001, which does not match the actual running server.

## Readiness
- Port 5001: **NOT READY**
- Port 5000: **READY** (server responding; `myapp` + `appuser` exist)

## Suggested remediation
Stop the existing Postgres instance using `/var/lib/postgresql/data` on port 5000, then start it on port 5001 (or ensure it starts on 5001 initially), and re-run readiness checks.

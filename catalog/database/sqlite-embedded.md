# SQLite Embedded Database in Go (modernc.org/sqlite, no cgo)

- **category**: database
- **provider**: SQLite via `modernc.org/sqlite` (pure-Go translation of SQLite) on Go's `database/sql`
- **reusable**: yes — the open/DSN pragmas, writer/reader pool split, embedded forward-only migrations and the foreign-key re-check are app-agnostic; only the schema is per-app.
- **docs**: https://pkg.go.dev/modernc.org/sqlite · https://www.sqlite.org/wal.html · https://www.sqlite.org/pragma.html · https://pkg.go.dev/embed

## Overview
A single-file SQLite database owned by one long-running Go process. It uses the cgo-free `modernc.org/sqlite` driver, so builds cross-compile with `CGO_ENABLED=0` to a static binary for amd64 or arm64. WAL journaling lets reads proceed during writes, a busy timeout absorbs short lock waits, and numbered SQL migrations are embedded into the binary and applied in order at startup. Reach for it for single-host apps, self-hosted dashboards and daemons where a database server adds nothing. It doesn't fit an app that must scale out to several writer processes.

## Playbook

### Prerequisites
- Go 1.21+ (`embed`, `errors.Join`). Current driver releases bundle SQLite 3.38 or later, so `unixepoch()` is available.
- `go get modernc.org/sqlite`. No C toolchain is needed.
- Optional on the host: the `sqlite3` CLI for inspection and online backups.

### Setup steps
1. Create `migrations/` next to your store package with zero-padded, ordered files: `0001_init.sql`, `0002_add_x.sql`, … Never edit or rename an applied file. Fix mistakes with a new one.
2. Open **two** `*sql.DB` handles on the same DSN: a writer capped at **1** connection, and a reader pool (e.g. 4). Every write goes through the writer, every read through the reader.
3. Put pragmas in the DSN (`_pragma=...`) so **every pooled connection** gets them, not just the first one.
4. On startup run migrations on the writer: create the version table, glob the embedded files, sort, skip the applied ones, and run each in a transaction together with its version insert.
5. After migrating, turn `foreign_keys` back on and run `PRAGMA foreign_key_check`. Refuse to start if it returns any row.
6. Scope every query by tenant (`account_id`, `user_id`, …) in the store layer. Make the scope a required function argument, never optional.
7. Keep timestamps as Unix seconds (`INTEGER`, `DEFAULT (unixepoch())`) and format them at the API edge.

### Core pattern
**Open (writer + readers)**
```go
package store

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"net/url"

	_ "modernc.org/sqlite" // registers driver name "sqlite"
)

type Store struct {
	w *sql.DB // single writer connection
	r *sql.DB // reader pool
}

func Open(ctx context.Context, path string) (*Store, error) {
	dsn := "file:" + url.PathEscape(path) +
		"?_pragma=busy_timeout(5000)" +
		"&_pragma=foreign_keys(1)" +
		"&_pragma=journal_mode(WAL)" +
		"&_pragma=synchronous(NORMAL)"

	w, err := sql.Open("sqlite", dsn)
	if err != nil {
		return nil, err
	}
	w.SetMaxOpenConns(1) // SQLite allows one writer; serialise in Go instead of hitting SQLITE_BUSY

	r, err := sql.Open("sqlite", dsn)
	if err != nil {
		w.Close()
		return nil, err
	}
	r.SetMaxOpenConns(4)

	s := &Store{w: w, r: r}
	if err := s.migrate(ctx); err != nil {
		s.Close()
		return nil, fmt.Errorf("migrate: %w", err)
	}
	return s, nil
}

func (s *Store) Close() error { return errors.Join(s.r.Close(), s.w.Close()) }

func (s *Store) withTx(ctx context.Context, fn func(*sql.Tx) error) error {
	tx, err := s.w.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	if err := fn(tx); err != nil {
		tx.Rollback()
		return err
	}
	return tx.Commit()
}
```

**Embedded, ordered migrations**
```go
import (
	"embed"
	"io/fs"
	"sort"
	"strings"
)

//go:embed migrations/*.sql
var migrationsFS embed.FS

func (s *Store) migrate(ctx context.Context) error {
	if _, err := s.w.ExecContext(ctx, `CREATE TABLE IF NOT EXISTS schema_migrations (
		version TEXT PRIMARY KEY, applied_at INTEGER NOT NULL DEFAULT (unixepoch()))`); err != nil {
		return err
	}
	names, err := fs.Glob(migrationsFS, "migrations/*.sql")
	if err != nil {
		return err
	}
	sort.Strings(names) // zero-padded names sort in apply order
	for _, name := range names {
		version := strings.TrimSuffix(strings.TrimPrefix(name, "migrations/"), ".sql")
		var n int
		if err := s.w.QueryRowContext(ctx, `SELECT count(*) FROM schema_migrations WHERE version = ?`, version).Scan(&n); err != nil {
			return err
		}
		if n > 0 {
			continue
		}
		raw, err := migrationsFS.ReadFile(name)
		if err != nil {
			return err
		}
		// PRAGMAs such as journal_mode / foreign_keys are ignored or rejected inside a
		// transaction, so run them first on the writer connection.
		pragmas, body := splitPragmas(string(raw))
		for _, p := range pragmas {
			if _, err := s.w.ExecContext(ctx, p); err != nil {
				return err
			}
		}
		if err := s.withTx(ctx, func(tx *sql.Tx) error {
			if _, err := tx.ExecContext(ctx, body); err != nil { // multi-statement exec is supported
				return err
			}
			_, err := tx.ExecContext(ctx, `INSERT INTO schema_migrations(version) VALUES (?)`, version)
			return err
		}); err != nil {
			return fmt.Errorf("%s: %w", version, err)
		}
	}
	return s.enforceForeignKeys(ctx)
}

func splitPragmas(text string) (pragmas []string, body string) {
	var b strings.Builder
	for _, line := range strings.Split(text, "\n") {
		if strings.HasPrefix(strings.ToUpper(strings.TrimSpace(line)), "PRAGMA ") {
			pragmas = append(pragmas, strings.TrimSpace(line))
			continue
		}
		b.WriteString(line + "\n")
	}
	return pragmas, b.String()
}

// A table-rebuild migration turns foreign keys off; turn them back on and refuse to
// start if the rebuilt data violates them.
func (s *Store) enforceForeignKeys(ctx context.Context) error {
	if _, err := s.w.ExecContext(ctx, `PRAGMA foreign_keys = ON`); err != nil {
		return err
	}
	rows, err := s.w.QueryContext(ctx, `PRAGMA foreign_key_check`)
	if err != nil {
		return err
	}
	defer rows.Close()
	if rows.Next() {
		var table, parent string
		var rowid sql.NullInt64
		var fkid int
		if err := rows.Scan(&table, &rowid, &parent, &fkid); err != nil {
			return err
		}
		return fmt.Errorf("foreign key violation: %s row %d references missing %s", table, rowid.Int64, parent)
	}
	return rows.Err()
}
```

**Table-rebuild migration (the SQLite way to change a column or constraint)**
```sql
-- 0007_items_add_tenant.sql
PRAGMA foreign_keys = OFF;

CREATE TABLE items_new (
  tenant_id  INTEGER NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
  id         TEXT NOT NULL,
  title      TEXT,
  created_at INTEGER NOT NULL DEFAULT (unixepoch()),
  PRIMARY KEY (tenant_id, id)
);
INSERT INTO items_new (tenant_id, id, title, created_at)
  SELECT 1, id, title, created_at FROM items;
DROP TABLE items;
ALTER TABLE items_new RENAME TO items;
CREATE INDEX idx_items_created ON items(tenant_id, created_at DESC);
```

**Scoped queries and upserts**
```go
func (s *Store) Item(ctx context.Context, tenantID int64, id string) (Item, error) {
	var it Item
	err := s.r.QueryRowContext(ctx,
		`SELECT id, title, created_at FROM items WHERE tenant_id = ? AND id = ?`, tenantID, id,
	).Scan(&it.ID, &it.Title, &it.CreatedAt)
	if errors.Is(err, sql.ErrNoRows) {
		return Item{}, ErrNotFound
	}
	return it, err
}

func (s *Store) UpsertItem(ctx context.Context, tenantID int64, it Item) error {
	_, err := s.w.ExecContext(ctx, `INSERT INTO items (tenant_id, id, title) VALUES (?, ?, ?)
		ON CONFLICT(tenant_id, id) DO UPDATE SET title = excluded.title`, tenantID, it.ID, it.Title)
	return err
}

// Clamp caller-supplied page sizes so one request can't scan the whole table.
func clampLimit(limit, def, max int) int {
	switch {
	case limit <= 0:
		return def
	case limit > max:
		return max
	}
	return limit
}
```

**Optional full-text search** (FTS5 is compiled into the pure-Go driver):
```sql
CREATE VIRTUAL TABLE search_fts USING fts5(
  tenant_id UNINDEXED, doc_id UNINDEXED, content,
  tokenize = 'unicode61 remove_diacritics 2'
);
```

### Env vars
- `APP_DATA_DIR` (directory holding the `.db` file). The DSN is built in code, so no secret is involved.

### Gotchas
- **Pragmas belong in the DSN.** `database/sql` pools connections, and a `PRAGMA busy_timeout` executed once applies to just one of them. `_pragma=name(value)` in the DSN runs on every new connection. `foreign_keys` in particular is per connection and off by default.
- **One writer connection.** SQLite has a single write lock. Letting the pool open several writer connections turns contention into `SQLITE_BUSY` errors and lock-upgrade deadlocks (a read transaction that later writes). `SetMaxOpenConns(1)` on the writer makes Go queue writes instead. Readers can stay parallel under WAL.
- **Don't read through the writer inside a long loop.** It holds the only writer connection, so other writes wait. Read with the reader pool, then write.
- **The busy timeout still matters** with a single writer: another process (a backup, the `sqlite3` CLI) can hold the lock. 5000 ms is a sensible default.
- **`synchronous=NORMAL` with WAL** is durable across application crashes and can only lose the last transactions on power loss. Use `FULL` if that is unacceptable.
- **PRAGMA inside a transaction**: `journal_mode` can't change inside a transaction, and `foreign_keys` is a no-op there. Split PRAGMA lines out of migration files and run them before the transaction.
- **Rebuilds and foreign keys.** SQLite's `ALTER TABLE` is limited, so changing a column means create-new, copy, drop, rename with `foreign_keys = OFF`. Always re-enable and run `foreign_key_check` afterwards, or a bad copy silently leaves orphan rows.
- **Migrations only move forward.** There are no down migrations. Take a backup before every upgrade, and roll back by restoring it.
- **One process per database file.** Run exactly one app instance against the file. Network filesystems (NFS/SMB) break SQLite locking, so keep the file on local disk.
- **Copy the `-wal` and `-shm` files together with the `.db`** when moving a live database, or use an online backup (`sqlite3 db ".backup out.db"` or `VACUUM INTO`). Copying only the `.db` loses recent commits.
- **Tenant scope is a code-review rule, not a database feature.** SQLite has no row-level security, so make the tenant id a required parameter of every store method and index `(tenant_id, …)` first in composite keys.
- **Test databases**: use a temp-dir file (`t.TempDir()`), not `:memory:`. Each pooled connection to `:memory:` is a different empty database.
- `modernc.org/sqlite` is somewhat slower than the cgo driver on heavy write loads, but you get static cross-compiled binaries and slim distroless or Debian images without a C toolchain.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace.

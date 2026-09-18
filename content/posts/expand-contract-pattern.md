+++
date = '2026-09-18'
title = 'The Expand-Contract Pattern'
description = "Zero-downtime schema migrations for tables that can't afford to stop"
series = ['system design']
tags = ['golang', 'system-design', 'postgres']
+++

Every production database eventually needs a schema change it wasn't built for: a column overflows, a table needs splitting, a type was wrong from the start. The change itself is simple. Doing it on a table taking thousands of reads and writes a second, without an outage, is not.

Expand-contract (aka parallel change) solves this by splitting the change into three phases — expand, migrate, contract — so old and new coexist for a while and every step is small enough to reverse.

## Why you can't just `ALTER TABLE`

Say `users.id` is a 32-bit `INTEGER` closing in on 2,147,483,647 rows. The obvious fix:

```sql
ALTER TABLE users ALTER COLUMN id TYPE BIGINT;
```

On a big table, this is a disaster:

- **It rewrites the whole table.** Postgres reads every row, writes it back with the new type, and rebuilds every index. Hours, for a large table.
- **It holds an `ACCESS EXCLUSIVE` lock the entire time.** No reads, no writes — the app is down until it finishes.
- **A failure wastes the whole window.** The DDL rolls back cleanly, but you've held the lock (and the outage) for nothing, and the next attempt starts from zero.
- **It's all-or-nothing.** No way to validate the new schema against real traffic before committing to it.

Expand-contract avoids all of this with one rule: never touch what's serving traffic. Build the new thing alongside it, prove it works, then retire the old one.

## Expand, migrate, contract

1. **Expand** — add the new schema (column, table, index) alongside the old one. Nothing changes for the application yet.
2. **Migrate** — writes start going to both old and new (dual writes), a background job backfills history, and once they're in sync, reads cut over behind a flag, gradually. This phase is where real traffic proves the new schema works.
3. **Contract** — once everything reads and writes through the new structure, drop the old column/table and the dual-write code.

Only one transition is ever "live," and each is cheap to undo — if migrate turns up a bug, flip reads back and you've lost nothing.

## Walkthrough: `INT` → `BIGINT` primary key

### Phase 1: Expand

```sql
ALTER TABLE users ADD COLUMN id_new BIGINT;
```

A nullable column with no default is metadata-only in Postgres — no rewrite, no meaningful lock.

`id` is a `SERIAL`, so Postgres assigns it on insert. The dual write is two statements in one transaction: insert, then copy the generated id into `id_new`.

```go
func CreateUser(ctx context.Context, db *sql.DB, name, email string) (*User, error) {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return nil, err
	}
	defer tx.Rollback()

	var id int64
	err = tx.QueryRowContext(ctx,
		`INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id`,
		name, email,
	).Scan(&id)
	if err != nil {
		return nil, err
	}

	if _, err := tx.ExecContext(ctx, `UPDATE users SET id_new = $1 WHERE id = $1`, id); err != nil {
		return nil, err
	}

	if err := tx.Commit(); err != nil {
		return nil, err
	}
	return &User{ID: id, IDNew: id, Name: name, Email: email}, nil
}
```

Every existing reader and writer still uses `id`. The catch: without a trigger, this invariant is enforced separately in every place that inserts a `users` row — a signup handler, an admin console, a bulk import — and any one that misses it creates a silent gap.

### Phase 2: Migrate

Backfill history in small batches, not one giant `UPDATE`:

```go
package main

import (
	"context"
	"database/sql"
	"log"
	"time"

	_ "github.com/lib/pq"
)

const batchSize = 5000

func backfillIDNew(ctx context.Context, db *sql.DB) error {
	for {
		res, err := db.ExecContext(ctx, `
			UPDATE users
			SET id_new = id
			WHERE id IN (
				SELECT id FROM users
				WHERE id_new IS NULL
				ORDER BY id
				LIMIT $1
			)
		`, batchSize)
		if err != nil {
			return err
		}

		rows, err := res.RowsAffected()
		if err != nil {
			return err
		}
		if rows == 0 {
			return nil // fully backfilled
		}

		log.Printf("backfilled %d rows", rows)
		time.Sleep(200 * time.Millisecond) // give the DB room to breathe
	}
}
```

Safe to stop and restart anytime — it just resumes from `id_new IS NULL`. Once it's done, add the constraints `id_new` needs, using Postgres's non-blocking variants:

```sql
CREATE UNIQUE INDEX CONCURRENTLY idx_users_id_new ON users (id_new);

ALTER TABLE users ADD CONSTRAINT id_new_not_null CHECK (id_new IS NOT NULL) NOT VALID;
ALTER TABLE users VALIDATE CONSTRAINT id_new_not_null;
```

`VALIDATE CONSTRAINT` scans the table but only takes a `SHARE UPDATE EXCLUSIVE` lock — reads and writes continue.

Then verify:

```sql
SELECT count(*) FROM users WHERE id_new IS DISTINCT FROM id;
-- expect: 0
```

Any table with a foreign key into `users.id` needs the same treatment — usually the longest part of a real migration. For `orders.user_id`:

```sql
ALTER TABLE orders ADD COLUMN user_id_new BIGINT;
CREATE INDEX CONCURRENTLY idx_orders_user_id_new ON orders (user_id_new);
```

Here the caller already has the value, so the dual write is just one extra column on the insert:

```go
func CreateOrder(ctx context.Context, db *sql.DB, userID int64, amountCents int64) (*Order, error) {
	var id int64
	err := db.QueryRowContext(ctx,
		`INSERT INTO orders (user_id, user_id_new, amount_cents) VALUES ($1, $1, $2) RETURNING id`,
		userID, amountCents,
	).Scan(&id)
	if err != nil {
		return nil, err
	}
	return &Order{ID: id, UserID: userID, AmountCents: amountCents}, nil
}
```

Backfill it the same way as `users.id_new`. Skip the foreign key constraint for now — Postgres needs a real `UNIQUE`/`PRIMARY KEY` constraint to reference, not a bare index, and wrapping `idx_users_id_new` in one now would tie it to that constraint's lifecycle, breaking the swap Phase 3 needs.

### Reads: cutting over behind a flag

```go
func GetUser(ctx context.Context, db *sql.DB, flags FlagClient, requestUserID int64, id int64) (*User, error) {
	query := `SELECT id, id_new, name, email FROM users WHERE id = $1`
	if flags.Enabled(ctx, "users-read-id-new", requestUserID) {
		query = `SELECT id, id_new, name, email FROM users WHERE id_new = $1`
	}

	row := db.QueryRowContext(ctx, query, id)

	var u User
	if err := row.Scan(&u.ID, &u.IDNew, &u.Name, &u.Email); err != nil {
		return nil, err
	}
	return &u, nil
}
```

Key the flag on something stable per caller so a given segment doesn't flap between columns. If the backfill missed an edge case Phase 2's verification didn't catch, only the flagged segment is affected, and flipping it off is an instant revert.

### Phase 3: Contract

`id_new` is populated, indexed, and verified. The cutover swaps which index backs the primary key — no rewrite. Old foreign keys must drop first, in the same transaction, since Postgres won't drop a referenced primary key:

```sql
BEGIN;
ALTER TABLE orders DROP CONSTRAINT orders_user_id_fkey;
ALTER TABLE users DROP CONSTRAINT users_pkey;
ALTER TABLE users ADD CONSTRAINT users_pkey PRIMARY KEY USING INDEX idx_users_id_new;
ALTER TABLE orders
  ADD CONSTRAINT orders_user_id_new_fkey
  FOREIGN KEY (user_id_new) REFERENCES users (id_new) NOT VALID;
COMMIT;

ALTER TABLE orders VALIDATE CONSTRAINT orders_user_id_new_fkey;
```

Everything in the transaction is a metadata-only catalog change. `VALIDATE CONSTRAINT` is the only step that scans `orders`, and like before, it only takes a `SHARE UPDATE EXCLUSIVE` lock. Repeat for every table with a foreign key into `users.id`.

Now clean up:

```sql
ALTER TABLE users DROP COLUMN id;
ALTER TABLE users RENAME COLUMN id_new TO id;
ALTER TABLE users DROP CONSTRAINT id_new_not_null;

ALTER TABLE orders DROP COLUMN user_id;
ALTER TABLE orders RENAME COLUMN user_id_new TO user_id;
ALTER INDEX idx_orders_user_id_new RENAME TO idx_orders_user_id;
ALTER TABLE orders RENAME CONSTRAINT orders_user_id_new_fkey TO orders_user_id_fkey;
```

`idx_users_id_new` doesn't need renaming — adopting it via `USING INDEX` back in Phase 3 already renamed it to `users_pkey`. The `orders` index and constraint were added directly, not adopted, so they need the explicit renames above.

`id_new` never got its own sequence (unlike the original `SERIAL`), so attach one before reverting `CreateUser` to a single insert:

```sql
CREATE SEQUENCE users_id_seq OWNED BY users.id;
SELECT setval('users_id_seq', (SELECT max(id) FROM users));
ALTER TABLE users ALTER COLUMN id SET DEFAULT nextval('users_id_seq');
```

Then drop the dual-write code in `CreateUser` and `CreateOrder`. Nothing in the schema hints that these columns used to be anything else.

## Timeline

For a table large enough that this is worth doing:

- **Expand** — hours. One deploy, low risk, nothing existing changes.
- **Migrate** — the long pole. Backfill takes anywhere from hours to days depending on table size and batch pacing. Soak with dual writes running and reads still on the old column for at least a full traffic cycle (a week is reasonable) to catch write paths that missed the dual-write change. Then stage the read cutover: internal traffic first, then a percentage, then everyone.
- **Contract** — minutes for the cutover itself, plus a day or two of soak before dropping the old column.

Total: weeks, mostly deliberate waiting. That's the trade — slower, in exchange for never having a moment where the system is down or unrecoverable.

## Trade-offs

**Costs:**

- More deploys, more elapsed time, than one `ALTER TABLE`.
- Every write path needs the dual-write change — no single choke point like a trigger would give you.
- `id` and `id_new` must stay in sync while dual writes run; rerun the verification query regularly, not just once.
- Rollback gets weaker over time — trivial early on, but once reads cut over, the old column is only a sync target you're not actively verifying anymore.

**Benefits:**

- No downtime — every phase runs against a live table.
- Reads and writes cut over independently, and read cutover itself is staged, so a bad interaction has a small blast radius.
- A real rollback path at every step.
- Each phase is its own deploy, giving you an audit trail for when something breaks later.

## When to skip it

This earns its complexity on large, high-traffic tables with real uptime requirements. For a small table, a staging database, or anywhere a maintenance window is fine, a direct `ALTER TABLE` during low traffic finishes in seconds — the three-phase version just adds process for no benefit.

The real question: can you afford a moment where the old and new schema can't both be true at once? If not, expand-contract is how you get from one to the other without ever answering yes.

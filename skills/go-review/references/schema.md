# Reference: schema migrations (`schema/*.sql`)

Load this when the diff touches `schema/*.sql`. The loader in `schema/schema.go` embeds every
`*.sql` and `db.ExecContext`s them **in lexical filename order**, re-applying the whole set
whenever any file changed — there is **no per-file version tracking**. `schema_migrations` holds
one row: the sha256 of the whole set, so an unchanged set is skipped entirely (the legacy
`schema_version` table is unrelated and still unused). Two consequences drive every rule below:

1. **Every statement must be idempotent** — it runs again on the next deploy.
2. **Editing an existing `create table` never alters an existing DB** — the table already exists,
   so `create table if not exists` is skipped and any newly-added column is silently ignored on
   deployed databases. New columns MUST come as their own `alter table` statement.

> ⚠️ This supersedes the old "modify the original CREATE TABLE, no ALTER files" convention (still
> written in some docs). The real, current practice is idempotent `alter table` files — see
> `0024`, `0025`, `0026`, `0029`, `0030`. Don't flag an `alter table … add column if not exists`
> file as wrong; it's the correct pattern.

## The patterns actually in use

- **New table** → new file `NNNN.sql` with `create table if not exists`, plus `create unique index if not exists` / `create index if not exists` for its indexes (e.g. `0027.sql`).
- **New column on an existing table** → new file with:
  ```sql
  alter table <t>
      add column if not exists <col> <type> not null default <v>;
  ```
  Multiple columns → one `alter table` with several `add column if not exists` clauses (`0024_admin_users.sql`).
- **Backfill** → `update <t> set … where <not-already-applied>` — the `where` must make it a no-op on re-run (`0025` guards with `where game_id is not null and game_ids = '{}'`).
- **Drop column** → `alter table <t> drop column if exists <col>` (`0025`).

## Naming & ordering

- Zero-padded 4-digit sequential prefix, applied in **lexical** order. Optional descriptive suffix: `NNNN_name.sql` (`0024_admin_users.sql`, `0030_account_lock.sql`).
- Same-number + suffix pairs exist (`0014.sql` + `0014_game_store.sql`, `0023.sql` + `0023_provider.sql`); lexical order puts the bare `NNNN.sql` before `NNNN_suffix.sql`. Fine, but don't introduce a suffixed file whose ordering matters relative to a later bare number.
- A new migration takes the next unused number. Reusing/renumbering an existing committed file is a 🔴 (it has already run on deployed DBs; the new content under an old name won't re-apply cleanly and breaks fresh-vs-existing parity).

## Column conventions (match the existing tables)

- IDs: `varchar(20)` (some `bigserial` for internal-only rows). Timestamps: `timestamptz not null default now()`; soft-delete `deleted_at timestamptz`.
- Money / rates / decimals: `numeric(14, 2)` (money) / `numeric(14, 8)` (rates) — mirrors `shopspring/decimal` on the Go side. 🔴 a `real`/`double precision`/`float` column for money.
- Arrays: `varchar(20)[]` / `text[] not null default '{}'`.
- Keywords lowercase (base schema is lowercase; `0024` drifted to uppercase — treat that as the exception, not the model). 🟡 uppercase keywords in a new file.

## Comments — keep them short

A migration file gets **at most a one- or two-line header comment** saying what it changes and, if
non-obvious, why. That's it. No multi-paragraph history, no explanation of the bug that motivated
the change, no notes on locking/rewrite behaviour — that belongs in the PR body or commit message,
which is where people actually look for it.

```sql
-- Rows the 0028 registry seed missed.
```

not

```sql
-- coin_transactions.method holds the payment channel by name, the same
-- payment_channels.channel value orders.payment_channel already stores.
--
-- It used to be a smallint mirroring a four-value enum in Go (card, promptpay,
-- ...
```

(Every file in `schema/` follows the short form as of this writing — keep it that way.)

## Review flags

- 🔴 **Non-idempotent statement** — `create table` / `create index` / `add column` / `drop column` without the matching `if [not] exists`. It errors on the second migrate run because every file re-executes.
- 🔴 **Column added by editing an existing `create table if not exists` block** — never reaches deployed DBs. Must be a separate `alter table … add column if not exists`.
- 🔴 **Renumbered/edited an already-committed migration** in a way that changes what a re-run does (breaks fresh-DB vs existing-DB parity).
- 🟡 **Backfill `update` without a guarding `where`** — reruns clobber data every deploy. Guard so it no-ops once applied.
- 🟡 Money column not `numeric`; uppercase keywords; missing index a query in the same PR relies on.
- 🟡 **Essay-length header comment** on a migration file — trim to one or two lines; move the history/rationale to the PR body.
- 🟢 Missing descriptive suffix on a themed migration; column alignment/formatting drift.

## PR note (git-workflow rule)

When a PR adds columns, the workflow still asks for an `ALTER TABLE … ADD COLUMN IF NOT EXISTS`
snippet in a PR comment for operators. With the alter-file pattern this is usually just the
content of the new migration file itself — point that out rather than hand-writing a divergent one.

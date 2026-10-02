---
name: go-review
description: Review Go code changes against personal Golang conventions (embedded checklist below). Use when user says "/go-review", "review my go code", "review go changes", or wants to audit a diff/PR against their Go rules. Also handles "/go-review setup godoclive" — wiring godoclive OpenAPI doc generation into a Go project.
---

# /go-review

Review Go code changes against the user's personal Golang conventions.

## Rule Sources

The full checklist is embedded below — self-contained, no external rule files to load. The one
exception is `~/.claude/CLAUDE.md`: the reviewer also applies its **Simplicity First** (comments only
where code needs clarification — never narration) and **Surgical Changes** rules.

References live alongside this skill in `references/`. **None are auto-loaded** — load one ONLY when the changed files match its workload, then review the relevant files against it.

- `references/project-structure.md` — full repo map, placement decision table, `api/` ↔ `pkg/` mirror, hard placement rules. Load ONLY when a change **adds, moves, or renames files across layers** (new endpoint / module / webhook / worker task / game-store proxy) or a file looks mislaid. For edits within existing files, the minimal placement pattern in the checklist is enough.
- `references/asynq-task.md` — load when the diff creates or changes a queue/task: a `worker/` task handler, an asynq task type + payload, or an enqueue call. Look for `asynq`, `asynqctx`, `worker.` enqueue helpers, or a `Handle*`/`Process*` task func.
- `references/schema.md` — load when the diff touches `schema/*.sql`. Covers the no-version-tracking / every-file-reruns loader, the idempotent `alter table … add column if not exists` pattern (NOT editing the original CREATE TABLE), naming/ordering, column-type conventions, and the one-or-two-line comment rule.
- `references/arpc-http-client.md` — load when the diff adds/changes a typed Go client that POSTs JSON to an arpc service and unwraps the `{ok, result, error}` envelope (look for a `Client` struct + `invoke`/per-method calls + `httpclient.Do` + envelope decoding).
- `references/godoclive-setup.md` — load ONLY for the `setup godoclive` mode (below), not during reviews.

## Setup Mode: `setup godoclive`

`/go-review setup godoclive` is **not a review** — it installs godoclive OpenAPI doc generation into the current Go project. Load `references/godoclive-setup.md` and follow it: create `cmd/docgen`, `.godoclive.yaml`, the `/docs` Scalar route, gitignore the generated spec, add a `Makefile` `docs` target and a CLAUDE.md section, then verify with `go run ./cmd/docgen`. Skip the review flow entirely.

## Flags

- (no flag) → **review only**, no edits. Default.
- `--fix` → after the review, apply 🔴 Must Fix items only. Safe edits with high confidence.
- `--fix-all` → apply 🔴 + 🟡 + 🟢 items. Use when you trust the rules and want everything aligned.

PR-scope review (`PR #123`) is **always review-only** — never `--fix` against a remote PR's working tree.

## What to Review

Reviewable files are `*.go` **and** schema SQL (`schema/*.sql`, `**/schema/*.sql`).
The checklist carries schema rules — column-change policy, lowercase keywords, index
coverage — so a review that skips `.sql` declares conventions it can never check.
Everything else in the diff (TS, YAML, Markdown, Dockerfiles) is out of scope.

Default scope (in order of preference):
1. If user passes args like `PR #123` or a PR URL → review that PR via `gh pr diff <n>`
2. If staged changes exist (`git diff --cached --name-only` returns reviewable files) → review staged
3. Otherwise → review unstaged + staged (`git diff HEAD -- '*.go' '*/schema/*.sql'`)
4. If nothing changed → review the current branch vs `main` (`git diff main...HEAD -- '*.go' '*/schema/*.sql'`)

**Audit scope** — `audit <path|package>` (e.g. `/go-review audit api/cart` or `audit api/cart/checkout.go`): review whole files at HEAD, no diff. Use for legacy code never covered by a diff review. The subagent Reads every non-test `.go` file under the path in full and runs the full checklist PLUS — mandatorily, not signal-gated — the Complexity Pass and Simplification Pass on every function that matches their activation signals. Start with the largest files (`wc -l`). For packages >3000 total lines, tell the user the file count and suggest narrowing before spawning. Audit scope is always review-only (`--fix` refused) — audit findings are refactor plans, not mechanical edits.

State which scope you picked in one line before diving in.

## Method

**Delegate to a Sonnet subagent.** Reading the full diff + every touched Go file is context-heavy work; it belongs in a fresh agent so the main session stays light.

1. Resolve scope first (in this session): run the `git`/`gh` command for the scope picked above and get the list of changed reviewable files (`*.go` + `schema/*.sql`). If zero → say so in one line and stop. Do NOT call the agent. **Never hand the agent a reviewable file marked "context only, do not review"** — that is how a schema file reaches a reviewer's eyes and still goes unreviewed. Context-only is for files outside the reviewable set.
2. Parse flags. If `--fix` against a PR scope → refuse and explain (`--fix` only works on local changes).
3. Spawn one `general-purpose` agent with `model: "sonnet"`. The prompt must be self-contained — the agent has no memory of this conversation. Include:
   - The absolute path `/Users/xkamail/.claude/CLAUDE.md` with instruction to read it first (Simplicity First + Surgical Changes).
   - Absolute paths to any **workload-specific references** whose workload the changed files match (see Rule Sources), with instruction to read and apply them. Include `references/project-structure.md` only when the diff adds/moves/renames files across layers; include `references/asynq-task.md` only when the diff touches queue/task code; include `references/schema.md` only when the diff touches `schema/*.sql`. Skip references that don't apply — don't make the agent read all of them.
   - The exact diff command to run, plus instruction to Read each changed file in full at HEAD for surrounding context.
   - The full checklist (copy the sections below into the prompt — don't reference "the SKILL.md").
   - An explicit instruction to run the **Complexity Pass** section's auto-detection: scan every changed function for its activation signals (transactions, row locks, concurrency, async side effects, >100-line/4+-helper functions, money computed in 2+ places) and deep-review correctness on the ones that fire, in addition to the style checklist. This is the highest-value part of the review — the style checklist can pass a function that has a rollback-ordering or lost-update bug.
   - The output format and severity guide.
   - The "Don'ts" list.
   - **Fix mode** (only if `--fix` or `--fix-all` was passed): instructions in the "Fix Mode" section below.
4. Relay the agent's report to the user verbatim, prefixed with the scope line. If fix mode ran, also include the "applied / skipped" summary the agent produces.

## Fix Mode (when `--fix` or `--fix-all` is passed)

Add this block to the subagent prompt:

> After completing the review, apply edits with the Edit tool:
> - `--fix` → apply only 🔴 Must Fix items.
> - `--fix-all` → apply 🔴 + 🟡 + 🟢 items.
>
> Rules:
> - Apply only fixes you can make with a deterministic, mechanical edit (rename `Request`→`Params`, remove `omitempty`, lowercase SQL keywords, swap `&x`→`new(x)`, switch `http.DefaultClient.Do`→`httpclient.Do`, swap `errors.As(err, &t)`→`errors.AsType[*T](err)`, swap `json.Marshal`+`bytes.NewReader`→`json.NewEncoder(&buf).Encode`, etc.).
> - **Skip** fixes that require judgment or multi-file refactors (route renames that touch callers, type changes from `float64` to `decimal.Decimal`, handler signature rewrites that change call sites, schema redesigns). Mark them `SKIPPED — needs human review` and explain why in one line.
> - Never delete code beyond what the fix requires.
> - Never run `go test`, `go build`, or any formatter — pure surgical edits.
> - Do NOT touch files outside the changed-files list.
> - After edits, emit an "Applied / Skipped" summary:
>   ```
>   ✏️ Applied
>     - file.go:42 — removed `omitempty`
>   ⏭️ Skipped (needs human)
>     - file.go:88 — float64→decimal.Decimal: 6 call sites, manual decision
>   ```

## Checklist (from rules)

### Naming & Structure
- Handler types use `Params` / `Result`, not `Request` / `Response`
- Handler signature: `func XXX(ctx context.Context, p *XXXParams) (*XXXResult, error)`
- Route naming: `{module}.{action}` — action does NOT repeat the module (e.g. `gamestore.create`, not `gamestore.createStore`)
- No `internal/` package — flat `api/` + `pkg/` only
- No `/api/v1` prefix in handler code (belongs in `mux.Group`)
- List item types named `Item`, not `StoreItem` / `OrderItem` etc.
- File section order, top to bottom: (1) sentinel error declarations (`var ErrXxx = ...`), (2) public function surface (exported funcs), (3) local/shared/reuse code (unexported funcs & helpers). Errors first, exported above unexported. Flag files where a sentinel error is declared mid-file or an unexported helper sits above an exported func.
- No alias imports unless there's a real package-name collision; collision case uses `{module}v1`, not `pkg{module}`

### File Placement (minimal — load `references/project-structure.md` for the full map)
- `api/{module}/{module}.go` holds the arpc handler funcs. Routes are registered centrally in `api/handler.go`'s `Mount(mux, am, db)`; `cmd/api/main.go` creates the mux + `arpc.Manager` and only calls `api.Mount(...)` — no mux/route init inside `handler.go`. Same shape for `backoffice/` and `callback/`.
- A route whose handler reads `auth.*(ctx)` (`UserID`/`AccountID`/`EmployeeID`) MUST register on the authed group (`a := mux.Group("", am.Middleware(authMiddleware))` → `a.Handle(...)`), not the public `mux.Handle(...)`. Wrong group = empty/panicking auth context at runtime → 🔴. Per-route middleware chains as `.Middleware(mw).Handle(...)`; a shared prefix+middleware as `mux.Group(prefix, mw...)`.
- Logic reused by both a handler and a worker task lives in `pkg/{module}/` — duplicating it in the handler is a 🔴.
- New endpoint → `api/{module}/` + route in `api/handler.go`. New webhook → `callback/{provider}/`. New async task → `worker/`. New game-store proxy → `cmd/{provider}/`.
- **Helpers live in the package of their real domain, not their first caller's.** A helper whose logic doesn't depend on its package's domain (address/currency/phone parsing, formatting, a shared enum list) is misplaced there. Check every new cross-package import the diff adds: if feature A imports `pkg/B` only for a helper unrelated to B's domain (e.g. `order` calling `refund.ParseAmount`), flag it → 🟡 and name the domain package to move it to. This applies even when the helper predates the diff — the new caller is what exposes the misplacement. Also flag an error code whose prefix names the wrong domain for the caller (a `refund/invalid-amount` returned from an order endpoint).
- **No duplicate of an existing helper.** For every new exported func/var in `pkg/`, grep the repo (not just the diff) for an existing equivalent by behavior — same inputs/outputs, similar name (`Format`/`FormatX`, `Parse`/`ToX`/`Normalize`). A second copy → 🟡: delete it and use the existing one. Reading only changed files cannot catch this; run the grep.

### File Organization (one domain per file)
- A package file should hold ONE domain. When a single file accumulates many handlers spanning distinct domains, split it into per-domain `*.go` files (e.g. `registration.go`, `bracket.go`, `match.go`, `standings.go`, `prizes.go`) named for the domain — not one catch-all `<package>.go` holding everything.
- Mirror the existing `*_test.go` layout: if `register_test.go` / `get_match_test.go` exist, the source belongs in matching domain files. Source-file names should track test-file domains.
- The package's core file (`<package>.go`) keeps only cross-domain shared declarations: sentinel errors, shared types, and helpers used by 2+ domains. A helper used by exactly one domain lives in that domain's file (and at the bottom, per file section order).
- Group related handlers of the SAME domain together in one file (e.g. `Register`/`Unregister`/`UpdateRegistration`/`Registrations`/`MyRegistration` → `registration.go`) rather than one file per handler.
- Flag a 🟡 when one non-test file holds **3+ exported handlers of distinct domains** OR exceeds **~600 lines** while mixing domains — and name the concrete split (list the domain files). A long file that is genuinely one cohesive domain is fine; size alone is not a violation.
- Escalate to 🔴 when a mixed-domain file passes **~1200 lines** or **~1500 lines with 3+ domains** — at that size the file is a maintenance and merge-conflict hazard, not a style nit. Do NOT soften this into "would not do it opportunistically"; state the concrete split as a required follow-up (e.g. `checkout.go` → `checkout.go` orchestration + `stock.go` locking/reservation + `coupon.go` validation + `campaign.go` overrides, with shared sentinel errors/types left in the core file). The reviewer's job is to name the split, not to defer it.

### JSON & Types
- No `omitempty` in JSON struct tags — every field always emitted
- **API docs come from doc comments (godoclive).** The OpenAPI spec is generated by `cmd/docgen` analyzing every handler's `Params`/`Result` structs — there is no hand-written spec. A new/changed endpoint's request **and** response structs (plus nested list `Item`s) should carry doc comments: a comment on the line(s) directly above each field → that field's `description`; the comment above `type Xxx struct` → the schema/endpoint description. Flag added/renamed exported `Params`/`Result` fields with no doc comment → 🟡. Notes: optional pointer fields should state null/zero-value meaning; a comment containing `Deprecated:` marks the field deprecated in the spec; a trailing same-line comment is only a fallback (prefer the comment above the field)
- **Field doc comments follow Go doc style — start with the field name.** godoc convention (and `go doc` output) wants `// CreatedAt is when the order was placed.`, not spec prose like `// Time the order was placed.` or `// When the account was created.` Rules:
  - Begin with the exact field identifier, then a verb: `// FieldName is …` / `// FieldName reports whether …` (bool) / `// FieldName returns …`. Same for the struct comment: `// Item is the public tournament view …`.
  - Full sentence, ends with a period. English only.
  - Don't echo the name with no information (`// ID is the ID`) — say what it identifies, its unit, or its source (`// AmountTHB is the charged amount in satang.`).
  - Enum-ish strings list their values (`// Status is one of pending, paid, cancelled.`); pointer fields state what nil means (`// Slug is the vanity URL segment; nil when unset.`).
  - No section-label comments inside a struct (`// Time`, `// Meta`) — those read as a doc comment for the field directly below them and land in the OpenAPI spec as that field's description. Group with a blank line instead.
  - Flag a field comment that doesn't start with its field name → 🟡, with the rewritten line.
- `shopspring/decimal` for money, balances, rates, quantities, in-game currencies, ledger fields — never `float64` or raw `int*` for these
- `int*` only for IDs, indexes, levels, enum counters, pagination

### Handlers & Validation
- `if err := p.Valid(); err != nil { return nil, err }` is the first statement in every `api/` and `callback/` handler
- **`backoffice/` inverts this**: `iam.Authorize(ctx, permission.X)` is first, `p.Valid()` second — an unauthorized admin must not reach validation (or the field normalization `Valid()` performs)
- Errors go through arpc codes — no `fmt.Errorf` for user-facing errors; return `err` directly or use `arpc.NewErrorCode`
- No empty-string guards on internal args (`if x == "" { return err }`) — trust callers; validate only at HTTP boundary (`Valid()`)
- `auth.UserID(ctx)` / `auth.EmployeeID(ctx)` / etc. — never checked for empty; auth middleware guarantees they're set
- Prefer `var r XxxResult` + field assignment over struct literal init for multi-line population

### Validation (`Valid()` method)
- Use `github.com/moonrhythm/validator`: `v := validator.New()`, one or more `v.Must(cond, "message")` calls (or `v.Mustf(cond, "message %d", i)` for messages with loop indices), then `return v.Error()`
- Field normalization (`strings.TrimSpace`, `strings.ToUpper`, etc.) happens inline at the top of `Valid()`, mutating `p`'s fields directly, before the `v.Must` checks that depend on it
- Use `github.com/asaskevich/govalidator` helpers inside `v.Must(...)` for format checks: `govalidator.IsEmail`, `IsURL`, `IsNumeric`, `IsLowerCase`, etc. — don't hand-roll regex for these
- Nested/composed param structs (e.g. `p.Sort`, `p.Paginate`) delegate via `if err := p.Sort.Valid(); err != nil { return err }` rather than re-checking their fields inline
- Fixed-length ID fields check exact length (`len(p.ID) == 20`), not just `> 0`

### SQL & Database
- All SQL keywords lowercase (`select`, `from`, `where`, `join`)
- No `::text` casts — Go's `database/sql` handles conversion
- Dynamic queries (optional filters, search, conditional WHERE) → `pgstmt`. Raw SQL via `pgctx` only for fixed queries
- One column per line in SQL strings; one arg per line in `Scan(...)`
- Multi-row reads use `pgctx.Iter` — never `pgctx.Query` + manual `rows.Next()`
- Single row → `pgctx.QueryRow`. Writes → `pgctx.Exec`
- Slices initialized empty before `pgctx.Iter`: `items := make([]Item, 0)` (not `var items []Item`)
- `pgsql.IsUniqueViolation(err)` checked BEFORE generic error return; returns `ErrDuplicate`
- Schema changes are idempotent files run on every migrate (no version tracking): new table → new `NNNN.sql` with `create table if not exists`; new column → new file with `alter table … add column if not exists` (editing the original `CREATE TABLE` does NOT reach existing DBs). Header comments in migration files stay to one or two lines — rationale/history goes in the PR body, not the `.sql`. See `references/schema.md`
- JSON/JSONB columns use `pgsql.JSON(&field)` in both Scan and query args — never manual `json.Marshal`/`Unmarshal`
- Nullable scans use `pgsql.Null(&field)`

### Schema Files (`schema/*.sql`)
- A new column goes in a new file as `alter table … add column if not exists` — editing the
  original `CREATE TABLE` does not reach existing databases (see `references/schema.md`)
- Every statement is re-runnable: `create table if not exists`, `create index if not exists`,
  `add column if not exists`. `Migrate()` replays every file on each deploy
- All keywords lowercase; one column per line
- Every foreign key states its `on delete` behavior explicitly
- Every column a `where`/`join`/`order by` in the Go diff touches is index-covered; say which
  query each new index serves. Flag an index the primary key already covers as redundant
- **Comment budget**: a schema comment says what the table is and the one invariant a reader
  cannot infer from the DDL. It is not the design doc — link that instead. More than ~4 lines
  of prose over a small table is a 🟡, and rationale copied verbatim from a design doc or issue
  body is the usual cause. The same budget applies per-column and per-index

### pgstmt Patterns
- `b.Columns("col1", "col2")` — separate args, not one comma-joined string
- Select: `pgstmt.Select(func(b pgstmt.SelectStatement) { ... }).IterWith(ctx, func(scan pgsql.Scanner) error { ... })`
- Update: `.ExecWith(ctx)`
- `b.Where(func(c pgstmt.Cond) { c.Eq("col", val) })`; multiple conds → `c.Mode().And()` / `c.Mode().Or()`
- Conditional filters guarded with `if` inside the Where callback
- Null checks → `c.IsNull("col")` / `c.IsNotNull("col")`, never `c.Raw("col is null")`
- Search uses a separate Where block with `c.Mode().Or()` + `c.ILike(...)`
- Join → `b.Join("table alias").On(func(c pgstmt.Cond) { c.EqRaw("a.col", "b.col") })`
- Partial update: `if p.Field != nil { b.Set("field").To(*p.Field) }`
- Raw SQL values: `b.Set("updated_at").ToRaw("now()")`

### Pagination (`pkg/restful`)
- Params embed `*restful.Paginate` with json tag `"paginate"`; `Valid()` checks `p.Paginate != nil`
- Result embeds `*restful.Paginate` with json tag `"paginate"`; initialized as `r.Paginate = p.Paginate`
- `r.Items = make([]Item, 0)`
- Count query first via `r.Paginate.Count(func() (int64, error) { ... })` using `pgstmt.Select` with `count(*)`
- Data query applies `r.Paginate.Do(b)` inside the `pgstmt.Select`

### General Patterns
- Inline over single-use helpers — see the Helpers & Utils section
- Pointers via `new(value)`, not `&variable` (e.g. `pgsql.Null(new(adminID))`)
- Prefer `slices` package over manual for-loops
- Prefer `json.NewDecoder(r.Body).Decode(&v)` over `io.ReadAll` + `json.Unmarshal` when reading a body
- Prefer `json.NewEncoder(&buf).Encode(v)` into a `bytes.Buffer` over `json.Marshal` + `bytes.NewReader` when building a request body
- Type-assert errors with `errors.AsType[*T](err)` (Go 1.26), not `errors.As(err, &target)`
- Env vars via `configfile.NewEnvReader()` — never `os.Getenv`
- HTTP via `httpclient.Do` — never `http.DefaultClient.Do`
- Routing via `github.com/moonrhythm/httpmux` (`httpmux.New()`, `.Group(prefix, mw...)`, `.Middleware(mw).Handle(...)`) — never stdlib `http.ServeMux` / `http.NewServeMux`. Middlewares are `httpmux.Middleware` (`func(http.Handler) http.Handler`) applied via `.Middleware(...)` / `.Group(...)`, not hand-wrapped `http.Handler`s. Method routing uses the `"POST /module.action"` pattern string
- Logging via `slog.InfoContext` / `slog.ErrorContext` — never the non-context variants
- Shared logic between worker and API in `pkg/{name}/`
- REST utilities (`Paginate`, `Sort`, etc.) in `pkg/restful` or `pkg/rest`

### Effective Go idioms

Baseline from [Effective Go](https://go.dev/doc/effective_go) + [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments). The house rules above win on any conflict (e.g. `var r XxxResult` over literals, `new(value)`, near-zero body comments). Flag only what the diff introduces.

**Names**
- Package names: short, lowercase, one word, no `_`/mixedCaps, no `util`/`common`/`base`. No stutter — `order.Service`, not `order.OrderService`; `gamestore.New`, not `gamestore.NewGameStore` → 🟢.
- MixedCaps only, never `snake_case`. Initialisms keep one case: `ID`, `URL`, `HTTP`, `JSON`, `API` (`userID`, `apiURL`, `ServeHTTP` — not `userId`, `ApiUrl`) → 🟢.
- Getters drop `Get`: method `Owner()`, setter `SetOwner()`. (Arpc handler funcs named `Get`/`List` are route actions, not getters — don't flag.)
- One-method interfaces take the `-er` name (`Reader`, `Notifier`). Don't reuse a canonical method name (`String`, `Read`, `Write`, `Close`, `Error`) with a different signature or meaning → 🟡. Short names for short scopes (`i`, `r`, `p`); descriptive names for wide scopes and package-level vars.
- Receiver name: 1–2 letters from the type, same across all methods; never `this`/`self`.

**Control flow**
- Happy path at the left margin: handle the error and `return` early; no `else` after a block that ends in `return`/`break`/`continue` → 🟢 (mechanical under `--fix-all`).
- Scope short-lived values to the statement: `if err := f(); err != nil`, `if v, ok := m[k]; ok`.
- `:=` in an inner block that re-declares an outer `err`/result var shadows it — the outer value is never set → 🔴 when the outer one is returned or checked later.
- A 3+ arm `if/else if` chain on one value → expression-less `switch` or `switch v`. Type dispatch → type switch, not repeated assertions.
- Assertions and map reads that can miss use comma-ok (`v, ok := x.(T)`); a bare `x.(T)` that can panic on untrusted input → 🔴.

**Functions & defer**
- `defer` args are evaluated when the `defer` runs, not at return (`defer log(time.Since(start))` logs ~0) — wrap in a closure when you want the late value → 🟡. Deferred calls run LIFO.
- `defer` cleanup right after the acquire succeeds (`f, err := os.Open(...); if err != nil {...}; defer f.Close()`). `defer` inside a loop holds every resource until the function returns → 🟡 (move the body into a func or close explicitly).
- Named results only when they document same-typed returns or a deferred func must modify them; no bare `return` in functions longer than a few lines → 🟢.
- Don't pass a pointer to save copying a small struct/string/slice/map; pass a pointer when the callee mutates or the struct is large.

**Data**
- Known final length → preallocate: `make([]T, 0, len(src))` (still satisfies the "empty, not nil" rule for results).
- `append` may reallocate — always assign the result back (`s = append(s, x)`); appending to a sub-slice that shares a backing array with a live slice silently overwrites it → 🔴.
- Map/type-assertion misses return the zero value; use comma-ok when zero is a valid value (`qty, ok := m[id]`).
- Receivers: pointer if any method mutates or the struct is large/holds a mutex; then **all** methods use pointer receivers — mixed receiver kinds on one type → 🟢. Never copy a struct containing `sync.Mutex`/`sync.WaitGroup` (value receiver, range copy, pass by value) → 🔴.
- Zero value should be useful: prefer structs that work without a constructor; don't add `NewX()` that only returns `&X{}`.
- `iota` enums get a `String()` via `stringer` (`//go:generate stringer`, `make generate`), not a hand-written switch.
- A `String()` method that formats its own receiver with `%v`/`%s` (`fmt.Sprintf("%v", t)`) recurses forever → 🔴; convert first (`string(t)`, or a field).
- Struct embedding promotes methods, but the receiver is still the inner type — flag embedding used as "inheritance" where the outer type expects its own fields to be seen → 🟡. Embed to expose the inner API; use a named field when you only need to delegate a method or two.

**Interfaces**
- Define interfaces where they're consumed, not next to the implementation; keep them small. Functions accept interfaces and return concrete types — returning the interface is right only when the type has no behavior beyond it (Effective Go "Generality", e.g. swappable gateway/SMS providers) → 🟡 otherwise.
- No interface with a single implementation "for mocking" unless a test fake actually implements it (compile-time `var _ I = T{}` check — only needed where no static conversion already proves it).

**Errors**
- Error strings: lowercase, no trailing punctuation, and name their origin (`"gamestore: unknown provider"`, `fmt.Errorf("load order: %w", err)`) — not `"failed to …"`/`"error: …"` noise → 🟢.
- Internal wrapping uses `%w` (never `%v` for an error you might `errors.Is` later); compare with `errors.Is`, never `==` or `err.Error()` string matching → 🟡. (User-facing errors are still `arpc` codes — see Handlers.)
- Handle an error once: log **or** return, not both → 🟡.
- `panic` only for programmer errors / impossible states at init (`regexp.MustCompile`, missing required config at startup). A `panic` on a runtime/input path → 🔴. `recover` only at a goroutine/request boundary, inside a deferred func.
- A panic in a goroutine spawned from a handler/task kills the whole process — the httpmux/asynq recovery only covers the request goroutine. Long-lived or fire-and-forget `go func` needs `defer func() { if r := recover(); r != nil { slog.ErrorContext(...) } }()` → 🟡.

**Concurrency** (details in the Complexity Pass)
- "Do not communicate by sharing memory; instead, share memory by communicating": prefer handing ownership through a channel or `errgroup` over a mutex-guarded shared slice when it's the simpler shape.
- Every goroutine has a clear exit — ctx cancellation or a closed channel. A goroutine blocked forever on a send nobody reads (unbuffered chan, early-returning receiver) is a leak → 🔴. Only the sender closes a channel.
- Unbounded `go` per item (one goroutine per row/request) → bound it: `errgroup.SetLimit(n)`, a buffered-channel semaphore, or a fixed worker pool reading one channel → 🟡.
- `context.Context` is the first param named `ctx`; never stored in a struct field; never `nil` (use `context.Background()`/`TODO()`).

**Misc**
- `gofmt`/`goimports -local <module-path>` clean (import groups: stdlib, third-party, local).
- Blank imports (`_ "pkg"`) only in `main` or tests, with a one-line reason.
- Exported names in `pkg/` carry a doc comment starting with the name (`// Charge debits …`) → 🟢 when a new exported func/type lacks one.

### Comments & Prose
The comment rule lives in `~/.claude/CLAUDE.md`, not in this checklist: *comments only where code
needs clarification (non-obvious why, gotchas) — never narration of what the code plainly does*.
Apply it to `.go` files with the same budget the schema section applies to `.sql`:

- A doc comment states what the thing is plus, at most, the one **why** a reader cannot infer from
  the body. Rationale copied from a design doc or an issue body belongs behind a link to it — a
  one-line pointer (`// See docs/....md`) replaces a paragraph
- **A file-header block over ~8 lines is a 🟡**, and a header that re-states rules the functions
  below already document is the usual cause
- Flag a comment that narrates the code beneath it, restates the function name in a sentence,
  labels an obvious struct, or repeats a point another comment in the same file already made
- Keep: the locking/ordering reason behind a transaction, why an error is mapped to a specific
  sentinel, a rejected alternative that will otherwise be "fixed" back in, a non-obvious SQL
  operator. Cut: everything that reads as an essay
- Rough signal to check, not a hard limit: comment lines much past ~10% of a non-test Go file
  usually means prose that belongs in the design doc or the PR body. **State the ratio you
  measured** (`wc -l` vs `grep -c '^\s*//'`) when raising it, so the finding is falsifiable
- In `_test.go`, a one-line comment pinning *what invariant a case exists to prove* is wanted —
  judge those on whether they say something the assertions do not

### Helpers & Utils (extract only when actually shared)

The owner dislikes helper sprawl: a wall of small `utils`-style functions each called from exactly one place, splitting one readable flow across a dozen jumps.

**The bar for a new unexported func: 2+ real call sites, today.** Not "it might be reused", not "it's cleaner". One caller → inline it back into the caller.

**Flag → 🟡 inline and delete:**
- An unexported func with exactly one call site whose body is short or exists only to name a step (`buildFilter`, `mapItems`, `calcTotal` called once). Name the caller and the exact lines to fold back.
- A wrapper that only forwards args to another call, adding no logic (`func getUser(ctx) { return user.Get(ctx, auth.UserID(ctx)) }`).
- Generic `utils.go` / `helper.go` / `common.go` files. There is no `utils` domain — a helper lives in the file of the domain that uses it (per File Organization), and a genuinely cross-domain one goes in the package's core `<package>.go`. Flag the file itself and say where each func belongs.
- A helper extracted "for testing" that the tests don't actually call directly.

**Zero call sites → 🔴 delete** (dead code; grep the package before claiming this).

**The exception, and it's real:** duplication across handlers beats this rule. When two handlers share a flow tail verbatim, ONE shared function is correct even though each new call site is "single-use" from its own file's view — see the Sibling-handler duplication check. Two drifting copies of an order path is a bug source, not a style choice.

Don't flag: exported package API (that's the package's surface, callers are elsewhere), a single-use func that genuinely tames deep nesting or is recursive, and `t.Helper()` test helpers.

### Comments (implementation only — fewer is better)

**Scope: comments inside function bodies.** Doc comments (above a type, field, func, or const) are OUT OF SCOPE for this rule — never flag or delete them; the `Params`/`Result` field docs generate the OpenAPI spec.

**Telling them apart — indentation does NOT.** A field doc comment inside a `type X struct { ... }` block is indented exactly like a body comment, and a naive "indented `//`" scan flags hundreds of them. Before flagging, confirm the comment sits between `func ...{` and that func's closing brace. A comment directly above a struct field, a const in a `const (...)` block, or an interface method is a doc comment — skip it.

**Never touch, at any length:**
- Tool and IDE directives: `//go:generate`, `//go:embed`, `//go:build`, `//nolint:...`, `// language=SQL` (JetBrains SQL injection hint — deleting it kills SQL highlighting/validation in the editor), `// Deprecated:`.
- Anything in a generated file — header `// Code generated ... DO NOT EDIT.`, or a `*_string.go` stringer output. Skip the whole file.

Inside an implementation, the owner wants near-zero comments. The code should read without narration. Two things get flagged: comments that shouldn't exist at all, and comments whose content is fine but that run long.

**Keep:**
- The *why*: a non-obvious business rule, a workaround with an issue link, a deliberate deviation (`// lock in id order to avoid deadlock with checkout`).
- The reason on a deliberately ignored error (`_ = x() // best-effort; logged upstream`).

**Flag → 🟡 delete:**
- Restating the next line (`// increment counter`, `// begin transaction`, `// fetch translations`, `// Verify campaign exists`).
- Bare label comments over a block (`// Sort`, `// auth`, `// start of this month`) — a label is not information; rename the variable or extract nothing.
- Step narration (`// 1. validate`, `// 2. load order`) — if the function needs a map to follow, split it instead.
- Section banners inside a body (`// ---- helpers ----`).
- Commented-out code. Delete it; git has it.
- TODO/FIXME with no owner or issue reference.
- A comment that has drifted from the code below it — 🔴 when it would mislead a reader into a wrong change.

**Length: a body comment is ONE line. Two at the absolute most.**

This is the rule that fires most often in practice, and it applies even to a *why* comment whose content is correct. A 4-6 line prose paragraph explaining rationale mid-function is still comment noise — it buries the code it sits above and it goes stale as a block. Flag any body comment block of **3+ lines → 🟡: compress to one line**, or, if the rationale is genuinely load-bearing for the whole function, move it up into the function's doc comment where it belongs. Relocation is not a loophole: a func doc comment gets the same budget (1-3 lines). Moving a 6-line essay up one level leaves total comment volume unchanged, which is the thing being flagged.

Compress by keeping the load-bearing clause and cutting the narration around it:

```
// The draw is not atomic — the generators write rounds, then matches, then
// seat participants, each in its own statement. The organizer polls the
// bracket every 2s while waiting, and that read caches any result with at
// least one round for the full TTL, so a poll landing mid-draw pins a
// round-shaped bracket with no entrants in it. Drop that here, once the
// bracket is whole.
```
→ `// Drop the cache: a poll landing mid-draw pins an entrant-less bracket for the full TTL.`

Never compress by deleting the fact. If a 5-line block can't survive as one line, that's a signal the function needs splitting or the invariant needs a named helper — say which, don't just shrug and keep the essay.

**Density.** More than ~1 body comment per 15 lines of non-trivial logic is a smell — usually bad names or a function that wants splitting. Report once per function (`file.go:120 — orderCheckout: 9 body comments in 60 lines, most restate the code. Keep the lock-ordering note, delete the rest.`), not line by line.

Under `--fix` / `--fix-all`, deleting a restating body comment is a safe auto-fix. Compressing a long *why* comment is a rewrite of meaning — do it only under `--fix-all`, and show the before/after line in the report so the dropped rationale is reviewable.

### Errors are never silently ignored
- Every returned `error` is checked or explicitly assigned to `_` with a reason in a comment

### Complexity Pass (correctness — auto-detected)

The rest of this checklist is convention/style matching. This section is different: it reviews **logic correctness**, and it only activates when the diff carries complexity signals. Run it **in addition to** the style checklist — never instead of it.

**Activation.** Turn this pass on for any changed function that matches ONE OR MORE of:
- a DB transaction — `pgctx.RunInTx` / `RunInTxOptions` / `MaxAttempts` / `TxOptions`
- row locking — `for update`, `lockXxx` helpers, `select ... for update`
- concurrency — `go func`, `sync.WaitGroup`, channels, `errgroup`, shared mutable state across goroutines
- async side effects — `task.Create`, asynq enqueue, `dispatchXxx`, any fire-and-forget started from a handler
- a function **> ~100 lines** or that fans out to **4+ same-package helpers**
- money/quantity arithmetic that is **computed in more than one place** for the same order/cart/ledger

If none match, skip this section silently (don't mention it). If any match, name which signal fired for each function you deep-review.

**1. Transaction correctness**
- Side effects that must not fire on rollback (enqueue, external call, cache write, notification) run **after** the `RunInTx` closure returns `nil` — never inside the closure. Pattern: capture what you need in a struct + a `runX bool` inside the tx, dispatch after. Flag any enqueue/HTTP/dispatch **inside** the closure → 🔴 (fires even when the tx aborts).
- `MaxAttempts: 1` under a serializable/repeatable-read tx means a concurrent-update serialization error (`pq` `40001` "could not serialize access") surfaces **raw** to the caller. Confirm the caller/handler maps it to a user-facing `arpc` code (use `pgsql.IsSerializationFailure(err)` — don't hand-unwrap `*pq.Error`; the package also has `IsErrorCode`/`IsErrorClass` for other codes) or the test asserts the abort is acceptable — a raw serialize error reaching the client is 🔴; an intentional single-attempt with a documented "loser is rejected" contract is fine (note it).
- Lock ordering: when a tx `FOR UPDATE`-locks **two or more tables** (e.g. `game_items` then `campaign_schedule_items`), every code path that locks the same pair must lock them in the **same order**, or two concurrent txs deadlock. Flag inconsistent ordering → 🔴.
- The rows locked `FOR UPDATE` must be the rows later mutated. Locking table A but `update`-ing table B under contention is a lost-update bug → 🔴.

**2. Concurrency & races**
- Distinguish **advisory** checks from **authoritative** ones. An availability/stock/balance check that does NOT take a lock or write is advisory — it can pass for N concurrent callers. The authoritative guard must be a locked read + conditional write **inside** the same tx that commits the effect. Flag when the only guard against oversell/overspend is an unlocked `select` → 🔴.
- TOCTOU: any "check then act" on shared state (stock, balance, usage count, uniqueness) where the check and the act are not in one atomic locked/tx step, or not backed by a DB constraint → 🔴.
- Goroutines: every `go func` writing shared state needs a mutex/channel; every spawned goroutine's error must be collected (channel / errgroup), not dropped. `WaitGroup.Add` before the `go`, `Done` via `defer`. Flag races and dropped goroutine errors → 🔴.

**3. Logic consistency**
- A value computed **twice** for the same entity (classic: coupon/discount estimated pre-lock in the handler, then re-computed post-lock in `createOrder`) must reconcile — the authoritative one wins and the estimate must not leak into the persisted row. Flag when the two can silently diverge (different inputs, different rounding, one path skips a cap) → 🟡, or 🔴 if the wrong one is persisted/charged.
- Money rounding: rounding direction and precision consistent across the flow; discounts never make `paid < 0`; a cap (`MaxDiscountAmount`, eligible amount) is actually applied, not just computed. `decimal` compared with `.Equal`/`.GreaterThan`, never `==`.
- Quantity splits (campaign vs base) must sum back to the requested quantity — flag off-by-one / leftover-unit bugs. Status/enum: every branch assigns a reachable terminal status; no path leaves status unset.

**4. Failure & idempotency**
- Partial failure: if step 3 of 5 fails, do steps 1–2 leave dangling rows / reserved stock / charged balance? Effects that must be atomic belong in ONE tx; effects that can't (external calls) need a compensating path or an idempotent retry.
- Retried async tasks (asynq redelivers) must be idempotent — a task that increments a counter, decrements stock, or credits balance without an idempotency key/guard double-applies on retry → 🔴.
- `pgsql.IsUniqueViolation` (and other expected DB errors) mapped to a domain error **before** the generic `return err`, so a race that trips a unique constraint becomes a clean `ErrDuplicate`, not a 500.
- User-facing errors are `arpc` codes; a raw `pq`/driver/`fmt.Errorf` error reaching the client leaks internals → 🔴.

### Simplification Pass (large files / large functions)

When the Complexity Pass activates on a function >~100 lines, or any changed file is a mixed-domain file >~600 lines, also hunt for **generated bloat** — code that looks necessary but does nothing. These checks found ~100 dead/duplicated lines and 3 real bugs in a single 1885-line audit; they pay for themselves.

- **Write-only param fields.** For every field of an internal params struct (like `createOrderParams`), verify it is *read* inside the callee — not just populated at the call site. A write-only field is dead, and so is the entire upstream computation chain that feeds it (trace it: the dead field often keeps a whole helper + adjustment blocks alive in 2+ callers). Flag the field AND the chain → 🟡, with exact lines to delete.
- **Estimate-then-recompute.** When a handler computes a value pre-transaction and the transaction recomputes it authoritatively (post-lock prices, in-tx coupon revalidation), the pre-tx result is usually dead except for early error surfacing and cheap flags. Keep the call only for what's actually consumed; delete the arithmetic that feeds nothing.
- **Zero-caller functions.** grep every unexported function in the touched file for callers; zero call sites = delete → 🟡.
- **Sibling-handler duplication.** When two handlers share the same flow tail (e.g. Checkout and QuickBuy: validate game → validate channel → coupon → store check → createOrder), quantify the verbatim overlap. ≥2 real call sites justifies ONE shared flow function (the "inline over single-use helpers" rule cuts the other way here). Two drifting copies of an order-placement path is a correctness time bomb, not a style choice.
- **Refactor gate: pins first.** Never propose a behavior-touching simplification without stating which existing test pins it (file:test name) or naming the missing test that must land FIRST. Untested branches to check for specifically: caps that every fixture sets to zero (`max_discount_amount = 0` seeds never exercise the cap), time-window guards, reject paths where only the accept path is tested, and boundary values (paid = 0, exact balance).
- **Money invariants.** `paid`/`total` amounts must be provably ≥ 0 across every discount combination (fixed coupon + percent discount stacking is the classic negative-paid hole — and a follow-up `Abs()` deduct charges the user for it). A missing clamp is 🔴.
- **`for update` needs deterministic order.** `where id = any($1) ... for update` without `order by id` deadlocks two overlapping multi-row orders. A comment claiming "consistent order" doesn't make it true — the SQL must. 🔴.

**Generation-time corollary** (applies when this skill's owner is *writing* a complex endpoint, not just reviewing): plan the domain split BEFORE writing (orchestrator + per-domain files); give every computed value exactly one authoritative computation site; when a second handler needs the same flow, extract the shared function immediately instead of pasting; and never populate a param field "for later" — add fields when a reader exists.

**How to report.** Complexity findings are correctness findings — severities follow the guide (a real bug is 🔴 even if the style checklist is spotless). For each, give the failure scenario in one line (concrete inputs → wrong outcome), not just the rule. If you deep-reviewed a complex function and found it correct, say so in one ✅ line naming what you verified (e.g. "createOrder: side effect correctly dispatched post-commit; lock order consistent") — this is the one place restating a passing check is worth it, because it tells the user the hard path was actually examined.

### Test Conventions (`*_test.go`)
- External test package: `package xxx_test`
- `tu.Setup()` + `defer tc.Teardown()` at the top of every `TestXxx`
- `t.Parallel()` as the first call inside `TestXxx`
- Subtests via `t.Run("case name", func(t *testing.T) { ... })`; standard names: `"success"`, `"ok"`, `"not found"`, `"validation"`, `"duplicate"`, `"expired"`
- `require` for fatal checks (setup, seed `NoError`); `assert` for value checks
- Sentinel errors via `assert.ErrorIs(t, err, pkg.ErrXxx)`
- Unordered slices → `assert.ElementsMatch`; time → `assert.WithinDuration`
- `decimal.Decimal` comparison: `assert.True(t, expected.Equal(got), ...)` — not `assert.Equal`
- Seed via raw SQL `pgctx.Exec` with `on conflict ... do nothing`
- Fixtures seeded in `pkg/tu/tu.go`'s setup method — not inline in individual test files
- Fakes for external services in `pkg/tu/` with `var _ Interface = fakeImpl{}` compile-time check; unused methods `panic("implement me")`
- Redis isolated via a unique key prefix per test, set up in `tu.Setup()`; teardown deletes keys by that prefix
- Payment gateways / HTTP clients: mock structs defined in the test file using a functional-options pattern (`func setTestXxx(t *testing.T, ctx context.Context, opts ...func(*mockXxx)) context.Context`)
- Mocks injected via context setters (`payment.SetGateway`)
- Concurrency tests use `sync.WaitGroup` + buffered channels: spawn goroutines, collect results/errors in channels, `wg.Wait()`, then assert both success count and error count
- Auth via `auth.WithAccountID(ctx, ...)` / `auth.NewContext(ctx, ...)`
- Test helpers at bottom of file, marked `t.Helper()`
- No `t.Skip` — tests must fail loudly when infra is missing

## Output Format

```
Scope: <staged | unstaged | PR #N | branch vs main> — <N Go files>

🔴 Must Fix
  - file.go:42 — <rule violated>. <one-line fix>.

🟡 Should Fix
  - file.go:88 — <rule violated>. <one-line fix>.

🟢 Nits
  - file.go:120 — <minor convention drift>.

✅ Looks Good
  - <one-line note on what was done well, only if there's something genuinely notable>
```

Severity guide:
- 🔴 **Must Fix** — bugs, security, incorrect SQL, wrong decimal usage, missing `Valid()`, ignored errors, zero-caller dead helpers, **Complexity-Pass correctness defects** (side effect inside tx, lost update, oversell via advisory-only check, TOCTOU, non-idempotent retried task, raw driver error to client, mixed-domain file past ~1200 lines)
- 🟡 **Should Fix** — naming, omitempty present, pgstmt patterns, test conventions, redundant/narrating comments, single-use helper extraction, divergent-but-not-yet-charged duplicate computations, mixed-domain file 600–1200 lines
- 🟢 **Nit** — style drift, ordering, helper extraction calls

If no Go files changed, say so in one line and stop. Don't invent findings.

**✅ is capped at two lines and is optional** — it exists for something genuinely notable (a
tricky invariant the code got right), not to reassure. Never list rules the code merely followed:
a passing rule earns silence, and a long ✅ block on a diff with real findings is how a review
reads as approval it did not give.

If no reviewable files changed, say so in one line and stop. Don't invent findings.

## Don'ts

- Don't auto-fix **unless** `--fix` or `--fix-all` was explicitly passed.
- Don't restate rules the code already follows. Silence = passing.
- Don't pad ✅ Looks Good. Two lines, or omit the section.
- Don't let dense prose read as care — verbose comments are a finding, not a mitigating factor.
- Don't review files outside the reviewable set (`*.go` + `schema/*.sql`), even if they're in the diff.
- Don't review non-Go files even if they're in the diff.
- Don't run `go test` or `go build` — pure static review.
- Don't apply `--fix` against PR scope — local edits only.

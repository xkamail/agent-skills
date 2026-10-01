# Reference: Project structure & file placement

Load this on **every** Go diff. Its job is placement: does each new/moved file live in the
right package for what it does? The inline checklist covers naming *within* a file; this covers
*which directory* the file belongs in. Flag a 🟡 when code lands in the wrong layer, a 🔴 when the
placement causes a real problem (e.g. business logic duplicated in a handler instead of `pkg/`).

## The repo map

Monorepo Go service, one codebase. Layers:

```
cmd/            entrypoints — one main per deployable
  api/            public API (embeds worker.Start in-process)
                  main.go creates the mux + arpc.Manager and calls `api.Mount(mux, am, db)`
                  + `callback.Mount(mux.Group("/callback"))`; it does NOT register routes itself
  backoffice/     admin API
  worker/         standalone asynq worker (same worker/ package)
  migrate/        applies schema/*.sql
  docgen/         generates openapi-api.json
  {provider}/
                  per-provider gamestore HTTP proxies (one external store each)

api/            public HTTP modules — one dir per feature (order, coupon, cart, …)
  handler.go      central mount: imports each module, registers `a.Handle("POST /order.list", …)`
  {feature}/      {feature}.go holds the arpc handlers + {feature}_test.go

backoffice/     admin HTTP modules — same shape as api/, admin-only features (iam, report, …)
  handler.go
  {feature}/

callback/       inbound webhooks — one dir per external source
  handler.go
  {source}/

worker/         asynq task handlers — flat files at root (worker.go, event_progress.go)

pkg/            shared libraries — one dir per feature or utility
  {feature}/      business logic reused by BOTH api/ and worker/ (order, coin, coupon, …)
  tu/             test setup, fakes, fixtures
  restful/        pagination, request id, error envelope
  httpclient/     shared HTTP client (15s timeout)
  payment/        gateway registry
  …

schema/         numbered SQL migrations (0001.sql, 0002.sql, …), applied in filename order
```

## Placement decision table

| The change is… | Goes in… |
|---|---|
| A new public endpoint | `api/{feature}/` (+ route in `api/handler.go`) |
| A new admin endpoint | `backoffice/{feature}/` (+ route in `backoffice/handler.go`) |
| An inbound webhook from a gateway/provider | `callback/{provider}/` (+ route in `callback/handler.go`) |
| A background/async task | `worker/` |
| Business logic used by **both** an HTTP handler and a worker task | `pkg/{feature}/` |
| A new external game-store proxy | `cmd/{provider}/` + its own `main.go` |
| A shared utility (client, helper, encoding) with 2+ callers across packages | `pkg/{util}/` |
| A DB schema change | new `schema/NNNN.sql` — `create table if not exists` for a new table, `alter table … add column if not exists` for a new column (idempotent; every file re-runs). See `references/schema.md` |

## The api/ ↔ pkg/ mirror (the highest-value structural check)

A feature typically appears in **both** places with the same name, split by responsibility:

- `api/{feature}/{feature}.go` — thin arpc handlers: `Valid()`, auth, param decode, call into pkg, shape the result.
- `pkg/{feature}/` — the real business logic (DB writes, state transitions, cross-cutting rules), so the worker can call the *same* code the API does.

Flag these:
- 🔴 **Duplicated logic** — a handler in `api/{feature}/` reimplements a state transition / DB write that already exists (or should exist) in `pkg/{feature}/`, instead of calling it. Same logic living in both an HTTP handler and a worker task is the classic case — it belongs in `pkg/`.
- 🟡 **Wrong layer** — non-trivial business logic (money math, inventory reservation, order state) written directly in the `api/` handler when a worker will eventually need it too → should be in `pkg/{feature}/`.
- 🟡 **Orphan pkg** — a new `pkg/{feature}/` with exactly one caller and no reuse across api/worker; if it's single-use it may belong inline in the handler (matches the "inline over single-use helpers" rule).

## Module internals (within `api/{feature}/` or `backoffice/{feature}/`)

- Every feature gets its own `{feature}/` subdirectory under `api/`, `backoffice/`, or `callback/` — even a single-handler feature (e.g. `health` with just `Ping`). Flag a 🟢 for a handler file sitting flat in the layer root (e.g. `api/health.go`) instead of `api/health/health.go`; this is not conditional on the feature growing a second handler.
- Handlers are exported funcs (`List`, `Get`, `Pay`) registered **centrally** in the layer's `handler.go` — a feature dir does **not** carry its own `handler.go`. Don't flag the absence of one; do flag a stray per-feature `handler.go` that duplicates the central registration.
- Route names follow `{feature}.{action}` and the action does not repeat the feature (`order.pay`, not `order.payOrder`) — enforced by the inline checklist; verify the `a.Handle(...)` line in `handler.go` matches.
- One domain per file (inline checklist). When a feature dir grows, split into per-domain files (`registration.go`, `bracket.go`, …), not one catch-all `{feature}.go`.

## Hard placement rules (each is a finding if violated)

- **No `internal/` package.** Flat `api/` + `backoffice/` + `callback/` + `worker/` + `pkg/` only.
- **No `/api/v1` prefix in handler code.** The version prefix is applied via `mux.Group` in `cmd/*/main.go`, never hard-coded in a module.
- **No new tenant branch/dir.** Tenant differentiation is env + deploy target, never a code fork or a per-tenant package.
- **Shared code goes to `pkg/`, not copied.** If a symbol is needed by two `cmd/` binaries or by both api and worker, it lives in `pkg/` — copying it into each is a 🔴.
- **Cross-provider proxy logic** is driven by `game_items.metadata` (jsonb), not hard-coded per provider inside shared packages.

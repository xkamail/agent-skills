# godoclive / OpenAPI docs

Load when the diff **adds or changes a handler `Params`/`Result` struct** (or a nested request/response type), or touches `cmd/docgen`, `.godoclive.yaml`, or the `/docs` serving in `cmd/api`. The OpenAPI spec is generated from Go source — there is **no hand-written spec** — so struct doc comments *are* the API docs, and a review must treat a missing/misplaced doc comment on a public field as a real defect, not cosmetics.

## How it's wired (setup)

`github.com/xkamail/godoclive` v0.6.0 (`pkg/godoclive`) parses the packages with an AST pass, extracts one `EndpointDef` per registered handler by reading its `Params`/`Result` structs, and emits OpenAPI 3.1.0.

- **`cmd/docgen/main.go`** — the generator. `godoclive.Analyze(".", "./...")` → `[]EndpointDef`; the slice is split into api vs backoffice by `strings.Contains(e.Package, "backoffice")`; each half goes through `godoclive.GenerateOpenAPI(..., WithTitle(...), WithAuth(...))` and is written to `openapi-api.json` / `openapi-backoffice.json`.
- **`.godoclive.yaml`** (repo root) — config merged into `Analyze`. Holds `exclude` glob patterns (`"<METHOD> <path>"`, e.g. `"* /callback/*"`, `"* /buy"`) that drop endpoints from the spec, and `auth:` (`header: Authorization`, `scheme: bearer`) applied to endpoints where auth is detected.
- **`cmd/api/main.go`** — serves docs: reads `openapi-api.json` at startup and renders it with `scalargo` (Scalar) at `GET /docs`. If the file is absent the route is simply not mounted.
- **Regenerate**: `go run ./cmd/docgen`. `openapi-api.json` / `openapi-backoffice.json` are **generated artifacts, not committed** — never hand-edit them; regenerate to preview.

`Analyze` reads `.godoclive.yaml` automatically (via `LoadConfig(dir)`); programmatic options (`WithMounts`, `WithAuth`, `WithTitle`, `WithVersion`) are merged on top. A new tenant/prefix split is done in `cmd/docgen`, not per-handler.

## How to bind / document a field

godoclive uses the **Go doc comment on the line(s) directly above a struct field** verbatim as that field's `description`.

```go
// Get returns a single order for the current account.   // ← documents the endpoint/schema
type GetResult struct {
	// จำนวนที่จ่ายจริง (ยอดสุทธิหลังหักส่วนลด)
	PaidAmount decimal.Decimal `json:"paidAmount"`         // ← preferred: above the field

	Status order.Status `json:"status"` // fallback only — trailing same-line comment
}
```

Rules godoclive applies:
- **Above-field comment → description** (preferred). A **trailing same-line comment is a fallback only** — flag fields that rely on it when the description matters.
- Document **request and response** structs — `Params`, `Result`, **and nested types** (list `Item`s, embedded summaries). Nested types are analyzed too; an undocumented nested response type is an undocumented part of the API.
- The **struct's own doc comment** (above `type Xxx struct`) documents the schema/endpoint.
- A comment containing `Deprecated:` marks the field deprecated in the spec.
- For **optional pointer fields, state null / zero-value meaning** in the comment (godoclive can't infer it).
- Endpoints matching a `.godoclive.yaml` `exclude` pattern won't appear — internal/callback/webhook routes are intentionally excluded there, not documented.

## Review checklist (godoclive-specific)

🟡 flag unless it breaks the build:
- New or changed **public field on a `Params`/`Result`/nested type with no doc comment** → undocumented API surface.
- Description-worthy field documented with a **trailing comment where an above-field comment is expected**.
- **Optional pointer field** whose comment doesn't state what null/zero means.
- New **nested response type** (list item, summary struct) left entirely undocumented.
- New endpoint or module that is internal/callback/webhook and should be added to `.godoclive.yaml` `exclude` (or, conversely, a public endpoint accidentally matched by an existing exclude glob).
- A **new tenant/package prefix** that the `cmd/docgen` api-vs-backoffice split (`strings.Contains(e.Package, "backoffice")`) doesn't account for → endpoints land in the wrong spec.

🔴:
- Hand-edited `openapi-api.json` / `openapi-backoffice.json` in the diff (generated artifacts — must come from `go run ./cmd/docgen`, and they aren't committed).

Don't flag: missing comments on unexported fields, `Valid()` methods, or fields already excluded from the spec.

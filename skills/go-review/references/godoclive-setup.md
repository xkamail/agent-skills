# Setup: godoclive OpenAPI docs

Load when invoked as `/go-review setup godoclive` (or asked to "set up API docs / docgen / godoclive" in a Go project). This is a **setup task, not a review** — it wires OpenAPI generation from handler `Params`/`Result` doc comments into the current project.

## What gets installed

`github.com/xkamail/godoclive` (use `@latest`; v0.6.0 at time of writing) AST-parses the module, extracts one `EndpointDef` per registered arpc handler from its `Params`/`Result` structs, and emits OpenAPI 3.1.0. `github.com/bdpiprava/scalar-go` (use `@latest`; v0.13.0 at time of writing) renders the spec as a Scalar docs page at `GET /docs`. There is **no hand-written spec** — struct doc comments *are* the API docs.

Pieces to create/modify:

1. `cmd/docgen/main.go` — the generator
2. `.godoclive.yaml` — excludes + auth config (repo root)
3. `/docs` route in `cmd/api/main.go` — **only** in the api binary
4. `.gitignore` — the generated `openapi*.json` files
5. `Makefile` — a `docs` target
6. `CLAUDE.md` — an "API documentation (OpenAPI)" section

## Steps

### 1. Dependencies

```sh
go get github.com/xkamail/godoclive@latest
go get github.com/bdpiprava/scalar-go@latest
```

### 2. `cmd/docgen/main.go`

**Single-spec** (one API surface):

```go
package main

import (
	"log"
	"os"

	"github.com/xkamail/godoclive/pkg/godoclive"
)

func main() {
	endpoints, err := godoclive.Analyze(".", "./...")
	if err != nil {
		log.Fatal(err)
	}

	spec, err := godoclive.GenerateOpenAPI(endpoints,
		godoclive.WithTitle("<Project> API"),
		godoclive.WithAuth(godoclive.AuthConfig{
			Header: "Authorization",
			Scheme: "bearer",
		}),
	)
	if err != nil {
		log.Fatal(err)
	}
	if err := os.WriteFile("openapi.json", spec, 0644); err != nil {
		log.Fatal(err)
	}
	log.Printf("generated api docs with %d endpoints", len(endpoints))
}
```

**Split-spec** (separate player API vs backoffice): split the `[]EndpointDef` by package before generating:

```go
	var apiEndpoints, backofficeEndpoints []godoclive.EndpointDef
	for _, e := range endpoints {
		if strings.Contains(e.Package, "backoffice") {
			backofficeEndpoints = append(backofficeEndpoints, e)
		} else {
			apiEndpoints = append(apiEndpoints, e)
		}
	}
```

then call `GenerateOpenAPI` once per slice with its own `WithTitle(...)`, writing `openapi-api.json` / `openapi-backoffice.json`. Pick split-spec when the repo has distinct `api/` and `backoffice/` handler trees (or separate binaries).

### 3. `.godoclive.yaml` (repo root)

`Analyze` auto-loads it; programmatic options (`WithTitle`, `WithAuth`, `WithVersion`, `WithMounts`) merge on top.

```yaml
exclude:
  - "* /"            # health/root
  - "* /callback/*"  # webhooks — internal, not documented
  - "* /webhook"
auth:
  header: "Authorization"
  scheme: "bearer"
```

Exclude patterns are `"<METHOD> <path>"` globs (`*` method matches all). Exclude every internal/callback/webhook route; keep the list minimal otherwise. Adjust `auth` if the project uses a different header (e.g. `x-admin-token`).

### 4. Serve `/docs` in `cmd/api/main.go`

Wire `/docs` **only into `cmd/api`** — do not add docs routes to other binaries (backoffice, worker, …), even in split-spec mode. After mounting the handlers, before the catch-all/NotFound handler:

```go
	if data, err := os.ReadFile("openapi.json"); err == nil {
		mux.HandleFunc("/docs", func(w http.ResponseWriter, r *http.Request) {
			html, err := scalargo.NewV2(
				scalargo.WithAuthenticationOpts(
					scalargo.WithHTTPBearerToken("your-bearer-token"),
				),
				scalargo.WithSpecBytes(data),
			)
			if err != nil {
				http.Error(w, err.Error(), 400)
				return
			}
			fmt.Fprint(w, html)
		})
	}
```

Import as `scalargo "github.com/bdpiprava/scalar-go"`. The `os.ReadFile` guard means the route simply isn't mounted when the spec file is absent (e.g. local run without docgen) — keep that behavior; don't `log.Fatal`.

Set `Cache-Control: no-store` on every `/docs` response (page and spec/assets alike) — browsers otherwise cache via `Last-Modified`/heuristics and keep showing stale docs after a redeploy:

```go
w.Header().Set("Cache-Control", "no-store")
``` For split-spec, `cmd/api` reads `openapi-api.json`; the other spec files are generated for reference/export only, not served.

Deployment note: the spec must exist in the image's working directory at runtime — add `RUN go run ./cmd/docgen` (or a `COPY` of the generated file) to the Dockerfile build stage if docs should be live in deploys.

**GOOS/GOARCH in Dockerfiles (must handle):** builder stages typically set `ENV GOOS=linux GOARCH=amd64` for cross-compiling the service binary. When the stage runs on a different build platform (e.g. native arm64 buildx on Apple Silicon), any step that must *execute* a Go tool breaks under those envs: `go run ./cmd/docgen` refuses to run a cross-compiled binary, and `go install <tool>@latest` drops the binary into `/go/bin/<goos>_<goarch>/` (→ "`/go/bin/tool: not found`") — and it couldn't execute anyway. Fix: unset both for the docgen step so the tool builds for the platform the stage is actually running on (spec/docs output is platform-independent):

```dockerfile
RUN env -u GOOS -u GOARCH go run ./cmd/docgen
# or, CLI flavor:
RUN env -u GOOS -u GOARCH go install github.com/xkamail/godoclive/cmd/godoclive@latest && \
    env -u GOOS -u GOARCH /go/bin/godoclive generate ./... --output ./docs --openapi ./docs/openapi.json
```

### 5. `.gitignore`

```
openapi.json
openapi-api.json
openapi-backoffice.json
```

(only the names the project actually generates). These are **generated artifacts, never committed, never hand-edited**.

### 6. `Makefile`

```make
docs:
	go run ./cmd/docgen
```

Add `docs` to `.PHONY`.

### 7. `CLAUDE.md` section

Add (adapting file names):

```markdown
## API documentation (OpenAPI)

`cmd/docgen` builds the OpenAPI spec with `godoclive` (`github.com/xkamail/godoclive`) by analyzing the `Params`/`Result` structs of every registered handler — there is no hand-written spec. To document the API, **write a Go doc comment on the line(s) directly above each struct field**; godoclive uses it verbatim as that field's `description`. Conventions:

- Doc comment above the field (preferred) → field description. A trailing same-line comment is used only as a fallback.
- **Go doc style: the comment starts with the field's own name.** `// CreatedAt is when the order was placed.` — not `// Time the order was placed.` or `// When the account was created.`. Full sentence, ends with a period, English only. Same for the struct comment (`// Item is …`). Being an OpenAPI `description` does not exempt it from godoc form.
- Never put a section-label comment inside a struct (`// Time`, `// Meta`) — godoclive reads it as the doc comment of the field below it and ships it as that field's description. Separate groups with a blank line.
- Document request **and** response structs (`Params`, `Result`, and nested types like list `Item`s). Keep comments short and factual; note null/zero-value meaning for optional pointer fields.
- A comment containing `Deprecated:` marks the field deprecated in the spec.
- The struct's own doc comment (above `type Xxx struct`) documents the schema/endpoint.
- Auth is `Authorization: Bearer <token>`, configured in `.godoclive.yaml`.
- `openapi.json` is a **generated artifact, not committed** (gitignored). `cmd/api` serves it at `/docs` (Scalar). Don't edit it by hand; regenerate with `make docs` (or `go run ./cmd/docgen`) to preview locally.
```

## Verify

1. `go build ./...` — everything compiles.
2. `go run ./cmd/docgen` — logs `generated api docs with N endpoints`, N > 0; spot-check the JSON contains a known endpoint and that a doc-commented field carries its `description`.
3. Excluded routes (callbacks/webhooks/health) do **not** appear in the spec.
4. Optionally run the api binary and open `/docs` — Scalar page renders.

## After setup

Doc comments on `Params`/`Result` fields are now API surface — the main checklist's godoclive item (flag undocumented exported `Params`/`Result` fields → 🟡) applies to this project from here on.

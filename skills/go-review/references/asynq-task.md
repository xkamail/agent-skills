# Reference: asynq queue/task pattern

Load this when the diff creates or changes a queue/task — a task type + payload in `pkg/task/`,
an enqueue call, or a `worker/` handler. It captures how a project wires asynq so a review can
flag drift from the established shape. Flag 🟡 for pattern drift, 🔴 when the drift breaks
enqueue/dispatch (wrong type string, unmarshal into the wrong struct, ignored enqueue error on a
must-run task).

## The three sides

Tasks live in **three** places. A new task touches all three; a review should check the diff hit
each one it needs.

1. `pkg/task/task.go` — the task **type enum**, the **payload struct**, and the **enqueue** path.
2. `worker/worker.go` — the **registration** (`mux.HandleFunc`) + the exported **processing func**.
3. Call sites in `api/` / `pkg/` — where the task is **enqueued**.

## 1. Task type + payload (`pkg/task/`)

- Task types are a `stringer` enum, not string literals:
  ```go
  //go:generate stringer -type=Type -linecomment
  type Type uint
  const (
      // TypePlaceOrderItem places an order item with the gamestore proxy for fulfillment.
      TypePlaceOrderItem Type = iota + 1 // TypePlaceOrderItem
      TypeEmail                          // TypeEmail
  )
  ```
  - 🟡 New task added as a raw `asynq.NewTask("my-task", …)` string instead of a `Type` const.
  - 🟡 `//go:generate stringer` / `type_string.go` not regenerated after adding a const (String() will be wrong → dispatch silently breaks). The linecomment must match the const name.
- Payload is a `XxxParams` struct with a doc comment `// XxxParams is the payload for TypeXxx tasks.`. JSON tags follow the normal checklist (no `omitempty` on required fields — but note the existing `EmailParams`/`EventProgressParams` use `omitempty` on genuinely-optional fields; match the local file).

## 2. Enqueue (`pkg/task/task.go`)

- Everything enqueues through the single `Create(ctx, t Type, in any, opts ...asynq.Option) error` helper — it marshals JSON, wraps in `asynq.NewTask(t.String(), b, asynq.TaskID(xid.New().String()))`, enqueues via `asynqctx.Client(ctx).EnqueueContext(ctx, tk, opts...)`, and logs with `slog.InfoContext`.
  - 🔴 A new task that builds `asynq.NewTask` + calls the client directly instead of going through `Create` — bypasses the `tu.InTest()` guard and the `xid` task-id dedup.
  - `Create` returns early (`return nil`) when `tu.InTest()` — enqueue is a **no-op under test**. Reviews of tests must call the processing func directly (see §4), not assert on the queue.
- Typed convenience wrappers (`SendEmail`, `EventProgress`) just call `Create(ctx, TypeXxx, params)`. Prefer adding one for a new task over open-coding `task.Create(ctx, task.TypeXxx, …)` at many call sites.

## 3. Registration + processing (`worker/worker.go`)

- Register in `Start`'s `asynq.NewServeMux()` keyed by the type string:
  ```go
  mux.HandleFunc(task.TypeXxx.String(), func(ctx context.Context, t *asynq.Task) error {
      var tk task.XxxParams
      if err := json.Unmarshal(t.Payload(), &tk); err != nil {
          return err
      }
      slog.InfoContext(ctx, "task: xxx", "params", tk)
      return ProcessXxx(ctx, tk)   // exported processing func
  })
  ```
  - 🔴 Handler unmarshals `t.Payload()` into the **wrong** `Params` struct, or registers under a type string that doesn't match the enqueue side.
  - 🔴 New `Type` const added but **not registered** in the mux (task will dead-letter). Check the diff added a `mux.HandleFunc`.
  - 🟡 Legacy/renamed type keys must keep the old `mux.HandleFunc("OldKey", handler) // legacy key for in-flight tasks` so tasks already queued under the old name still dispatch. Removing one without draining the queue is a 🔴.
- The real work is an **exported** func (`PlaceOrderItem`, `ProcessEventProgress`, `SendEmail`) so tests call it directly. The inline `mux.HandleFunc` closure stays thin: unmarshal → log → call the func.
- Cross-cutting rules still apply inside the processing func: `pgctx` for DB, `httpclient.Do` for HTTP, `slog.*Context`, decimal for money, `fmt.Errorf("...: %w", err)` for wrapping (internal errors, not user-facing arpc codes).
- `Concurrency` comes from `cfg.IntDefault("worker_concurrency", 4*runtime.NumCPU())`; errors are logged centrally in `ErrorHandler` with `asynq.GetRetryCount(ctx)`. Don't add per-handler retry/logging that duplicates this.

## 4. Test conventions (`worker/*_test.go`)

- `package worker_test`; `tu.Setup()` + `defer tc.Teardown()`; drive the flow, then **call the exported processing func directly**: `worker.PlaceOrderItem(ctx, task.PlaceOrderItemParams{id})`. No asynq server is started in tests.
- Because `task.Create` is a no-op under test, code under test that enqueues a follow-up task will **not** run it — the test must invoke that processing func itself if it needs the effect. Flag a test that asserts a downstream task's effect without calling its processing func.
- Otherwise the standard test checklist applies (`t.Parallel()` where the suite uses it, `require` for setup, `assert` for values, no `t.Skip`).

## Enqueue-error handling at call sites

- Must-run side effects check the error: `if err := task.EventProgress(ctx, …); err != nil { … }`.
- Fire-and-forget notifications may drop it as `_ = task.Create(ctx, task.TypeEmail, …)` — this is an accepted pattern. Don't flag `_ =` on a notification enqueue; do flag a silently-dropped enqueue whose delivery the caller depends on.

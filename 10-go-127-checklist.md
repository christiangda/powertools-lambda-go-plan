# 10. Go 1.26 and 1.27 feature checklist

Every change in the Go 1.26 and Go 1.27 release notes (plus the Go blog post and the
VictoriaMetrics write-up for 1.27) was checked against the plan. The table records what
is adopted, where, and what is deliberately not used. "Adopted" means the plan relies on
it; "allowed" means code may use it when it helps; "n/a" means it cannot matter here.

A reference implementation of this plan later walked the table against its code. Every
"adopted" row held, with the refinements recorded below (marked "in practice"); one row
changed status (`T.ArtifactDir`, now "not used"), and two policy rows were added (build
tags on test files, Go tip). "Allowed" rows that the implementation did not need are left
as allowed.

## Language

| Feature | Version | Status | Where |
| --- | --- | --- | --- |
| Generic methods (a method may declare its own type parameters; interfaces may not) | 1.27 | adopted | `(*Tracer).CaptureResult[T]` is a method; the package-level `tracer.CaptureResult[T]` stays for code that only has `ctx` (04 §4). No other generic methods: the library exports no interfaces, so the interface restriction never bites. |
| Generalised function type inference (assignment, conversion, composite literals) | 1.27 | adopted | `lambda.Start(powertools.Wrap(handle, opts))` infers `In`/`Out`; typed handler variables need no type arguments (01 §4.2). |
| Struct literal keys may be any field selector (promoted fields) | 1.27 | allowed | Table-test rows that embed fixture structs; not used in exported API. |
| `new(expr)` | 1.26 | adopted | Tri-state option fields whose nil means "environment or default": `tracer.Options{CaptureResponse: new(false)}`, `logger.BufferOptions{FlushOnErrorLog: new(false)}`, `powertools.Options{LogEvent: new(true)}` (01 §3, decisions 33 and 42). In practice it is also the idiom in tests (`metrics.Options{Disabled: new(false)}`) to pin a switch regardless of `POWERTOOLS_DEV`. For the same zero-value reason `BufferOptions.BufferAtLevel` is a `slog.Leveler`, where nil means DEBUG. |
| Self-referential generic constraints | 1.26 | n/a | |

## Standard library

| Feature | Version | Status | Where |
| --- | --- | --- | --- |
| `encoding/json/v2`, `encoding/json/jsontext` GA; v1 backed by v2 | 1.27 | adopted | Decision 23: the only JSON APIs in the repository (01 §2). `jsontext.Value` for raw payloads; `jsontext.AppendQuote` for strings; `json.MarshalWrite` for arbitrary values; `json.Deterministic(true)` only in tests because v2 does not sort map keys. Raw passthrough validates with `IsValid(jsontext.AllowDuplicateNames(true), jsontext.AllowInvalidUTF8(true))` so a Lambda payload is never rejected (02 §8). |
| `errors.AsType[T]` | 1.26 | adopted | Documented way to match typed errors from this library; tests use it instead of `errors.As`, and no `errors.As(` remains in the module (01 §6). In practice `errcheck` with `check-blank` flags its discarded first result, so the lint configuration excludes `errors.AsType` from that check. |
| `log/slog.NewMultiHandler` | 1.26 | adopted | Composition story for consumers who tee the Powertools handler with another (02 §3). |
| `log/slog.DiscardHandler` | 1.24 | adopted | Silences loggers a test does not assert on, including the X-Ray SDK's logger in `TestMain` (`xray.SetLogger(tracer.NewXRayLogger(slog.New(slog.DiscardHandler)))`). |
| `testing/synctest.Sleep`, `httptest.NewTestServer` | 1.27 | adopted | `tracer.HTTPClient` tests run against an in-memory server inside a synctest bubble, so timeouts are deterministic (04 §8); take `srv.Client()` before reading `srv.URL`. In practice `synctest` carries the whole deadline suite as well: the margin timer's arm, fire and stop, the flush content on margin, and the proof that a stopped timer never fires (return before the margin, `synctest.Sleep` past the deadline, assert no pre-timeout call) (06 §1). |
| `testing/cryptotest.SetGlobalRandom` | 1.26 | adopted | Deterministic X-Ray segment ids in golden tests (06 §2). In practice the comparisons still ignore times, `Dummy`, `parent_id` on embedded children, `cause.working_directory` and `stack`, which the SDK fills from sources other than the random reader. |
| `T.ArtifactDir` | 1.26 | not used | Planned for benchmark profiles and `benchstat` output. In practice the bench gate writes its inputs, report and profiles under `build/` from `make` targets and CI uploads them as job artifacts, so `go test -artifacts` adds nothing (05 §5, decision 39). |
| `B.Loop` | 1.24 | adopted | All benchmarks (05 §5). |
| `runtime/pprof` `goroutineleak` profile GA | 1.27 | adopted | After N wrapped invocations the leak test asserts the profile is empty and that no stack in it contains the module path (06 §1). In practice: call `WriteTo` before `Count` (the count is meaningful only after a write), and know that the profile reports only goroutines blocked forever, so the test also scans the full `goroutine` profile for module frames and uses `synctest` for the timer. |
| pprof labels in tracebacks (`tracebacklabels`) | 1.27 | adopted (opt-in) | `powertools.Options.PprofLabels` runs the handler under `pprof.Do` with `function_request_id` and `function_name`, so a panic traceback or `SIGQUIT` dump names the invocation (01 §4.2, §6). Off by default: one labelled context per invocation. |
| `strings.CutLast`, `bytes.CutLast` | 1.27 | allowed | Any last-separator parsing; none required by the design. |
| `context.AfterFunc`, `sync.WaitGroup.Go` | 1.21, 1.25 | adopted (guidance) | The patterns the docs and examples recommend for handler goroutines and cancellation hooks (01 §7.5); the library itself uses neither, and tests use `WaitGroup.Go` for concurrent callers. |
| `math/rand/v2 (*Rand).N` | 1.27 | allowed | Sampling decision uses a plain `rand.Float64`; `N` is not needed. |
| `uuid` package | 1.27 | n/a | X-Ray ids are 8 random bytes in hex, not UUIDs; EMF has no ids. |
| `hash/maphash.Hasher`, `math/big.Int.Divide`, `crypto/mldsa`, `simd` | 1.27 | n/a | |
| `net/http` body drain on close, HTTP/2 priorities, `MaxHeaderValueCount` | 1.27 | n/a | The library owns no HTTP server and only wraps the client transport through the X-Ray SDK. |
| `time` channels always unbuffered (`asynctimerchan` removed) | 1.27 | n/a | The only timer is the opt-in `DeadlineMargin` `time.AfterFunc` (01 §7 D6), which has no channel; the library never uses `time.Timer.C` or `time.After`. |
| `io.ReadAll` smaller final slice, `fmt.Errorf("x")` fewer allocations | 1.26 | allowed | Cold paths only. |
| `crypto/rand` random parameter ignored | 1.26 | n/a | The X-Ray SDK calls `rand.Read` directly. |

## Runtime and compiler

| Feature | Version | Status | Where |
| --- | --- | --- | --- |
| Size-specialised allocation (objects under 80 bytes up to 30 % cheaper) | 1.27 | adopted | Principle 7 in 05 §1: unavoidable per-record allocations are kept under 80 bytes (for example the `*XRayLogger` message wrapper, segment-name strings); large per-invocation objects (`State`) are allocated once. The performance history records the toolchain and `GOEXPERIMENT` defaults; no committed baseline is a gate input, since the bench gate compares the merge base and the head on one runner (05 §6). |
| Green Tea garbage collector default | 1.26 | noted | Benchmarks and Lambda measurements run with the default GC; the plan does not tune `GOGC`. |
| Goroutine labels in tracebacks for `go 1.27` modules | 1.27 | adopted | See `PprofLabels` above. |
| Known-bits analysis, loop-invariant code motion, switch lookup tables | 1.27 | noted | The unit-validation `switch` in metrics benefits from lookup-table compilation; no code shaped around it. |
| Type metadata relocation; `linkname` restrictions (`linknamestd`) | 1.27 | policy | `//go:linkname` is forbidden in this repository (08 §1). |
| Heap base randomisation, faster cgo | 1.26 | n/a | CGO is disabled for Lambda builds. |
| `GOEXPERIMENT=mapsplitgroup` | 1.27 | n/a | Experimental; not enabled. |

## Tooling

| Feature | Version | Status | Where |
| --- | --- | --- | --- |
| `go vet`: `%w` with `*E` where `E` implements `error` | 1.27 | adopted | Runs via `go test`; the error policy already forbids pointer-to-error wrapping (01 §6, 06 §4). |
| `go test` runs the `stdversion` vet check | 1.27 | adopted | Guarantees no symbol newer than the `go 1.27.0` directive is used (06 §4). |
| `go fix` modernizers (`atomictypes`, `embedlit`, `slicesbackward`, `unsafefuncs`, `waitgroupgo`, `//go:fix inline`) | 1.26, 1.27 | adopted | `go fix -diff ./...` is a `lint` gate; the cold-start flag is an `atomic.Bool` from the start (06 §4). In practice `go fix` must receive the test build tag, if the repository uses one, or it never sees the tests; and it can leave the tree not compiling, so its diff is read before committing. `embedlit` flattens nested embedded-struct literals, and re-nesting them by hand is undone on the next run. |
| `gofmt` alignment changes | 1.27 | adopted | `go.mod` carries a `toolchain go1.27.N` line and required jobs read `go-version-file: go.mod`, so `gofmt`, `vet` and `fix` give the same result everywhere (06 §4). A routine dependency-update pull request bumps the line; the pinned Go tools are re-installed after it. |
| `go mod tidy` merges `require` blocks (two blocks) | 1.27 | adopted | A CI check runs `go mod tidy -diff` on the module and on `examples`; there is no release target (releases come from commits, 06 §6). `go.mod` keeps exactly two `require` blocks, direct then indirect; re-splitting them by hand is undone by the next `tidy`. |
| `go doc pkg@version`, `go doc -ex` | 1.27 | adopted | `go doc -ex` lists runnable examples per package in the docs check (06 §4, §7). In practice: it ignores examples in files with a build tag and has no `-tags` flag, so example files stay untagged (decision 38); it does not type-check, so an untagged `go vet` compiles the examples; and it documents only its first package argument, so scripts loop over packages. |
| `go test -json` `OutputType` field | 1.27 | allowed | CI log parsing may use it. |
| `go tool trace -http` localhost only, `bzr` removed, response files | 1.27 | n/a | |
| `go mod init` lower default `go` version | 1.26 | noted | The directive is set explicitly to `go 1.27.0` in the module and in `examples`, both with the same `toolchain` line (a CI check compares them). |
| Build tags on test files | n/a | policy (optional) | If test files carry a tag such as `//go:build unit`, every tool passes it (`go test`, `go vet`, `go fix`, golangci-lint `run.build-tags`, and betteralign through `GOFLAGS`, since betteralign ignores its own `-tags`). Allocation-budget files carry `unit && !race` and run in a separate non-race step; `example_test.go` files carry no tag (row above); the real-AWS harness has its own tag (06 §1). |
| Go tip | n/a | policy | A nightly, non-required job builds tip from source and runs the suite without the `!race` budget files; a failure opens an issue and never blocks a merge (06 §5.3). |

## Deprecations and removals that could bite

| Item | Effect on the plan |
| --- | --- |
| `GOEXPERIMENT=nojsonv2` (temporary opt-out) | Unsupported build configuration; not in the CI matrix (01 §2). |
| `GOEXPERIMENT=nosizespecializedmalloc` (removed in 1.28) | Not used; budgets are measured with the default. |
| Removed `GODEBUG` settings (`asynctimerchan`, `gotypesalias`, TLS settings) | None used. `go.mod` carries no `godebug` lines. |
| Darwin requires macOS 13+ for the toolchain | Developer machines only; Lambda targets `linux/arm64` and `linux/amd64`. |

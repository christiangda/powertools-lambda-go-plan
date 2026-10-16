# 10. Go 1.26 and 1.27 feature checklist

Every change in the Go 1.26 and Go 1.27 release notes (plus the Go blog post and the
VictoriaMetrics write-up for 1.27) was checked against the plan. The table records what
is adopted, where, and what is deliberately not used. "Adopted" means the plan relies on
it; "allowed" means code may use it when it helps; "n/a" means it cannot matter here.

## Language

| Feature | Version | Status | Where |
| --- | --- | --- | --- |
| Generic methods (a method may declare its own type parameters; interfaces may not) | 1.27 | adopted | `(*Tracer).CaptureResult[T]` is a method; the package-level `tracer.CaptureResult[T]` stays for code that only has `ctx` (04 §4). No other generic methods: the library exports no interfaces, so the interface restriction never bites. |
| Generalised function type inference (assignment, conversion, composite literals) | 1.27 | adopted | `lambda.Start(powertools.Wrap(handle, opts))` infers `In`/`Out`; typed handler variables need no type arguments (01 §4.2). |
| Struct literal keys may be any field selector (promoted fields) | 1.27 | allowed | Table-test rows that embed fixture structs; not used in exported API. |
| `new(expr)` | 1.26 | adopted | Tri-state option fields: `tracer.Options{CaptureResponse: new(false)}` (01 §3). |
| Self-referential generic constraints | 1.26 | n/a | |

## Standard library

| Feature | Version | Status | Where |
| --- | --- | --- | --- |
| `encoding/json/v2`, `encoding/json/jsontext` GA; v1 backed by v2 | 1.27 | adopted | Decision 23: the only JSON APIs in the repository (01 §2). `jsontext.Value` for raw payloads; `jsontext.AppendQuote` for strings; `json.MarshalWrite` for arbitrary values; `json.Deterministic(true)` only in tests because v2 does not sort map keys. Raw passthrough validates with `IsValid(jsontext.AllowDuplicateNames(true), jsontext.AllowInvalidUTF8(true))` so a Lambda payload is never rejected (02 §8). |
| `errors.AsType[T]` | 1.26 | adopted | Documented way to match typed errors from this library; tests use it instead of `errors.As` (01 §6). |
| `log/slog.NewMultiHandler` | 1.26 | adopted | Composition story for consumers who tee the Powertools handler with another (02 §3). |
| `log/slog.DiscardHandler` | 1.24 | adopted | Silencing in tests (06 §3). |
| `testing/synctest.Sleep`, `httptest.NewTestServer` | 1.27 | adopted | `tracer.HTTPClient` tests run against an in-memory server inside a synctest bubble, so timeouts are deterministic (04 §8). |
| `testing/cryptotest.SetGlobalRandom` | 1.26 | adopted | Deterministic X-Ray segment ids in golden tests without ignoring the field (06 §2). |
| `T.ArtifactDir` | 1.26 | adopted | Benchmark profiles and `benchstat` outputs are written there in CI (05 §5). |
| `B.Loop` | 1.24 | adopted | All benchmarks (05 §5). |
| `runtime/pprof` `goroutineleak` profile GA | 1.27 | adopted | Lifecycle tests assert the profile is empty after N wrapped invocations: the library must never leak a goroutine (06 §1). |
| pprof labels in tracebacks (`tracebacklabels`) | 1.27 | adopted (opt-in) | `powertools.Options.PprofLabels` runs the handler under `pprof.Do` with `function_request_id` and `function_name`, so a panic traceback or `SIGQUIT` dump names the invocation (01 §4.2, §6). Off by default: one labelled context per invocation. |
| `strings.CutLast`, `bytes.CutLast` | 1.27 | allowed | Any last-separator parsing; none required by the design. |
| `context.AfterFunc`, `sync.WaitGroup.Go` | 1.21, 1.25 | adopted (guidance) | The patterns the docs and examples recommend for handler goroutines and cancellation hooks (01 §7.5). |
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
| Size-specialised allocation (objects under 80 bytes up to 30 % cheaper) | 1.27 | adopted | Principle 7 in 05 §1: unavoidable per-record allocations are kept under 80 bytes (for example the `*XRayLogger` message wrapper, segment-name strings); large per-invocation objects (`State`) are allocated once. Baselines record `GOEXPERIMENT` defaults. |
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
| `go fix` modernizers (`atomictypes`, `embedlit`, `slicesbackward`, `unsafefuncs`, `waitgroupgo`, `//go:fix inline`) | 1.26, 1.27 | adopted | `go fix -diff ./...` is a `lint` gate; the cold-start flag is an `atomic.Bool` from the start (06 §4). |
| `gofmt` alignment changes | 1.27 | adopted | CI pins the toolchain so `gofmt -l` is stable (06 §4). |
| `go mod tidy` merges `require` blocks (two blocks) | 1.27 | adopted | Release target runs `tidy` on the module and on `examples` (06 §6). |
| `go doc pkg@version`, `go doc -ex` | 1.27 | adopted | `go doc -ex` lists runnable examples per package in the docs job (06 §7). |
| `go test -json` `OutputType` field | 1.27 | allowed | CI log parsing may use it. |
| `go tool trace -http` localhost only, `bzr` removed, response files | 1.27 | n/a | |
| `go mod init` lower default `go` version | 1.26 | noted | The directive is set explicitly to `go 1.27.0` in the module and in `examples`. |

## Deprecations and removals that could bite

| Item | Effect on the plan |
| --- | --- |
| `GOEXPERIMENT=nojsonv2` (temporary opt-out) | Unsupported build configuration; not in the CI matrix (01 §2). |
| `GOEXPERIMENT=nosizespecializedmalloc` (removed in 1.28) | Not used; budgets are measured with the default. |
| Removed `GODEBUG` settings (`asynctimerchan`, `gotypesalias`, TLS settings) | None used. `go.mod` carries no `godebug` lines. |
| Darwin requires macOS 13+ for the toolchain | Developer machines only; Lambda targets `linux/arm64` and `linux/amd64`. |

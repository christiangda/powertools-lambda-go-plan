# 08. Style and SOLID

## 1. Google Go style guide, applied

| Rule (guide / decisions / best practices) | How this codebase applies it |
| --- | --- |
| `gofmt`, MixedCaps, no underscores | enforced in CI; JSON keys with underscores are string literals, never identifiers |
| Package names short, lowercase, no `util`/`common`/`helper` | `logger`, `metrics`, `tracer`, `powertools`; shared code lives under `internal/` (`invocation`, `lambdaenv`, `jsonenc`, `bufpool`, `syncwriter`), invisible to consumers by the ordinary Go rule |
| Avoid repetition: `widget.New`, not `widget.NewWidget` | `logger.New`, `metrics.New`, `tracer.New`; `metrics.Count`, not `metrics.UnitCount` |
| No `Get` prefix on getters | `CorrelationID(ctx)`, `TraceID(ctx)`, `Header(ctx)`, `Keys(ctx)`, `Handler.Level()` |
| Type suffix for variants | `Add` / `AddHighResolution`; `AddMetadata` / `AddMetadataToNamespace` |
| `context.Context` first, never stored in structs | every per-invocation entry point takes `ctx` first; `State` is stored *in* ctx, no struct stores ctx; `Segment` holds no ctx |
| Option structs when most callers pass options; no ctx in options | `Options` structs everywhere; zero value usable; no functional options. An option whose default is "on", or that overrides an environment variable in both directions, is a tri-state `*bool` (nil = environment or default), set with `new(false)`, never a `bool` documented as "default true": `tracer.Options.Enabled`, `CaptureResponse`, `CaptureError`, `logger.Options.Pretty`, `logger.BufferOptions.FlushOnErrorLog`, `metrics.Options.Disabled`, `powertools.Options.LogEvent` (§4). A level option whose default is not the zero level is a `slog.Leveler` (nil = default): `logger.BufferOptions.BufferAtLevel` (nil means DEBUG) |
| Accept interfaces, return concrete types; consumer-defined interfaces | we accept `io.Writer`, `slog.Leveler`, `slog.Handler` and return `*Handler`, `*Metrics`, `*Batch`, `*Tracer`, `*xray.Segment`, `*slog.Logger`, `*XRayLogger`; the library exports **no** interface type. Where we implement an interface it is the producer's own protocol (`slog.Handler`, `xraylog.Logger` for the adapter, and, in tests only, the X-Ray SDK's `xray.Emitter` for a recorder that counts emissions; the tracer's real test seam is a per-test UDP listener, 06 §1) |
| Errors: return, do not panic; sentinel values with `errors.Is`; lowercase messages; `%w` only when part of the contract | 01 §6 |
| Panics only for truly unreachable code; `Must*` only at init | no `Must*` in v1; the only `panic` is the re-panic in the wrapper |
| Global state with extreme scrutiny; provide instances, no hidden globals | no default instances; only the cold-start atomic, the pools and one shared stdout writer (used when a utility's `Writer` is nil, so log lines and EMF blobs never interleave) are package-level (01 §5). Environment values are read by constructors and cached in the instance, not snapshotted at init, so tests can `t.Setenv` before constructing. The library never mutates X-Ray SDK globals: consumers call `xray.SetLogger(tracer.NewXRayLogger(lg))` in `main` |
| Goroutine lifetimes obvious; prefer synchronous | no goroutines, timers or channels in the library's own code, except the opt-in `DeadlineMargin` `time.AfterFunc`, held in an unexported struct local to `begin`/`end` (not in `State`), whose lifetime ends in `end()` (05 §1, 01 §7 D6) |
| Document concurrency safety, cleanup, sentinel errors | every exported type's doc states "safe for concurrent use" or the locking rule; `Close`/`Flush` ownership documented |
| Receiver names short and consistent | `h *Handler`, `m *Metrics`, `b *Batch`, `t *Tracer` |
| Zero-value usability | `Options{}` works; a nil `*Batch` is a no-op (documented). `*xray.Segment` methods are **not** nil-safe (only `Close` is): every function that returns a segment documents "nil when not traced; check before calling methods", and the nil-safe path is the package helpers `tracer.AddAnnotation(ctx, …)`, `tracer.AddMetadata(ctx, …)` and `tracer.InstrumentAWS` (decision 31) |
| Nil vs empty slices not distinguished in APIs | `Keys(ctx)` returns a possibly nil slice; callers use `len` |
| Tests: `testing` only, `cmp.Diff`, `got/want`, table-driven, `t.Helper` | 06 §2 |
| Generics only where they pay | `Wrap[In, Out]` and `CaptureResult[T]` (package function and, with Go 1.27 generic methods, `(*Tracer).CaptureResult[T]`): they exist to keep the consumer's types; nothing else is generic |
| No `//go:linkname`, no `unsafe` in library code | Go 1.27 restricts linkname access and relocated type metadata; nothing in the design needs either |
| Doc comments on all exported names, full sentences | CI doc-coverage rule |
| Imports grouped: stdlib, then others | `goimports` with `-local github.com/<owner>` |
| Least mechanism: standard library first | JSON through `encoding/json/v2` and `jsontext` only (decision 23); `log/slog` rather than a bespoke logger; no third-party assertion or mocking libraries |
| Keyed struct literals for types from other packages (optional for local types) | keyed literals for **all** struct literals, including local ones: it is the reason `betteralign` can safely reorder hot structs (05 §4) and it keeps table-test rows readable |

## 2. SOLID mapping

| Principle | Where it shows |
| --- | --- |
| **Single responsibility** | one package per concern; inside packages: encoder (`jsonenc`), writer (`syncwriter`), lifecycle (root), state (`invocation`) are separate; the slog handler formats, it does not decide sampling (the wrapper does) nor buffer policy (options do) |
| **Open/closed** | behaviour extends through option-struct fields, `ReplaceAttr`, `slog` composition (`NewMultiHandler`, `With`), new modules for new integrations; no existing call site changes when a field is added |
| **Liskov substitution** | `*logger.Handler` is a drop-in `slog.Handler` proven by `slogtest`; `*tracer.XRayLogger` is a drop-in `xraylog.Logger`; `WrapHandler` returns a `lambda.Handler` usable anywhere one is accepted. Its input is narrower by contract: only hand-written `lambda.Handler` implementations that own the bytes they return, never `lambda.NewHandler(fn)`, which returns a pooled buffer and races under `MaxConcurrency > 1`; every other handler shape goes through `Wrap` with a one-line closure (decision 29). The tracer returns the SDK's own `*xray.Segment`, so every SDK function accepts it |
| **Interface segregation** | the library consumes the narrowest standard interfaces (`io.Writer`, not `*os.File`; `slog.Leveler`, not `*slog.LevelVar`) and exports none, so consumers can declare the one-method interface they actually use (`type metricRecorder interface{ Add(string, metrics.Unit, float64) error }`) and satisfy it with `*metrics.Batch` |
| **Dependency inversion** | hot paths depend on abstractions owned by the standard library; the X-Ray SDK is imported only by the `tracer` package's production code, and `logger`, `metrics` and `internal/...` never reach it or another utility, even transitively (both enforced by the `deps` CI job, 06 §4); the wrapper depends on the utilities only through their exported lifecycle API, frozen in Phase 1 (`(*Metrics).NewBatch`, `(*Metrics).EmitColdStart`, `(*Batch).Flush`, `(*Batch).FlushSnapshot`, `(*logger.Handler).SampleRate`, `logger.FlushBuffer`, `logger.FromContext`, `(*Tracer).Start`, and the end-of-handler helper `(*Tracer).EndHandler`), the same calls consumers use without the wrapper |

## 3. API review checklist (before `v1.0.0`)

- [ ] Every exported identifier has a doc comment that starts with its name.
- [ ] No exported interface types.
- [ ] No exported function takes more than four parameters; otherwise an options struct.
- [ ] No exported function panics on input; all validation returns `error`.
- [ ] Every constructor's option struct zero value is usable and documented as such; no `bool` option is documented as "default true" (use `*bool`), and every option backed by an environment variable can switch it both on and off.
- [ ] `context.Context` is the first parameter wherever present and is never stored.
- [ ] Every mutable exported type documents its concurrency contract.
- [ ] Every function that returns `*xray.Segment` documents "nil when not traced; check before calling methods", and nil-safe package helpers cover the common calls.
- [ ] The `WrapHandler` doc states that it accepts only hand-written handlers that own their returned bytes, and that `lambda.NewHandler(fn)` is not supported.
- [ ] Every sentinel error is exported and documented with the function(s) that return it.
- [ ] Names audited against the "no repetition" rule and the "no Get" rule.
- [ ] Runnable examples exist for each utility's primary workflow, and `go doc -ex` lists them per package.
- [ ] No exported type (with its constants) that no exported function or method takes or returns: consumers could not use it, and `v1.0.0` would freeze it.
- [ ] No leftover stub sentence from the API-freeze phase in any doc comment.
- [ ] The parity matrices in 02, 03, 04 are reflected in `docs/COMPATIBILITY.md`.
- [ ] `apidiff` (golang.org/x/exp/cmd/apidiff) between the Phase 1 API-freeze tag and the head shows only intended changes, each mapped to a decision.
- [ ] `README.md` states the versioning rule: v0 versions may be pinned without a compatibility promise, install examples use `@latest`, and after `v1.0.0` an incompatible change ships only under a `/v2` module path.

Every item is answered with evidence (a command and its output, or a file and line), not
by reference to the plan, and the answers are kept in a review file next to the design so
they survive the pull request.

## 4. API review outcome (lessons from a reference implementation)

A reference implementation ran the checklist above against its code before cutting
`v1.0.0`. Most items passed as designed; the findings below are the ones a new
implementation should expect, and each is a decision in 09 (42 to 44).

| Finding | Change | Rule it illustrates |
| --- | --- | --- |
| `powertools.Options.LogEvent bool` could switch event logging on, but `false` could not switch a truthy `POWERTOOLS_LOGGER_LOG_EVENT` off. It was the only environment-backed option that was not tri-state | `LogEvent *bool`: nil means the variable, `new(true)` and `new(false)` win over it. Consumers migrate `LogEvent: true` to `LogEvent: new(true)` | tri-state options as `*bool`, set with `new(expr)` (Go 1.26) |
| `metrics.Resolution` with the constants `Standard` and `High` was exported, but no exported function took or returned the type: the resolution is chosen by the method (`Add`, `AddHighResolution`, `Single.HighResolution`). `metrics.High` also reads poorly without its type | type and constants unexported; the EMF output is unchanged. A later `AddWithResolution` style API can export a type again without breaking anything | remove exported identifiers nobody can use; the "type suffix for variants" rule makes the type redundant |
| `metrics.New` returns `(*Metrics, error)` while `logger.New` and `tracer.New` do not | kept: an invalid namespace or dimension would make every later blob invalid, so metrics validates at construction; the logger has a safe fallback for every bad input (one WARN line), and the tracer has nothing that can fail | asymmetry is fine when each signature follows from what can fail |
| The end-of-handler helper (`tracer.HandlerEnd`, `(*Tracer).EndHandler`) is exported although only the wrapper calls it | kept, documented as wrapper plumbing that consumers of `powertools.Wrap` never call: it takes an `*xray.Segment`, and `internal/...` may never reach the X-Ray SDK, so it cannot move there | the import rules win over a smaller surface; say so in the doc comment |
| `apidiff` reported `logger.Handler`, `metrics.Batch` and `metrics.Single` as "old is comparable, new is not" | kept: the Phase 1 stubs had only comparable fields, the real bodies hold slices, maps and functions; the library hands out only pointers, which stay comparable, and no doc promised comparing values | an expected consequence of stubs-then-bodies; read every `apidiff` line, do not suppress it |
| `(*XRayLogger).Log(xraylog.LogLevel, fmt.Stringer)` takes no context | kept: the signature is fixed by the SDK's `xraylog.Logger` interface | implement the producer's interface, return the concrete type (`NewXRayLogger` returns `*XRayLogger`) |
| No exported interface type, no function with more than four parameters (the maximum is `logger.Log`, which mirrors `slog.Logger.Log`), no panic on input (the only `panic` is the wrapper's re-panic) | none | the checklist holds when the rules are applied from Phase 1 |

The changes ship during v0 as `feat` and `fix` commits, so the last v0 release is
API-identical to `v1.0.0`, and the release commit only declares the API stable.

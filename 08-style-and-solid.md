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
| **Interface segregation** | the library consumes the narrowest standard interfaces (`io.Writer`, not `*os.File`; `slog.Leveler`, not `*slog.LevelVar`) and exports none, so consumers can declare the one-method interface they need (`type metricRecorder interface{ Add(string, metrics.Unit, float64) error }`) and satisfy it with `*metrics.Batch` |
| **Dependency inversion** | hot paths depend on abstractions owned by the standard library; the X-Ray SDK is imported only by the `tracer` package's production code, and `logger`, `metrics` and `internal/...` never reach it or another utility, even transitively (both enforced by the `deps` CI job, 06 §4); the wrapper depends on the utilities only through their exported lifecycle API, frozen in Phase 1 (`(*Metrics).NewBatch`, `(*Metrics).EmitColdStart`, `(*Batch).Flush`, `(*Batch).FlushSnapshot`, `(*logger.Handler).SampleRate`, `logger.FlushBuffer`, `logger.FromContext`, `(*Tracer).Start`, and the end-of-handler helper `(*Tracer).EndHandler`), the same calls consumers use without the wrapper |

## 3. API review checklist (before `v1.0.0`)

The review runs in Phase 6, while the library is still in v0, so every change it asks for
ships as an ordinary `feat` or `fix` and the last v0 release is API-identical to
`v1.0.0`. The rules behind the items are in §4.

- [ ] Every exported identifier has a doc comment that starts with its name.
- [ ] No exported interface types; every constructor returns a concrete type.
- [ ] No exported function takes more than four parameters; otherwise an options struct.
- [ ] No exported function panics on input; all validation returns `error`.
- [ ] Every constructor's option struct zero value is usable and documented as such; no `bool` option is documented as "default true" (use `*bool`), and every option backed by an environment variable can switch it both on and off, `powertools.Options.LogEvent` included.
- [ ] `context.Context` is the first parameter wherever present and is never stored.
- [ ] Every mutable exported type documents its concurrency contract.
- [ ] Every function that returns `*xray.Segment` documents "nil when not traced; check before calling methods", and nil-safe package helpers cover the common calls.
- [ ] The `WrapHandler` doc states that it accepts only hand-written handlers that own their returned bytes, and that `lambda.NewHandler(fn)` is not supported.
- [ ] Every sentinel error is exported and documented with the function(s) that return it.
- [ ] Names audited against the "no repetition" rule and the "no Get" rule.
- [ ] Runnable examples exist for each utility's primary workflow, and `go doc -ex` lists them per package.
- [ ] No exported type (with its constants) that no exported function or method takes or returns: consumers could not use it, and `v1.0.0` would freeze it.
- [ ] Exported plumbing that only the wrapper calls says so in its doc comment, with the reason it cannot be internal.
- [ ] No leftover stub sentence from the API-freeze phase in any doc comment.
- [ ] The parity matrices in 02, 03, 04 are reflected in `docs/COMPATIBILITY.md`.
- [ ] `apidiff` (golang.org/x/exp/cmd/apidiff) between the Phase 1 API-freeze tag and the head shows only intended changes, each mapped to a decision; every line is read, none suppressed.
- [ ] `README.md` states the versioning rule: v0 versions may be pinned without a compatibility promise, install examples use `@latest`, and after `v1.0.0` an incompatible change ships only under a `/v2` module path.

Every item is answered with evidence (a command and its output, or a file and line), not
by reference to the plan, and the answers are kept in a review file next to the design so
they survive the pull request.

## 4. API design rules

The checklist above verifies these rules; they apply from the first stub in Phase 1, so
the review in Phase 6 confirms the API instead of reshaping it. Each rule is a decision
in 09.

| Rule | How the API applies it | Why |
| --- | --- | --- |
| **Tri-state options are `*bool`**, set with `new(expr)` (Go 1.26); nil means "environment variable, then default" | `powertools.Options.LogEvent *bool`: nil reads `POWERTOOLS_LOGGER_LOG_EVENT`, `new(true)` and `new(false)` win over it. The same for `tracer.Options.Enabled`, `CaptureResponse`, `CaptureError`, `logger.Options.Pretty`, `logger.BufferOptions.FlushOnErrorLog`, `metrics.Options.Disabled` (decisions 33 and 42) | a plain `bool` can switch an environment-backed feature on but never off, because `false` is indistinguishable from "not set"; one non-tri-state option among tri-state ones is a trap |
| **No exported identifier that no exported function or method takes or returns** | the metric storage resolution is chosen by the method (`Batch.Add`, `Batch.AddHighResolution`, `Single.HighResolution`), so there is no exported `metrics.Resolution` type and no `Standard` or `High` constants; the EMF output carries the resolution all the same (decision 43) | consumers cannot use such an identifier, yet `v1.0.0` would freeze it; the "type suffix for variants" rule already makes the type redundant, and a bare `metrics.High` reads poorly. A later `AddWithResolution` style API can export a type without breaking anything |
| **Concrete return types, no exported interfaces** | constructors return `*Handler`, `*Metrics`, `*Batch`, `*Tracer`, `*XRayLogger`, `*slog.Logger`; the tracer returns the SDK's own `*xray.Segment` | a concrete type can gain methods without breaking anyone; consumers declare the narrow interface they need (§5) |
| **Implement the producer's interface, return the concrete type** | `(*XRayLogger).Log(xraylog.LogLevel, fmt.Stringer)` takes no context because the SDK's `xraylog.Logger` fixes the signature; `NewXRayLogger` returns `*XRayLogger` | the interface belongs to the SDK; the library only satisfies it |
| **A constructor returns an error only if something can fail** | `metrics.New` returns `(*Metrics, error)`: an invalid namespace or dimension would make every later blob invalid, so it validates at construction. `logger.New` has a safe fallback for every bad input (one `WARN` line) and `tracer.New` has nothing that can fail, so neither returns an error (decision 44) | the asymmetry follows from what can fail; an always-nil error teaches callers to ignore it |
| **The import rules win over a smaller surface** | the end-of-handler helper (`(*Tracer).EndHandler` with its `tracer.HandlerEnd` argument) is exported although only the wrapper calls it: it takes an `*xray.Segment`, and `internal/...` may never reach the X-Ray SDK, so it cannot move there. Its doc comment says it is wrapper plumbing that consumers of `powertools.Wrap` never call | keeping the SDK out of `internal/...` protects every logger-only binary; one documented exported method is cheaper |
| **Hand out pointers; never promise value comparability** | `logger.Handler`, `metrics.Batch` and `metrics.Single` are used only through pointers, and no doc comment promises comparing values, so their fields may hold slices, maps and functions | the Phase 1 stubs have only comparable fields and the real bodies do not, so `apidiff` reports "old is comparable, new is not" for these types; that line is expected, read and accepted, never suppressed |
| **At most four parameters, no panic on input** | the widest exported function is `logger.Log(ctx, level, msg, args...)`, which mirrors `slog.Logger.Log`; validation returns `error`, and the only `panic` is the wrapper's re-panic of a handler panic | larger signatures become option structs; a library that panics on input takes the decision away from the caller |
| **One name per concept across packages** | `logger.FromContext`, `metrics.FromContext` and `tracer.FromContext` share a name; `RemainingTime` returns the maximum `time.Duration` without a deadline, so arithmetic on it works in local runs (decisions 35 and 44) | a consumer who learned one utility already knows the others |

## 5. Usage the rules enable

A consumer that wants to unit-test its own code declares the narrow interface it needs and
passes the library's concrete type, or a fake, without the library exporting anything for
testing:

```go
// metricRecorder is the one method the order service uses; *metrics.Batch satisfies it.
type metricRecorder interface {
	Add(name string, unit metrics.Unit, value float64) error
}

func recordOrder(m metricRecorder, total float64) error {
	if err := m.Add("Orders", metrics.Count, 1); err != nil {
		return err
	}

	return m.Add("OrderTotal", metrics.None, total)
}
```

Tri-state options read the same everywhere: leave the field nil to follow the
environment, or pin it in either direction.

```go
tr := tracer.New(tracer.Options{
	CaptureResponse: new(false), // never capture, whatever POWERTOOLS_TRACER_CAPTURE_RESPONSE says
})

h := powertools.Wrap(handle, powertools.Options{
	Tracer:   tr,
	LogEvent: new(true), // log every event, even when POWERTOOLS_LOGGER_LOG_EVENT is false
})
```

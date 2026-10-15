# 08. Style and SOLID

## 1. Google Go style guide, applied

| Rule (guide / decisions / best practices) | How this codebase applies it |
| --- | --- |
| `gofmt`, MixedCaps, no underscores | enforced in CI; JSON keys with underscores are string literals, never identifiers |
| Package names short, lowercase, no `util`/`common`/`helper` | `logger`, `metrics`, `tracer`, `powertools`; the shared module is named `internal` so the compiler enforces its visibility, and its packages are `invocation`, `lambdaenv`, `jsonenc`, `bufpool`, `syncwriter` |
| Avoid repetition: `widget.New`, not `widget.NewWidget` | `logger.New`, `metrics.New`, `tracer.New`; `metrics.Count`, not `metrics.UnitCount` |
| No `Get` prefix on getters | `CorrelationID(ctx)`, `TraceID(ctx)`, `Header(ctx)`, `Keys(ctx)`, `Handler.Level()` |
| Type suffix for variants | `Add` / `AddHighResolution`; `AddMetadata` / `AddMetadataToNamespace` |
| `context.Context` first, never stored in structs | every per-invocation entry point takes `ctx` first; `State` is stored *in* ctx, no struct stores ctx; `Segment` holds no ctx |
| Option structs when most callers pass options; no ctx in options | `Options` structs everywhere; zero value usable; no functional options |
| Accept interfaces, return concrete types; consumer-defined interfaces | we accept `io.Writer`, `slog.Leveler`, `slog.Handler` and return `*Handler`, `*Metrics`, `*Batch`, `*Tracer`, `*xray.Segment`, `*slog.Logger`, `*XRayLogger`; the library exports **no** interface type. Where we implement an interface it is the producer's own protocol (`slog.Handler`, the X-Ray SDK's `xray.Emitter` in tests and `xraylog.Logger` for the adapter) |
| Errors: return, do not panic; sentinel values with `errors.Is`; lowercase messages; `%w` only when part of the contract | 01 §6 |
| Panics only for truly unreachable code; `Must*` only at init | no `Must*` in v1; the only `panic` is the re-panic in the wrapper |
| Global state with extreme scrutiny; provide instances, no hidden globals | no default instances; only the cold-start atomic, env snapshot and pools are package-level (01 §5) |
| Goroutine lifetimes obvious; prefer synchronous | no goroutines in the library |
| Document concurrency safety, cleanup, sentinel errors | every exported type's doc states "safe for concurrent use" or the locking rule; `Close`/`Flush` ownership documented |
| Receiver names short and consistent | `h *Handler`, `m *Metrics`, `b *Batch`, `t *Tracer`, `s *Segment` |
| Zero-value usability | `Options{}` works; nil `*Batch` and `*Segment` are no-ops (documented) |
| Nil vs empty slices not distinguished in APIs | `Keys(ctx)` returns a possibly nil slice; callers use `len` |
| Tests: `testing` only, `cmp.Diff`, `got/want`, table-driven, `t.Helper` | 06 §2 |
| Generics only where they pay | `Wrap[In, Out]` and `CaptureResult[T]` (package function and, with Go 1.27 generic methods, `(*Tracer).CaptureResult[T]`): they exist to keep the consumer's types; nothing else is generic |
| No `//go:linkname`, no `unsafe` in library code | Go 1.27 restricts linkname access and relocated type metadata; nothing in the design needs either |
| Doc comments on all exported names, full sentences | CI doc-coverage rule |
| Imports grouped: stdlib, then others | `goimports` with `-local github.com/aizon-shared` |
| Least mechanism: standard library first | JSON through `encoding/json/v2` and `jsontext` only (decision 23); `log/slog` rather than a bespoke logger; no third-party assertion or mocking libraries |
| Keyed struct literals for types from other packages (optional for local types) | keyed literals for **all** struct literals, including local ones: it is the reason `betteralign` can safely reorder hot structs (05 §4) and it keeps table-test rows readable |

## 2. SOLID mapping

| Principle | Where it shows |
| --- | --- |
| **Single responsibility** | one package per concern; inside packages: encoder (`jsonenc`), writer (`syncwriter`), lifecycle (root), state (`invocation`) are separate; the slog handler formats, it does not decide sampling (the wrapper does) nor buffer policy (options do) |
| **Open/closed** | behaviour extends through option-struct fields, `ReplaceAttr`, `slog` composition (`NewMultiHandler`, `With`), new modules for new integrations; no existing call site changes when a field is added |
| **Liskov substitution** | `*logger.Handler` is a drop-in `slog.Handler` proven by `slogtest`; `*tracer.XRayLogger` is a drop-in `xraylog.Logger`; `WrapHandler` returns a `lambda.Handler` usable anywhere one is accepted; the tracer returns the SDK's own `*xray.Segment`, so every SDK function accepts it |
| **Interface segregation** | the library consumes the narrowest standard interfaces (`io.Writer`, not `*os.File`; `slog.Leveler`, not `*slog.LevelVar`) and exports none, so consumers can declare the one-method interface they actually use (`type metricRecorder interface{ Add(string, metrics.Unit, float64) error }`) and satisfy it with `*metrics.Batch` |
| **Dependency inversion** | hot paths depend on abstractions owned by the standard library; the X-Ray SDK appears only in the `tracer` module, and `internal` never imports a utility (enforced by the `deps` CI job); the wrapper depends on the utilities only through their exported lifecycle methods (`NewBatch`, `Flush`, `Start`, `Close`, `FlushBuffer`), the same ones consumers use without the wrapper |

## 3. API review checklist (before `v1.0.0`)

- [ ] Every exported identifier has a doc comment that starts with its name.
- [ ] No exported interface types.
- [ ] No exported function takes more than four parameters; otherwise an options struct.
- [ ] No exported function panics on input; all validation returns `error`.
- [ ] Every constructor's option struct zero value is usable and documented as such.
- [ ] `context.Context` is the first parameter wherever present and is never stored.
- [ ] Every mutable exported type documents its concurrency contract.
- [ ] Every sentinel error is exported and documented with the function(s) that return it.
- [ ] Names audited against the "no repetition" rule and the "no Get" rule.
- [ ] Runnable examples exist for each utility's primary workflow.
- [ ] The parity matrices in 02, 03, 04 are reflected in `docs/COMPATIBILITY.md`.
- [ ] `apidiff` (golang.org/x/exp/cmd/apidiff) run between `v0.x` tags shows only intended changes.

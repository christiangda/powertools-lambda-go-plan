# 09. Decision log

## Decisions taken, first round (internal review, 2026-10-15)

| # | Question | Decision | Where applied |
| --- | --- | --- | --- |
| 1 | Go floor | **Go 1.27** (`go 1.27.0` directive). `encoding/json/v2` and `jsontext` are used by `internal/jsonenc`. | 01 §2 |
| 2 | Log levels | **All six Lambda ALC levels as `slog` custom levels**: `TRACE` (-8), `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL` (12). Emitted names are exactly Lambda's, so filtering never downgrades a line. Powertools spellings `WARNING`, `CRITICAL`, `SILENT` are accepted when parsing. | 02 §2 |
| 3 | X-Ray | **Wrap `aws-xray-sdk-go/v2`** (AWS maintains it). Only the `tracer` package imports the SDK, so it is linked only into binaries that use tracing (see decision 25 for the module layout). | 04 |
| 4 | Independence of the utilities | **Superseded by decision 25.** Originally one Go module per utility plus a shared `internal` module and a root wrapper module. | 01 §2 |
| 5 | Wrapper shapes | **Both** `Wrap[In, Out]` and `WrapHandler`. | 01 §4.2 |
| 6 | Log keys | **Invocation-scoped** `AppendKeys(ctx, …)`; `log.With` for lexical scope; `PersistentAttrs` for process constants. | 02 §4 |
| 7 | Global instances | **None in v1.** Construct in `main`, pass or use `FromContext`. | 01 §5 |
| 8 | OpenTelemetry | **Out of scope**; the tracer stays small enough for a later bridge module. | 04 |
| 9 | Community port as reference | Behavioural contracts tied to the official Python and TypeScript editions and the AWS SDKs only; the community port `rambow-cloud/powertools-lambda-go` was not used as a reference. | README |
| 10 | Timestamps | **RFC 3339 only**, `Options.TimestampLayout` with a doc warning. | 02 §1 |
| 11 | Response capture | On by default, 60 kB guard, env switch. | 04 §5 |
| 12 | Segment finaliser | The SDK's `Close(err)`; no wrapper type. | 04 §4 |
| 13 | EMF schema validation | Test-only `jsonschema` dependency allowed. | 03 §8 |
| 14 | `location` key | Off by default (`Options.AddSource`). | 02 §3 |
| 16 | `metrics.Batch.Add` | Returns `error`. | 03 §3 |

Decision 15 concerned repository visibility and license and is omitted from this public copy.
| 17 | Response streaming | Unsupported in v1, documented. | README non-goals |
| 23 | JSON API | **`encoding/json/v2` and `jsontext` only**; v1 `encoding/json` is never imported. `jsontext.Value` replaces `json.RawMessage`. Verified interop with the X-Ray SDK's v1 marshal is a Phase 4 task. | 01 §2, 02, 03, 04, 05, 06 |

## Decisions taken, second round (internal review, 2026-10-15)

| # | Question | Decision | Where applied |
| --- | --- | --- | --- |
| 18 | Shared `internal` module | **Superseded by decision 25.** `internal/` is now an ordinary internal directory of the single module. | 01 §2 |
| 19 | `SILENT` level | **Implemented**: `logger.LevelSilent = slog.Level(16)`, a threshold only; `ParseLevel("SILENT")` maps to it. | 02 §2 |
| 20 | SDK logger adapter | **Yes**: `tracer.NewXRayLogger(*slog.Logger)` for `xray.SetLogger`. | 04 §4 |
| 21 | Lockstep tagging | **Superseded by decision 25.** With one module there is one tag; nothing to keep in lockstep. | 06 §6 |
| 22 | HTTP client helper | **Yes**: `tracer.HTTPClient(c)` honouring `POWERTOOLS_TRACER_CAPTURE_HTTPS_REQUESTS`, deferring to `xray.Client`. | 04 §7 |
| 24 | Deadlines, cancellation, freeze, shutdown (raised in the internal review) | `end()` never waits on `ctx`; nothing pending between invocations; per-invocation objects never pooled; closed-state behaviour for late goroutines; `RemainingTime`/`Deadline` helpers; opt-in `DeadlineMargin` pre-timeout flush as the only timer in the library; `OpenSegments` warning. Added after the internal review. | 01 §7, 02 §9, 03 §4, 04 §5.1, 05, 06 |

## Decisions taken, third round (internal review, 2026-10-16)

| # | Question | Decision | Where applied |
| --- | --- | --- | --- |
| 25 | Versioning of the components | **One module, one version.** The repository publishes a single Go module `github.com/aizon-shared/powertools-lambda-go` with a single tag `vX.Y.Z` for every package; no per-utility modules, no shared `internal` module, no lockstep tagging, no `go.work`. `examples/` is an untagged module with a `replace` to the parent. Accepted cost: the X-Ray SDK is in every consumer's module graph, never in logger-only binaries, enforced by the `deps` CI job. Supersedes 4, 18 and 21. | 01 §2, 04 §1, 05, 06 §5 and §6, 07, 08, 11, README |

No item is pending. New questions go to the pull request or to `docs/DECISIONS.md` once the
plan is archived (11 §3).

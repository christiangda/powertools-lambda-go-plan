# 01. Architecture

> **How to read the implementation notes.** A reference implementation of this plan was
> built and released as v1.0.0. Where it confirmed the plan, a short **Validated in
> implementation** note says so; where it forced a change, a **Changed after
> implementation** note gives the new rule and the reason. Decision numbers up to 25 are
> the reviewer decisions of the original plan; 26 to 44 were taken while implementing it
> (09). The text below is still a plan: it says what the library does and why, not how a
> particular code base is organised.

## 1. Runtime facts that constrain the design

Each rule below is a hard constraint on the code. The source is the document or file
that establishes it.

| # | Rule | Source |
| --- | --- | --- |
| R1 | Go Lambdas run on `provided.al2023`, an OS-only runtime. `AWS_EXECUTION_ENV` and `_X_AMZN_TRACE_ID` are **not defined** by the runtime for the `provided` family. "Am I in Lambda" is decided from `AWS_LAMBDA_FUNCTION_NAME` alone (set and non-empty), never from `AWS_EXECUTION_ENV`. | Lambda "Defined runtime environment variables" |
| R2 | `aws-lambda-go` puts the trace header in the invocation context under the plain string key `"x-amzn-trace-id"` on every invoke, possibly as `""`. It also calls `os.Setenv("_X_AMZN_TRACE_ID", …)` with the same value, but only when `lambdacontext.MaxConcurrency() == 1`. The X-Ray SDK reads only the context value. The context value is therefore the **only** source of the trace header: there is no `_X_AMZN_TRACE_ID` fallback and no `os.Getenv` per invocation, and a missing or empty value means "no trace" for the logger and the tracer alike. The key is a built-in string type, which staticcheck flags (SA1029); tests set it through one shared helper that carries the single lint exemption. | `lambda/invoke_loop.go` v1.55.1, `xray/lambda.go` v2.0.4 |
| R3 | `lambdacontext.FromContext(ctx)` yields `AwsRequestID`, `InvokedFunctionArn`, `TenantID`; `lambdacontext.FunctionName`, `MemoryLimitInMB`, `FunctionVersion`, `LogGroupName`, `LogStreamName` are package variables filled from `AWS_LAMBDA_*` at init. | `lambdacontext/context.go` |
| R4 | Under Lambda Managed Instances, `AWS_LAMBDA_MAX_CONCURRENCY > 1` and several invocations run concurrently in one process, sharing stdout and all globals. Any per-invocation state in a package variable is a bug. The first N invocations may all be "first". | Lambda env vars page, `lambdacontext.MaxConcurrency`, R2 |
| R5 | Lambda Advanced Logging Controls (ALC) filter JSON logs by the `level` key with values `TRACE, DEBUG, INFO, WARN, ERROR, FATAL`. "If the `level` value field is invalid or missing, Lambda assigns the log output the level INFO." The `timestamp` must be RFC 3339 or Lambda assigns its own and treats the line as INFO. ALC sets `AWS_LAMBDA_LOG_LEVEL` and `AWS_LAMBDA_LOG_FORMAT`. | Lambda "Log-level filtering" |
| R6 | `AWS_LAMBDA_INITIALIZATION_TYPE` is `on-demand`, `provisioned-concurrency`, `snap-start` or `lambda-managed-instances`. Powertools Python treats `provisioned-concurrency` as "never a cold start". `lambda-managed-instances` is treated like `on-demand`: the first invocation to win the `CompareAndSwap` is the cold start, which is what keeps concurrent first invocations correct. | Lambda env vars page, `tracing/tracer.py` |
| R7 | `aws-lambda-go` recovers handler panics in `callBytesHandlerFunc` and reports them to the Runtime API with a stack. A wrapper that recovers must re-panic so this behaviour is preserved. | `lambda/invoke_loop.go` |
| R8 | `aws-lambda-go` accepts twelve handler shapes: `func()`, `func(In)`, `func() error`, `func(In) error`, `func() (Out, error)`, `func(In) (Out, error)`, and the same six with a leading `context.Context`; plus any value implementing `lambda.Handler` (`Invoke(ctx, []byte) ([]byte, error)`). `lambda.NewHandler(fn)` turns a function into a `lambda.Handler`, but its `Invoke` returns bytes from a pooled buffer that it has already put back in the pool. A caller that reads them while another invocation runs in the same process (R4) races with that invocation's writes, and `bytes.Clone` right after `Invoke` does not fix it. `lambda.Start(fn)` is safe: the runtime sends the buffer before releasing it. | `lambda/entry.go`, `lambda/handler.go` v1.55.1 |
| R9 | X-Ray in Lambda: the service creates the function segment (the "facade"); user code may only send **subsegments** with `type: "subsegment"` and `parent_id` equal to the `Parent` field of the trace header, over UDP to `AWS_XRAY_DAEMON_ADDRESS` prefixed by `{"format": "json", "version": 1}\n`. Documents are capped at 64 kB. Segment names are at most 200 characters. The SDK sends each document as one datagram (about 65 507 bytes on Linux): a larger document fails to send and the whole handler trace is lost. The SDK streams completed children separately only above 20 subsegments, a count, not a size. | X-Ray "Sending trace data", "Segment documents", `xray/lambda.go`, `xray/segment.go`, `xray/default_emitter.go`, `xray/default_streaming_strategy.go` |
| R10 | EMF: at most 100 metric definitions per log event, at most 30 dimension keys per set (Powertools uses 29), at most 100 values per metric array, `Timestamp` in epoch milliseconds within 14 days past / 2 hours future, numeric values within ±2^360, namespace and names 1 to 255 printable ASCII. | EMF specification, Powertools `cloudwatch.py` constants |
| R11 | `aws-xray-sdk-go/v2` requires Go 1.26 and depends on `grpc`, `protobuf`, `fasthttp`, `aws-sdk-go-v2`, `route53`. The tracer wraps it (decision 3). Go links only imported packages, so a binary that imports neither `tracer` nor the root wrapper does not contain any of this; the SDK does appear in every consumer's module graph because the library is one module (decision 25). Only `tracer` imports the SDK **directly** in production code. The root wrapper reaches it transitively through `tracer` and never names an SDK type. Test files (`_test.go`) may import the SDK, because they are never linked into consumer binaries. CI checks both rules (06 §5). | `aws-xray-sdk-go` `go.mod` v2.0.4 |

> **Changed after implementation.**
> - **R1** (decision 35): the original rule also required `LAMBDA_TASK_ROOT`. Nothing
>   needed it, and a single variable is easier to reproduce in tests.
> - **R2** (decision 30): the original plan read `_X_AMZN_TRACE_ID` as a fallback when
>   max concurrency is 1. The SDK never reads it, so a fallback could make the logger
>   print a trace ID that the tracer does not use. One source, one answer.
> - **R8** (decision 29): the original plan let `WrapHandler` wrap
>   `lambda.NewHandler(fn)` and observe typed values through `handlertrace`. Reading
>   aws-lambda-go showed the pooled-buffer race above, so `WrapHandler` takes only
>   hand-written handlers that own their returned bytes, and `handlertrace` is not used
>   (§4.2).
> - **R9 and R11** gained the datagram and transitive-import facts; both shaped the
>   tracer's metadata budget (04 §5) and the import checks (06 §5).

## 2. Module and package layout

Decision 25: the library is **one Go module with one version**. A single tag `vX.Y.Z`
versions every package; there is one `go.mod`, no `go.work`, no per-utility tags and no
release script that rewrites `require` lines. Consumers import the packages they need
(`.../logger`, `.../metrics`, `.../tracer`, or the root `powertools` wrapper) from the
same module version.

> **Validated in implementation.** One module was never a problem during the whole
> build: the root wrapper and the three utilities changed together many times (the
> shared `State` contract, the API freeze, the API review) without any version
> juggling. Tags are best cut by a release tool from conventional commits (06 §6),
> with v0 minor releases while the API settles and one v1.0.0 after an API review
> (08 §3); after v1.0.0 a breaking change needs a `/v2` module path.

| Package | Imports | Linked into a binary when |
| --- | --- | --- |
| `powertools` (root) | `internal/...`, `logger`, `metrics`, `tracer`, `aws-lambda-go`; production code never imports the X-Ray SDK directly (its tracer wrapper tests may) | the wrapper is used (all three utilities come with it, and the X-Ray SDK through `tracer`, even when tracing is off) |
| `logger` | `internal/...`, standard library | imported |
| `metrics` | `internal/...`, standard library | imported |
| `tracer` | `internal/...`, `aws-xray-sdk-go/v2` (including `instrumentation/awsv2`), `smithy-go/middleware` | imported; the only package whose production code imports the X-Ray SDK |
| `internal/...` | standard library, `aws-lambda-go`; never the X-Ray SDK or a utility package, not even transitively | any of the above |
| `examples/...` | everything, plus SAM and AWS service clients | never: a separate, untagged module with a `replace` to the parent, so its dependencies stay out of the library |

Test-only dependencies: `github.com/google/go-cmp` (explicitly allowed by the Google
style guide) and a JSON schema validator for the EMF tests (decision 13). No assertion
libraries.

### Why one module

- **One version.** The root wrapper, `logger`, `metrics` and `tracer` share the
  `internal/invocation.State` contract. In one module there is nothing to mismatch: a
  consumer at `v0.3.0` has the `v0.3.0` of every package. Powertools Python ships the
  same way.
- **Standard mechanics only.** Tagging is `git tag vX.Y.Z` (done by the release tool);
  `go get …/powertools-lambda-go@vX.Y.Z` brings every package; `internal/` is the
  ordinary Go visibility rule, nothing clever.
- **The accepted cost.** The X-Ray SDK is a requirement of the module, so it appears in
  every consumer's module graph (`go.sum` entries, Dependabot alerts for logger-only
  users). It does **not** reach their binaries unless they import `tracer` or the
  wrapper: Go links only imported packages, and only `tracer` imports the SDK. The rule
  covers production code only: `_test.go` files are never linked into consumer
  binaries, so they may import the SDK. The `deps` CI job (06 §5) checks it in two
  parts: (a) no package except `tracer` lists the SDK in `go list`'s `.Imports`
  (`.TestImports` and `.XTestImports` are not checked); (b) `go list -deps` of the
  production code of `logger`, `metrics` and `internal/...` reaches neither the SDK nor
  a utility package other than itself (`logger` does not reach `metrics` or `tracer`,
  `internal/...` reaches no utility). `govulncheck` reports only reachable
  vulnerabilities, so logger-only users are not paged for SDK-only findings.
- **Reversible.** If a later need appears (separate owners, a utility with a heavy
  dependency that must stay optional), a package can be split into its own module at a
  major version boundary. The decision log records why it is not done now.

> **Measured in implementation.** A reference implementation built every example for
> `linux/arm64` with `-tags lambda.norpc -trimpath -ldflags='-s -w'`. A bare
> `lambda.Start` binary was about 6.0 MiB; adding `logger` and `metrics` without the
> wrapper added about 128 KiB; using `Wrap` with all three utilities added about
> 5.4 MiB (the X-Ray SDK and its tree, linked through `tracer` even when tracing is
> off); a full example that also used the AWS SDK for Go v2 with one service client
> added about 9.5 MiB. A size-sensitive function therefore uses `logger` and `metrics`
> without the wrapper. A CI size job that builds a bare baseline and the logger-only
> example in the same run, and fails when the difference exceeds a budget (1 MiB was
> used) or when `go version -m` shows the SDK in the logger-only binary, kept this
> property honest.

```mermaid
flowchart TB
  subgraph graph["every consumer's module graph (go.mod, go.sum)"]
    g["powertools-lambda-go vX.Y.Z<br/>aws-lambda-go<br/>aws-xray-sdk-go/v2 and its tree"]
  end
  subgraph binaries["what each binary actually links"]
    b1["imports logger only:<br/>logger, internal, aws-lambda-go"]
    b2["imports logger and metrics:<br/>+ metrics"]
    b3["imports the wrapper or tracer:<br/>+ tracer, X-Ray SDK and its tree"]
  end
  g --> b1
  g --> b2
  g --> b3
  classDef heavy fill:#fde2e2,stroke:#c0392b,color:#000;
  class b3 heavy;
```

### Dependency pitfalls found during implementation

> **Changed after implementation** (decision 35). Two `go.mod` surprises appeared the
> day the SDK was added; both are worth planning for:
>
> - **`github.com/aws/smithy-go` becomes a direct require.** `tracer.InstrumentAWS`
>   takes `*[]func(*middleware.Stack) error`, the type of `aws.Config.APIOptions`, so
>   `tracer` imports `smithy-go/middleware`. The module was already in the SDK's graph,
>   so the graph gains nothing, but `go mod tidy` moves it to the direct block.
> - **The SDK's pinned `golang.org/x/net` failed `govulncheck`.** The version the SDK
>   required carried HTTP/2 advisories that `govulncheck` reported as reachable through
>   the library's own packages, so a blocking
>   `govulncheck` job fails on the first commit that adds the SDK. The fix is an explicit
>   higher `golang.org/x/net` requirement in the library's `go.mod` (which also moves
>   `x/sys` and `x/text`), removed again once an SDK release requires a fixed version.
>   Plan a `govulncheck` run in the same change that adds the SDK.
>
> Two more rules kept the module graph calm: `govulncheck` covers the library module
> only, and the untagged `examples/` module is left to Dependabot, so an AWS SDK
> advisory in an example never blocks an unrelated library change (decision 37).

### Test layout and build tags

> **Changed after implementation** (decisions 26, 28, 37, 38). The plan did not say
> where tests live or how they are selected. What worked:
>
> - Every unit test file carries `//go:build unit`, and every tool is told about it
>   (`go test -tags=unit`, `go vet -tags=unit`, `go fix -tags=unit`, the linter's
>   `build-tags`). A tool that is not told silently sees no tests.
> - Two exceptions. The deployed-example harness carries `//go:build integration`, and
>   is proved to skip cleanly when its environment variables are unset. The
>   `example_test.go` files carry **no** tag, because `go doc -ex` does not list examples
>   from tagged files and has no `-tags` flag. A plain, untagged `go vet` over the public
>   packages then proves the examples compile on their own, so an example never depends
>   on a helper defined in a tagged test file.
> - Allocation budget tests additionally carry `!race` (`//go:build unit && !race`):
>   the race detector makes `sync.Pool` drop items, so allocation counts are only
>   meaningful in a separate non-race run (05 §6).
> - The root package's wrapper tests are split per utility (one test file each for the
>   logger, metrics and tracer interaction, all in the external test package), so the
>   three utility phases never edit the same file and can merge in parallel (07).
>   The tracer wrapper test may import the SDK, for example to point
>   `xray.ContextWithConfig` at a per-test UDP listener, because the import rule covers
>   production code only (R11).
> - Shared test helpers (the SA1029 trace-header helper of R2, fixtures) live in an
>   internal test-support package that only test files import, so every lint exemption
>   exists exactly once.

### Go version floor

Decision 1: the module declares `go 1.27.0` (and so does `examples/go.mod`). `encoding/json/v2` and
`encoding/json/jsontext` (GA in 1.27) back `internal/jsonenc`: `jsontext.AppendQuote`
for strings, `json.MarshalWrite` for arbitrary values, with no hand-rolled escaping.

> **Validated in implementation.** Pin the patch toolchain with a `toolchain go1.27.N`
> line and let required CI jobs read `go-version-file: go.mod`; run Go tip in a
> non-required nightly job. `go test` runs the `stdversion` vet check, so a standard
> library symbol newer than the `go` directive fails the tests, and tools built with an
> older Go fail on 1.27 sources: re-install them after every toolchain bump.

### JSON policy (decision 23)

`encoding/json/v2` and `encoding/json/jsontext` are the **only** JSON APIs used in
library and test code. The v1 `encoding/json` package is never imported by this
repository. Consequences:

- Raw JSON payloads (the payload and response bytes in `WrapHandler`, pre-marshalled
  metadata) are typed `jsontext.Value`, the v2 counterpart of `json.RawMessage`.
- Logger: `KindAny` and nested values are marshalled with `json.MarshalWrite` into a
  scratch region of the pooled buffer, with `jsontext.AllowInvalidUTF8(true)`. On error
  the region is truncated and the string `"!ERROR:<err>"` is written, as
  `slog.JSONHandler` does; a line is never dropped (02 §5).
- Metrics: each metadata value is marshalled with `json.Marshal` and then written with
  `enc.WriteValue`, never with `MarshalWrite` into the encoder's buffer. A value that
  fails to marshal is dropped with one `WARN` per batch, and `Flush` still succeeds
  (03 §5).
- Options. Encoding uses `jsontext.AllowInvalidUTF8(true)` for consumer values,
  `jsontext.WithIndent` for `POWERTOOLS_DEV`, and `json.Deterministic(true)` where output
  order matters in tests. Log lines may carry duplicate keys, as slog's do, so every
  decoder (golden and `slogtest` tests, the dev pretty-printer) passes
  `jsontext.AllowDuplicateNames(true)` and `jsontext.AllowInvalidUTF8(true)`.
- Third-party code keeps its own choice: `aws-lambda-go` decodes events with v1 and the
  X-Ray SDK marshals segments with v1. A `jsontext.Value` handed to the SDK as metadata
  is embedded as raw JSON, but v1 re-validates, compacts and HTML-escapes it (`<`, `>`
  and `&` grow to 6 bytes each). `json.RawMessage` behaves the same; it is equivalent,
  not a fallback. An invalid value makes the SDK drop the whole document, so `IsValid`
  is mandatory before any value reaches `AddMetadataToNamespace`, and the tracer's
  metadata budget counts the escaped size (04 §5). Tracer golden tests compare decoded
  values, not bytes.
- `GOEXPERIMENT=nojsonv2` builds are unsupported and the CI matrix does not include them.

> **Changed after implementation** (decisions 32, 33). The original text expected a
> `jsontext.Value` to reach the SDK "verbatim" and kept `json.RawMessage` as a fallback.
> A recorded segment showed the v1 re-validation and HTML escaping above, which matters
> for the 64 kB datagram limit (R9). The marshal-failure rules per utility were added
> because the plan did not say what happens when a consumer value cannot be encoded.

### Package responsibilities (single responsibility per package)

| Package | Responsibility | Exports |
| --- | --- | --- |
| `powertools` (root) | Handler wrapper, invocation lifecycle, cold-start detection, deadline helpers, context accessors | `Wrap`, `WrapHandler`, `Options`, `ColdStart(ctx)`, `RequestID(ctx)`, `FunctionARN(ctx)`, `RemainingTime(ctx)`, `Deadline(ctx)` |
| `internal/invocation` | The per-invocation `State` struct, its context key and the `LogBuffer` type. Shared by root and the three utilities. Imports only the standard library. The mutable part of `State` changes only through its locked, nil-safe methods. | `State` (methods such as `AppendAttrs`, `RemoveAttrs`, `ResetAttrs`, `AttrsSnapshot`, `SetCorrelationID`, `EnsureLogBuffer`), `TraceHeader`, `LogBuffer`, `NewContext`, `FromContext` |
| `internal/lambdaenv` | The only package that reads the environment: typed getters and truthy parsing, in-Lambda (R1) and SAM local detection, function metadata (aws-lambda-go's `lambdacontext` variables), the cold-start eligibility rule for `AWS_LAMBDA_INITIALIZATION_TYPE`, and zero-alloc trace header parsing (R2, §4.1). There is no init-time snapshot (§3). It holds its own `"x-amzn-trace-id"` key constant, because only `tracer` may import the SDK's `xray.LambdaTraceHeaderKey`. | `String`, `Bool`, `BoolOr`, `Float`, `ServiceName`, `InLambda`, `SAMLocal`, `ColdStartEligible`, `ParseTraceHeader`, `TraceHeaderFromContext`, `Function` |
| `internal/jsonenc` | Two tiers. Hot tier for the logger record path: append-style helpers over `jsontext.AppendQuote` and `strconv` (`AppendString`, `AppendValue` for `slog.Value`, `AppendAttr` for one `slog.Attr`, `AppendAny` with the `"!ERROR:<err>"` fallback). Cold tier for EMF blobs and metadata: a pooled `jsontext.Encoder` driven with `WriteToken`/`WriteValue`, which enforces balanced structure (11 §3 A). Both tiers copy `jsontext.Value` verbatim after `IsValid`, which is what makes raw-event logging and raw metadata zero re-encode in our own output. NaN and the infinities are written as the strings `"NaN"`, `"+Inf"` and `"-Inf"`. | `AppendString`, `AppendValue`, `AppendAttr`, `AppendAny`, `Encoder` (pooled) |
| `internal/jsonpath` | Dot path with optional double-quoted segments (`headers."x-amzn-trace-id"`), parsed once at wrap time and evaluated on raw `jsontext.Value` bytes for `CorrelationIDPath` (02 §8). Strings, numbers and booleans are accepted as IDs, always returned as an owned copy. | `Parse`, `Path`, `ScalarString`, `ErrInvalidPath` |
| `internal/bufpool` | `sync.Pool` of `[]byte` with size cap to avoid pinning huge buffers | `Get`, `Put` |
| `internal/syncwriter` | `io.Writer` wrapper: mutex, one `Write` per record, never partial. `Stdout` is the one process-wide instance over `os.Stdout`, used by every utility whose `Writer` is nil, so logger and metrics share one mutex. A writer supplied by the consumer is wrapped in a `syncwriter` owned by that utility instance and is not shared with other utilities. | `New(w io.Writer) *Writer`, `Stdout` |
| `internal/benchutil` | Benchmark fixtures shared by every package: a realistic invocation context and events (05 §5). Never imports the X-Ray SDK. | see 05 §5 |
| `internal/testsupport` | Test helpers imported only by test files, including the one helper that puts a trace header into `ctx` under the string key with the SA1029 exemption (R2). | see 06 §1 |
| `logger` | `slog.Handler` with Powertools output, six ALC levels, package-level `Trace…Fatal(ctx, …)`, sampling, buffering, invocation keys, correlation ID | see 02 |
| `metrics` | EMF batching, validation, serialization, cold-start metric (`EmitColdStart`) and single metrics | see 03 |
| `tracer` | Powertools semantics over `aws-xray-sdk-go/v2`: enable rules, handler segment and its end-of-handler helper, `Capture` helpers, nil-safe context helpers (`AddAnnotation`, `AddMetadata`), `HTTPClient` and `InstrumentAWS`, the per-invocation metadata budget | see 04 |

> **Changed after implementation** (decision 33). A separate `internal/env` package was
> folded into `internal/lambdaenv`, so exactly one package reads the environment.
> `internal/jsonpath`, `internal/benchutil` and `internal/testsupport` joined the
> layout. `internal/invocation` exposes locked methods instead of letting the logger
> take `State.Mu` itself, which kept the lock order in one place (§5).

## 3. Configuration model

### Option structs, not functional options

Every constructor takes a value option struct whose zero value is usable:

```go
l := logger.New(logger.Options{})          // service from env or "service_undefined"
m, err := metrics.New(metrics.Options{})   // namespace from env, else ErrNamespaceMissing at Flush
t := tracer.New(tracer.Options{})          // enable rules from env (R1, AWS_SAM_LOCAL, switches)
```

Reasons (Google best practices): callers almost always set a few fields, the literal is
self-documenting, new fields never break call sites, and `context.Context` is never part
of an option struct. Functional options are not used anywhere. Tri-state fields are
`*bool` and are set with Go 1.26's `new(expr)`: `tracer.Options{CaptureResponse: new(false)}`.

> **Changed after implementation** (decisions 33, 42). Two more rules keep the zero
> value usable:
>
> - **A boolean whose default is `true` is a `*bool`** (nil means true), for example
>   `logger.BufferOptions.FlushOnErrorLog`. A plain `bool` would make the zero value mean
>   "off" for a feature that should be on.
> - **Every option backed by an environment variable is a `*bool`**, so the caller can
>   switch the variable off as well as on. `powertools.Options.LogEvent` started as a
>   `bool`, which could only switch event logging on; it became a `*bool` in the API
>   review. Likewise `logger.BufferOptions.BufferAtLevel` is a `slog.Leveler` whose nil
>   means DEBUG.
>
> `metrics.New` is the only constructor that returns an error: an invalid namespace,
> service or default dimension would make every later blob invalid, so it fails early,
> whereas the logger falls back to its defaults with one warning and the tracer has
> nothing that can fail (decisions 36, 44).

Illustrative use of the intended API:

```go
lg := logger.New(logger.Options{
    Service:    "payment",
    Level:      slog.LevelDebug,                        // ignored when AWS_LAMBDA_LOG_LEVEL is set
    SampleRate: 0.1,                                    // per-invocation DEBUG sampling, needs the wrapper
    Buffer:     logger.BufferOptions{Enabled: true},    // FlushOnErrorLog nil means true
})
met, err := metrics.New(metrics.Options{
    Namespace:         "ServerlessAirline",
    Service:           "payment",                       // "service" is the first dimension of every blob
    DefaultDimensions: map[string]string{"environment": "prod"},
})
if err != nil {
    log.Fatal(err) // invalid namespace, service or dimension
}
trc := tracer.New(tracer.Options{Service: "payment", CaptureResponse: new(false)})
```

### Precedence

For every setting: **explicit option field > environment variable > default**, with one
exception mandated by ALC (R5):

```text
log level = AWS_LAMBDA_LOG_LEVEL  >  Options.Level  >  POWERTOOLS_LOG_LEVEL  >  INFO
```

```mermaid
flowchart TD
  start([effective log level]) --> alc{AWS_LAMBDA_LOG_LEVEL set?}
  alc -- yes --> useALC["use the ALC level"]
  alc -- no --> opt{Options.Level set?}
  opt -- yes --> useOpt["use Options.Level"]
  opt -- no --> env{POWERTOOLS_LOG_LEVEL set?}
  env -- yes --> useEnv["parse with ParseLevel"]
  env -- no --> info["INFO"]
  useALC --> warn{Powertools level configured<br/>and more verbose than ALC?}
  warn -- yes --> once["log one warning:<br/>lines below ALC are discarded by Lambda"]
  warn -- no --> done([done])
  once --> done
  useOpt --> done
  useEnv --> done
  info --> done
```

If `AWS_LAMBDA_LOG_LEVEL` is set and is stricter than the Powertools level, the handler
logs one warning at construction ("current log level is lower than ALC, lines below X
will be discarded") exactly as the Python and TypeScript editions do. When ALC is set, a
`*slog.LevelVar` passed as `Options.Level` is ignored as well.

Environment is read **once, in the constructors** (`logger.NewHandler`, `metrics.New`,
`tracer.New`, `powertools.Wrap` and `WrapHandler`), which cache what they need. Nothing
reads `os.Getenv` per invocation; the trace header comes only from the invocation
context (R2).

> **Changed after implementation** (decision 35). The original plan took an
> **environment snapshot at package init** in `internal/lambdaenv`. That made tests
> order-dependent: `t.Setenv` runs after init, so a test could not change what the
> library saw. The getters now read the environment when called, and only constructors
> call them. Consequences an implementer should keep: tests set variables with
> `t.Setenv` *before* constructing; a value read at construction never changes for that
> instance; the only process-wide copy of environment values is aws-lambda-go's own
> `lambdacontext` variables (R3). `tracer.HTTPClient` and `tracer.InstrumentAWS` are
> constructors in this sense too: they evaluate the enable rules when called, at init.

### Environment variables honoured

| Variable | Utility | Default | Notes |
| --- | --- | --- | --- |
| `POWERTOOLS_SERVICE_NAME` | all | `service_undefined` | `service` key, `service` dimension (the first dimension of every blob), `Service` annotation; each utility falls back to `service_undefined` independently |
| `POWERTOOLS_LOG_LEVEL` | logger | `INFO` | `TRACE DEBUG INFO WARN ERROR FATAL`, plus `WARNING`, `CRITICAL`, `SILENT` as aliases, case-insensitive |
| `AWS_LAMBDA_LOG_LEVEL` | logger | unset | ALC; wins over everything (R5) |
| `AWS_LAMBDA_LOG_FORMAT` | logger | unset | informational; output is always JSON |
| `POWERTOOLS_LOGGER_SAMPLE_RATE` | logger | `0` | `0.0` to `1.0`, clamped; the per-invocation decision is made by the wrapper, so sampling requires it |
| `POWERTOOLS_LOGGER_LOG_EVENT` | wrapper | `false` | used when `Options.LogEvent` is nil (truthy turns event logging on); `new(true)` or `new(false)` wins, so `new(false)` switches a truthy variable off |
| `POWERTOOLS_DEV` | logger, metrics | `false` | pretty-print logs, never inside Lambda (an indented line becomes several CloudWatch events); silence metrics unless `POWERTOOLS_METRICS_DISABLED=false` |
| `POWERTOOLS_METRICS_NAMESPACE` | metrics | unset | required by `Flush` |
| `POWERTOOLS_METRICS_FUNCTION_NAME` | metrics | unset | `function_name` dimension of `ColdStart`: `metrics.Options.FunctionName`, then this variable, then the argument of `EmitColdStart` |
| `POWERTOOLS_METRICS_DISABLED` | metrics | `false` | no output at all; arguments are still validated |
| `POWERTOOLS_TRACE_DISABLED` | tracer | `false` | Python name; also makes `tracer.HTTPClient` return an untraced client and `tracer.InstrumentAWS` a no-op. `xray.Client` and `awsv2` used directly ignore it |
| `POWERTOOLS_TRACE_ENABLED` | tracer | `true` | TypeScript name; both honoured, disabled wins |
| `POWERTOOLS_TRACER_CAPTURE_RESPONSE` | tracer | `true` | |
| `POWERTOOLS_TRACER_CAPTURE_ERROR` | tracer | `true` | |
| `POWERTOOLS_TRACER_CAPTURE_HTTPS_REQUESTS` | tracer | `true` | honoured by `tracer.HTTPClient` |
| `AWS_XRAY_DAEMON_ADDRESS` | tracer (SDK) | `127.0.0.1:2000` | read by the X-Ray SDK at init |
| `AWS_XRAY_CONTEXT_MISSING` | tracer (SDK) | `LOG_ERROR` in Lambda | read by the X-Ray SDK |
| `AWS_XRAY_SDK_DISABLED` | tracer | `false` | compared case-insensitively, as the SDK does; checked by the tracer and by the SDK; the only switch that also stops `xray.Client` and `awsv2` used directly |
| `AWS_SAM_LOCAL` | tracer | unset | disables tracing |
| `AWS_LAMBDA_FUNCTION_NAME`, `AWS_LAMBDA_FUNCTION_MEMORY_SIZE`, `AWS_LAMBDA_FUNCTION_VERSION` | all | | via `lambdacontext`; `AWS_LAMBDA_FUNCTION_NAME` also decides "in Lambda" (R1) |
| `AWS_LAMBDA_INITIALIZATION_TYPE` | wrapper | `on-demand` | cold-start rule (R6) |
| `AWS_LAMBDA_MAX_CONCURRENCY` | none | `1` | informational: read by aws-lambda-go only (R2, R4); the library never consults it, because it is correct at any concurrency |
| `_X_AMZN_TRACE_ID` | none | | never read: the trace header comes only from `ctx` (R2) |
| `TZ` | logger | `:UTC` | timestamps are UTC unless `Options.Location` is set |

> **Changed after implementation.** `AWS_LAMBDA_MAX_CONCURRENCY` and `_X_AMZN_TRACE_ID`
> were consumed by the original plan; both are now never read (decisions 30, 35).
> `POWERTOOLS_LOGGER_LOG_EVENT` follows the `*bool` rule (decision 42).

## 4. Invocation lifecycle and context propagation

This is the backbone of the library. One struct per invocation, created by the wrapper,
stored once in the context, read by every utility.

### 4.1 The per-invocation state

```go
package invocation // internal; imported by root, logger, metrics and tracer

// State is the per-invocation record. Fields are exported because the owning
// packages live outside this one; the comments say who writes what. The package
// must not import logger, metrics or tracer (they import it), so the slots those
// packages own are typed any and type-asserted by their owner (a pointer type
// assertion allocates nothing). No field holds an X-Ray SDK type: the handler
// segment is a local variable of the root wrapper (§4.2) and the DeadlineMargin
// timer lives in an unexported root struct. Fields are shown here in logical order;
// the final order is decided by betteralign (05 §4) to minimise padding and GC scan
// bytes. Contract rule: within a major version fields are only ever added, never
// renamed, retyped or removed, because three packages read this struct (11 §2).
type State struct {
    // Immutable after begin(). Safe to read from any goroutine without locking.
    RequestID    string
    FunctionARN  string
    ColdStart    bool
    Trace        TraceHeader   // parsed once from ctx.Value("x-amzn-trace-id"); zero value = no trace
    DebugSampled bool          // this invocation emits DEBUG regardless of level
    HandlerName  string        // "## main.handle"
    Started      time.Time
    Deadline     time.Time     // from ctx.Deadline(); zero when absent (local runs)
    Logger       *slog.Logger  // Options.Logger; nil means logger.FromContext returns slog.Default()
    Tracer       any           // *tracer.Tracer, for the package-level tracer.Start and Capture
    Metrics      any           // *metrics.Batch created by the wrapper, for metrics.FromContext

    // Mutable. Guarded by Mu, changed only through locked State methods
    // (AppendAttrs, RemoveAttrs, ResetAttrs, SetCorrelationID, EnsureLogBuffer).
    Mu            sync.Mutex
    CorrelationID string
    Attrs         []slog.Attr   // AppendKeys / RemoveKeys / ResetKeys
    AttrsVersion  uint32        // bumped on change; the logger caches its rendering by version
    AttrsJSON     []byte        // cached rendering of Attrs (written by the logger under Mu)
    LambdaJSON    []byte        // cached rendering of the Lambda context keys (written once by the logger)
    LogBuffer     *LogBuffer    // set once, lazily, by the first logger Handler that buffers a record;
                                // its contents are guarded by its own mutex, never by Mu (02 §9)

    // Atomics: read and written from any goroutine without Mu.
    Closed        atomic.Bool   // set by end(); late calls degrade as defined in 01 §7.4
    MetadataBytes atomic.Int64  // escaped X-Ray metadata bytes added so far by the handler segment
                                // and its Capture children; the tracer enforces the budget (04 §5)
}

type TraceHeader struct {
    Root    string // "1-5759e988-bd862e3fe1be46a994272793"
    Parent  string // "53995c3f42cd8ad8"
    Sampled bool   // true only for "Sampled=1"
    Rest    string // trailing contiguous run of other fields, e.g. "Lineage=a87bd80c:0";
                   // a substring (zero-alloc), forwarded by tracer.Header
}

func NewContext(parent context.Context, s *State) context.Context
func FromContext(ctx context.Context) *State   // nil when the wrapper is not in use
```

**Trace header parsing rule.** `ParseTraceHeader` agrees with the SDK's
`header.FromString` on `Root`, `Parent` and `Sampled`: each is recognised in any
position, a later duplicate wins, `Sampled` is true only for `Sampled=1`, and `Self=` and
fields without `=` are skipped. `Rest` is one substring over the *trailing contiguous
run* of other fields, so it costs no allocation. An unknown field placed *before* a later
known field is therefore not forwarded, whereas the SDK keeps it and `Header.String()`
would forward it. This is an accepted divergence, because Lambda appends extra fields
such as `Lineage` last; a table test in `tracer` runs the same inputs through both
parsers and checks that they agree on the three known fields.

> **Changed after implementation** (decisions 30, 31, 33, 35). Compared with the
> original struct: `Segment any` is gone (the handler segment is a local variable of the
> wrapper, so root never names an SDK type); `OpenSegments atomic.Int32` is gone (the
> SDK already knows whether a segment was emitted, §7 D7); `marginTimer` moved out of
> `State` into a root-local struct; `MetadataBytes` was added for the per-invocation
> metadata budget; `TraceHeader.Rest` was added so `tracer.Header` can forward fields
> such as `Lineage`. The `LogBuffer` got its own mutex because a flush writes many
> records and must not hold `State.Mu` while doing so.

Design notes:

- **One `context.WithValue` per invocation.** The three utilities each do one
  `ctx.Value` lookup for `State`, not three different keys. The current segment is
  carried only under the SDK's `xray.ContextKey`, in the `ctx` that the SDK returns
  when a subsegment begins, so `xray.Client`, `awsv2` and direct SDK calls see the same
  parent as the tracer.
- **Render caches are handler-independent.** `ReplaceAttr` applies only to record attrs
  and `With` attrs; it never sees built-in keys, Lambda keys or `AppendKeys` attrs
  (decision 33). `AttrsJSON` and `LambdaJSON` are therefore valid for every handler that
  logs in the invocation, including composed ones.
- **No pooling of `State`.** A goroutine started by the handler may outlive the
  invocation and still hold the context. Reusing the struct would be a data race.
  One allocation per invocation is the price; buffers inside it are pooled.
- **Goroutine inheritance.** Any `go func() { … }(ctx)` in the handler logs with the
  same keys, adds to the same metric batch, and opens subsegments under the current
  segment, because all of that is reachable from `ctx`. Work that finishes after the
  handler returns is not part of the invocation any more; what happens to it, and what
  consumers must do with their goroutines, is specified in §7.
- **Without the wrapper** (`FromContext` returns nil) every utility degrades: the logger
  omits Lambda keys, and `logger.FromContext` returns `slog.Default()` (also when
  `State.Logger` is nil); `metrics.FromContext` returns a nil `*Batch` whose methods are
  no-ops; sampling is off, because it requires the wrapper. `tracer.FromContext` returns
  nil when there is no segment, and **`*xray.Segment` methods are not nil-safe**
  (`AddAnnotation`, `AddMetadataToNamespace`, `AddError` and `DownstreamHeader` panic on
  nil; only `Close` checks). Handler code therefore uses the nil-safe helpers
  `tracer.AddAnnotation(ctx, …)` and `tracer.AddMetadata(ctx, …)`, which return nil
  without a segment, or checks for nil itself. Package-level `tracer.Capture` and
  `CaptureResult` use the SDK when a segment or a sampled trace header is present in
  `ctx` and run `fn` untraced otherwise. This keeps unit tests of handler code free of
  ceremony.

> **Changed after implementation** (decision 31). The original plan said
> "`tracer.FromContext` returns a nil `*Segment` whose methods are no-ops". That is
> false for the X-Ray SDK, and it would have turned every untraced invocation into a
> panic. The nil-safe helpers above are the fix; the wrapper itself nil-checks the
> segment before every SDK call.

Illustrative handler code that is safe with and without the wrapper:

```go
func handle(ctx context.Context, in Order) (Receipt, error) {
    logger.AppendKeys(ctx, slog.String("order_id", in.ID))  // no-op without the wrapper
    logger.Info(ctx, "processing order")                     // falls back to slog.Default()

    _ = metrics.FromContext(ctx).Add("Orders", metrics.Count, 1) // nil *Batch: no-op, nil error
    _ = tracer.AddAnnotation(ctx, "order_id", in.ID)              // no segment: no-op, nil error

    // Never: tracer.FromContext(ctx).AddAnnotation(...) — nil when not traced, and it panics.
    return Receipt{OrderID: in.ID}, nil
}

func TestHandle(t *testing.T) {
    got, err := handle(t.Context(), Order{ID: "o-1"}) // no wrapper, no Lambda context
    if err != nil {
        t.Fatalf("handle() error = %v", err)
    }
    if got.OrderID != "o-1" {
        t.Errorf("handle().OrderID = %q, want %q", got.OrderID, "o-1")
    }
}
```

```mermaid
flowchart TB
  subgraph chain["context chain for one invocation"]
    direction TB
    base["runtime ctx<br/>deadline, lambdacontext.LambdaContext,<br/>value 'x-amzn-trace-id'"]
    st["+ invocation.State<br/>(one WithValue)"]
    seg["+ current *xray.Segment under xray.ContextKey<br/>(only when sampled)"]
    base --> st --> seg
  end
  seg --> handler["handler(ctx, in)"]
  handler --> g1["goroutine A (ctx)"]
  handler --> g2["goroutine B (ctx)"]
  handler --> sub["tracer.Start(ctx, name)<br/>+ child segment"]
  g1 --> log["logger.FromContext(ctx)<br/>same keys, same xray_trace_id"]
  g2 --> met["metrics.FromContext(ctx)<br/>same Batch (mutex)"]
  sub --> http["tracer.HTTPClient / InstrumentAWS<br/>X-Amzn-Trace-Id: Root=..;Parent=child-id;Sampled=.."]
  http --> downstream["downstream service"]
  classDef state fill:#fff4d6,stroke:#b7791f,color:#000;
  class st,seg state;
```

Reading the diagram: everything below the chain inherits the same `State`, so a log line
from goroutine B carries the same `function_request_id` and `xray_trace_id` as one from
the handler, and a child segment opened inside goroutine A hangs off the handler
segment. Outbound calls carry the current segment id as `Parent`.

### 4.2 The wrapper

```go
package powertools

type Options struct {
    Logger  *slog.Logger      // from logger.New; nil means the wrapper's own lines and logger.FromContext use slog.Default()
    Metrics *metrics.Metrics  // may be nil
    Tracer  *tracer.Tracer    // may be nil

    LogEvent                   *bool  // log the incoming event at INFO ("Lambda invocation event"); nil means
                                      // POWERTOOLS_LOGGER_LOG_EVENT, else off; new(false) switches it off
    CorrelationIDPath          string // dot path on the raw payload (WrapHandler) or on the marshalled typed event (Wrap)
    CorrelationIDFunc          func(ctx context.Context, event any) string // wins over the path; event is In under Wrap,
                                                                           // the raw payload as jsontext.Value under WrapHandler
    FlushBufferOnUncaughtError bool
    CaptureColdStartMetric     bool
    CaptureResponse            *bool  // nil = tracer default
    CaptureError               *bool
    HandlerName                string // replaces the derived name; the segment is always named "## " + name
    PprofLabels                bool   // run the handler under pprof.Do with function_request_id and function_name labels (§6)
    DeadlineMargin             time.Duration // > 0 arms a pre-timeout flush of logs and metrics this long before the deadline (01 §7, D6)
}

// Wrap adapts a typed handler. In and Out are inferred.
func Wrap[In, Out any](fn func(context.Context, In) (Out, error), opts Options) func(context.Context, In) (Out, error)

// WrapHandler adapts a hand-written lambda.Handler that owns the bytes it returns.
// It does not support lambda.NewHandler(fn), whose Invoke returns bytes from a pooled
// buffer it has already released (R8). Payload and response are seen as raw JSON.
func WrapHandler(h lambda.Handler, opts Options) lambda.Handler
```

Options is read once, when `Wrap` or `WrapHandler` is called, together with the
environment variables it depends on (`POWERTOOLS_LOGGER_LOG_EVENT`,
`POWERTOOLS_LOGGER_SAMPLE_RATE`, `AWS_LAMBDA_INITIALIZATION_TYPE`). Changing an Options
value afterwards has no effect on a wrapper already built, and one Options value may
configure several wrappers. With every utility nil, the wrapped handler behaves exactly
like the unwrapped one plus the per-invocation state in its context.

> **Changed after implementation.** `ResetKeys` was removed: `State` is per invocation,
> so clearing it at the end was a no-op (decision 33). `RaiseOnEmptyMetrics` moved to
> `metrics.Options`, because `Flush` is what reads it and consumers who flush without
> the wrapper need it too (decision 32). `LogEvent` became a `*bool` (decision 42).

`Wrap` is the recommended form: no reflection, no re-encoding, generic type inference
gives the caller `lambda.Start(powertools.Wrap(handle, opts))`. The other handler shapes
of R8 use `Wrap` with a one-line closure adapter; v1 has no extra adapter API
(decision 29). Illustrative:

```go
// func(context.Context, Event) error, adapted once in main
lambda.Start(powertools.Wrap(func(ctx context.Context, e Event) (any, error) {
    return nil, process(ctx, e)
}, opts))

// a hand-written lambda.Handler, for example a router that owns its response bytes
type router struct{ /* routes */ }

func (r *router) Invoke(ctx context.Context, payload []byte) ([]byte, error) { /* … */ }

lambda.Start(powertools.WrapHandler(&router{}, powertools.Options{
    Logger:            lg,
    CorrelationIDPath: logger.CorrelationPathAPIGatewayHTTP, // evaluated on the raw payload
}))
```

`WrapHandler` exists for hand-written `lambda.Handler` implementations, such as
middleware chains, that own the bytes they return. It works on the raw payload (the
`Invoke` argument) and the raw response bytes: it passes them as `jsontext.Value` to
`CorrelationIDPath`, `CorrelationIDFunc` and `LogEvent`, and to the tracer for response
metadata. It never decodes the payload and never observes typed values. The payload
passed to `CorrelationIDFunc` is valid only until the call returns (`bytes.Clone` it to
keep it).

Both entry points share one unexported lifecycle in the root package. The handler
segment is a local variable of the per-invocation closure, captured by the deferred end:
root never names its type (that would be a direct SDK import, R11) and never stores it
in `State`. The tracer's exported end-of-handler helper does everything that touches the
segment. The margin timer sits in an unexported root struct local to `begin` and `end`.
A sketch of the shape, not of the code:

```go
return func(ctx context.Context, in In) (out Out, err error) {
    ctx, s, c := o.begin(ctx, in)                     // steps 1 to 9 of §4.3
    ctx, seg := o.tracer.Start(ctx, s.HandlerName)    // step 10; nil-safe on a nil *Tracer; nil unless sampled
    defer func() {
        p := recover()
        c.stopTimer()                                 // step 13
        held := o.tracer.EndHandler(ctx, seg, tracer.HandlerEnd{ /* name, cold start, out, err or panic */ })
        if e := o.end(ctx, s, held, err, p); err == nil && e != nil {
            err = e                                   // the Flush error, only when the handler succeeded
        }                                             // end re-panics when p != nil
    }()
    return fn(ctx, in)                                // inside pprof.Do when PprofLabels is set
}
```

The tracer side of that contract is exported, documented as wrapper plumbing that
consumers of `Wrap` never call; it cannot live in `internal/`, because `internal/...`
must never reach the SDK (decision 44):

```go
// HandlerEnd carries what the wrapper knows when the handler returns.
type HandlerEnd struct {
    Name            string         // handler name without the "## " prefix
    ColdStart       bool           // "ColdStart" annotation
    Response        any            // Wrap: the typed response, marshalled once with encoding/json/v2
    RawResponse     jsontext.Value // WrapHandler: the response bytes, used after IsValid
    Err             error          // the handler error, or the panic as an error
    CaptureResponse *bool          // wrapper overrides; nil = tracer default
    CaptureError    *bool
}

// EndHandler annotates, adds metadata within the invocation budget and closes seg.
// It reports whether the SDK held back the handler document because a descendant is
// still open. Safe on a nil *Tracer and a nil seg.
func (t *Tracer) EndHandler(ctx context.Context, seg *xray.Segment, e HandlerEnd) (heldBack bool)
```

The handler subsegment name defaults to `"## " + runtime.FuncForPC(reflect.ValueOf(fn).Pointer()).Name()`
under `Wrap`, computed **once at wrap time** (for example `## main.handle`), mirroring
Python's `## lambda_handler`. Under `WrapHandler` the derived name is the dynamic type of
`h`, `reflect.TypeOf(h).String()` (for example `## *main.router`), and `""` for a nil `h`
(guarded, so `WrapHandler(nil, …)` does not panic at wrap time; the nil handler panics at
invocation like any handler panic). `_HANDLER` is not used because on `provided.al2023`
it is the executable name. `Options.HandlerName` replaces the derived name; the `"## "`
prefix is always added.

> **Changed after implementation** (decisions 29, 31, 35). The original `begin`/`end`
> took raw and typed events plus the response; they now take only what they need, and
> the segment never crosses the root package boundary. The `WrapHandler` name rule did
> not exist: `runtime.FuncForPC` has no function to look at for an interface value.

### 4.3 Sequence per invocation

```text
lambda runtime ──▶ Wrap(fn)(ctx, in)
  begin (every immutable State field is computed before NewContext, §5):
    1. lc, ok := lambdacontext.FromContext(ctx)               (request id, ARN; both "" without a Lambda context)
    2. hdr := ParseTraceHeader(ctx.Value("x-amzn-trace-id"))  (only source; "" means no trace; no os.Getenv, R2)
    3. cold := started.CompareAndSwap(false, true) && coldEligible
                                                              (inverted flag: its zero value needs no init; the CAS always
                                                               runs; coldEligible = initType != "provisioned-concurrency",
                                                               resolved at wrap time)
    4. logger:  sampled := rate > 0 && rand.Float64() < rate  (rate resolved at wrap time: (*logger.Handler).SampleRate()
                                                               when opts.Logger.Handler() is ours, else from env)
       metrics: b := m.NewBatch() when opts.Metrics != nil
    5. s := &State{…, Metrics: b}; ctx = invocation.NewContext(ctx, s)   (no typed nil in an any slot)
    6. event bytes, at most once: the raw payload under WrapHandler; under Wrap, json.Marshal(in) only when
       LogEvent is on or a path is set without CorrelationIDFunc
       logger:  s.CorrelationID = CorrelationIDFunc(ctx, event) or the path lookup on the event bytes
    7. metrics: if cold && opts.CaptureColdStartMetric { m.EmitColdStart(lambdacontext.FunctionName) }
                                                              (separate EMF blob; an error is a WARN, never an invocation error)
    8. logger:  if logEvent { log "Lambda invocation event" with the event }
    9. if opts.DeadlineMargin > 0, a deadline exists and more than the margin remains {
           arm time.AfterFunc(remaining - margin, preTimeout) }                        (D6)
  call:
   10. tracer:  ctx, seg := t.Start(ctx, s.HandlerName)       ((ctx, nil) unless enabled and hdr.Sampled; the SDK builds the
                                                               facade from the ctx header, parent_id = hdr.Parent; seg stays local)
   11. defer end(...) with recover()
   12. out, err := fn(ctx, in)                                (WrapHandler: out, err := h.Invoke(ctx, payload))
  end:
   13. stop the margin timer                                  (D6; a callback already running is harmless)
   14. tracer:  held := t.EndHandler(ctx, seg, …): annotate ColdStart and Service; response or error metadata
                within the budget; seg.Close(err), so the SDK sets fault and cause and emits (one UDP write);
                held = the SDK did not emit the segment           →  if held { WARN }   (D7)
   15. logger:  on err or panic && FlushBufferOnUncaughtError, logger.FlushBuffer(ctx)
   16. metrics: s.Metrics.Flush(), one stdout write; empty batch: WARN, or ErrEmptyMetrics with RaiseOnEmptyMetrics.
                Any Flush error becomes the invocation error only when the handler succeeded; otherwise the
                handler's error (or panic) wins and the Flush error is logged at WARN
   17. s.Closed.Store(true)                                   (D4)
   18. if panicked { log ERROR with debug.Stack(); panic(panicVal) }     (R7)
   19. return out, err
```

Wrapper diagnostics are fixed strings, written through `logger.FromContext(ctx)`, so
`Options.Logger`, else `slog.Default()`; they are never dropped:

| Level | Message | Attributes |
| --- | --- | --- |
| INFO | `Lambda invocation event` | `event` |
| WARN | `invocation within <margin> of its deadline` | `remaining_ms` |
| WARN | `handler trace held back by open subsegments` | |
| WARN | `metrics flush failed` | `error` |
| WARN | `metrics snapshot flush failed` | `error` |
| WARN | `cold start metric not emitted` | `error` |
| WARN | `invalid CorrelationIDPath, correlation ID extraction disabled` | `path`, `error` (once, at wrap time) |
| ERROR | `handler panicked` | `panic`, `stack` |

**Edge cases the implementation had to decide.**

- A malformed `CorrelationIDPath` is reported once at `WARN` when the wrapper is built
  and disables the extraction; it never fails an invocation.
- A panic in `CorrelationIDFunc` (consumer code that runs in `begin`, before the
  handler) is **not** recovered; it propagates like a handler panic.
- A panic in the event's own `MarshalJSON` method, raised while the wrapper marshals the
  event for `LogEvent` or the path, **is** recovered: the event is logged as
  `"!ERROR:panic: <value>"` and no correlation ID is extracted. A plain marshal error
  logs `"!ERROR:<err>"`. The wrapper must not crash on a value the handler never asked it
  to encode.
- `In` is boxed into an `any` only when an option needs the event, and the event is
  marshalled at most once, so a wrapper with no event option adds no allocation for it.

```mermaid
sequenceDiagram
  autonumber
  participant RT as Lambda runtime
  participant W as powertools.Wrap
  participant ST as invocation.State (ctx)
  participant LG as logger
  participant MT as metrics
  participant TR as tracer / X-Ray SDK
  participant H as handler
  RT->>W: invoke(ctx, event)
  W->>MT: NewBatch(), when Metrics is set
  W->>ST: build State (request id, ARN, trace header from ctx, cold start CAS, sampling decision, batch)
  W->>LG: correlation id
  W->>MT: EmitColdStart on a cold start (opt-in)
  W->>LG: optional event log
  W->>W: arm the margin timer (opt-in, only when more than the margin remains)
  W->>TR: Start(ctx, "## main.handle"), nil unless sampled
  TR-->>W: ctx with segment, segment kept in a local variable
  W->>H: fn(ctx, in)
  H->>LG: Info / AppendKeys (reads State from ctx)
  H->>MT: Batch.Add (mutex)
  H->>TR: Capture / Start child segments, tracer.AddAnnotation
  H-->>W: (out, err) or panic
  W->>W: stop the margin timer
  W->>TR: EndHandler: annotations, response or error metadata, Close(err)
  TR-->>TR: emit one UDP datagram
  TR-->>W: held back by open subsegments? WARN when yes
  W->>LG: flush buffer on error (opt-in)
  W->>MT: Flush(), one stdout write
  W->>ST: Closed = true
  W-->>RT: (out, err) or re-panic
```

The wrapper drives the utilities only through their exported API
(`(*Metrics).NewBatch`, `(*Metrics).EmitColdStart`, `(*Batch).Flush`,
`(*Batch).FlushSnapshot`, `(*Tracer).Start`, `(*Tracer).EndHandler`,
`(*logger.Handler).SampleRate`, `logger.FlushBuffer`, `logger.FromContext`), which is
the same API consumers use when they do not use the wrapper. There is no privileged back
door, and no SDK type is named in the root package.

Ordering rationale: `xray_trace_id` comes from `State.Trace` (step 2), so every line
carries it from `begin` on, independent of the segment; the handler segment is opened
right before the call, so its duration is the handler's; the timer is stopped first in
`end` so a late `preTimeout` has as little to race with as possible; the trace is closed
before the stdout writes; the log buffer is flushed before metrics so an error's
diagnostic lines precede the EMF line; metrics flush last so a `ColdStart` blob and the
batch appear in a stable order.

> **Changed after implementation** (decisions 30 to 35). The original sequence opened
> the segment first "so that `xray_trace_id` is available"; that coupled logging to
> sampling and stretched the segment over the wrapper's own work. With the trace ID
> taken from the parsed header, the segment moved to just before the call. The batch is
> created before `NewContext`, so `State` is never mutated after it is published. The
> open-subsegment counter became a check of the SDK's own `Emitted` flag, and the
> `Flush` error rule was made explicit for every error, not only `ErrEmptyMetrics`.
>
> **Measured in implementation.** Wrapper overhead with all three utilities configured
> but writing nowhere (no I/O, outside Lambda) was 3 to 4 allocations per invocation
> (the budget was 12); with every utility nil, 2. `PprofLabels` added 3 allocations and
> `DeadlineMargin` 2. `WrapHandler` cost about 12 allocations. On a laptop-class arm64 CPU, `Wrap` with all three utilities took about
> 0.2 µs per invocation; allocation counts are machine-independent, times are not
> (05 §3).

### 4.4 Outbound propagation

| Direction | Mechanism |
| --- | --- |
| Logs ← trace | `xray_trace_id` (Root) on every line; `function_request_id` on every line |
| Metrics ← invocation | `function_name` dimension on `ColdStart`; consumers may `AddMetadata("request_id", powertools.RequestID(ctx))` |
| Traces ← logs | `correlation_id` can be annotated by the consumer: `tracer.AddAnnotation(ctx, "correlation_id", logger.CorrelationID(ctx))`, nil-safe, returns nil when there is no segment |
| HTTP → downstream | `tracer.HTTPClient` (honours the Powertools env switches) or `xray.Client` / `xray.RoundTripper` from the SDK sets `X-Amzn-Trace-Id` from `seg.DownstreamHeader()` and records a `remote` subsegment; with `Sampled=0` the header is still forwarded so downstream services do not start orphan traces. The SDK drops extra header fields such as `Lineage` (SDK behaviour, documented) |
| AWS SDK v2 → downstream | `tracer.InstrumentAWS(&cfg.APIOptions)`: no-op when tracing is disabled, otherwise `awsv2.AWSV2Instrumentor` from the X-Ray SDK (`namespace: "aws"`, `aws.operation / region / request_id`). Calling `awsv2` directly ignores `POWERTOOLS_TRACE_DISABLED` |
| Manual | `tracer.Header(ctx) string` returns the outbound header for any other client, always when a trace header exists (`Sampled=0` included, 04 §7), with the trailing other fields of the incoming header (`TraceHeader.Rest`, for example `Lineage`) appended; `tracer.TraceID(ctx)` returns the root ID for user-facing error messages |

Illustrative instrumentation, done once at init and reused by every invocation:

```go
func main() {
    cfg, err := config.LoadDefaultConfig(context.Background())
    if err != nil {
        log.Fatal(err)
    }
    tracer.InstrumentAWS(&cfg.APIOptions)          // before building service clients; no-op when tracing is off
    ddb := dynamodb.NewFromConfig(cfg)             // every call records an "aws" subsegment
    httpClient := tracer.HTTPClient(&http.Client{Timeout: 5 * time.Second}) // "remote" subsegments

    // ... build lg, met, trc, then lambda.Start(powertools.Wrap(...))
}

// Any other client: forward the header yourself.
req.Header.Set("X-Amzn-Trace-Id", tracer.Header(ctx)) // "" when there is no trace header
```

> **Changed after implementation** (decisions 22, 30, 31). `tracer.InstrumentAWS`
> mirrors `tracer.HTTPClient` so that the Powertools switches reach the AWS SDK as well;
> calling `xray.Client` or `awsv2` directly is still possible but only
> `AWS_XRAY_SDK_DISABLED` stops them. `Header` forwards `Rest`, which the SDK's own
> clients drop.

## 5. Concurrency model

| State | Classification | Protection |
| --- | --- | --- |
| `logger.Handler`, `*metrics.Metrics`, `*tracer.Tracer` | immutable after construction | none needed; documented as safe for concurrent use |
| stdout writer | shared across invocations and utilities | `syncwriter.Stdout`, one process-wide instance used when a utility's `Writer` is nil: mutex + single `Write` per record (R4). A writer supplied by the consumer is wrapped in a `syncwriter` owned by that utility instance and not shared with other utilities |
| X-Ray SDK global config and UDP emitter | shared, owned by the SDK | never mutated by the library; `xray.SetLogger(tracer.NewXRayLogger(lg))` (not goroutine-safe per the SDK) is the consumer's call from `main`; `xray.ContextWithConfig` is never called by the library, and consumers call it once at init, never per invocation |
| cold-start flag | process-wide | `atomic.Bool.CompareAndSwap(false, true)` on an inverted `started` flag; exactly one invocation wins even when N first invocations run concurrently under Managed Instances |
| `State` immutable fields | per invocation | written only in `begin` before `NewContext` |
| `State.Attrs`, `CorrelationID`, the `LogBuffer` pointer | per invocation, mutable | `State.Mu`, through locked `State` methods |
| `LogBuffer` contents | per invocation, mutable | the buffer's own `sync.Mutex` (02 §9), never `State.Mu` |
| `*metrics.Batch` | per invocation | its own `sync.Mutex`; a handler fanning out goroutines may add metrics concurrently; warnings are logged after the mutex is released |
| `*xray.Segment` tree | per invocation | the SDK's own `sync.RWMutex` per segment; only `tracer` calls SDK methods, and it reads `Emitted` under `RLock`; root never touches a segment |
| margin timer (`DeadlineMargin`, opt-in) | per invocation | `time.AfterFunc` held in a root-local struct, never in `State`; stopped first in `end()`; the only goroutine the library can create (01 §7 D6); `preTimeout` uses the same mutex-protected paths as the handler |
| `State.Closed`, `State.MetadataBytes` | per invocation | atomics, read and written from any goroutine |

**Lock order.** `LogBuffer` mutex, then `State.Mu`, then the `syncwriter` mutex.
`State.Mu` is never held across a `LogBuffer` call, and a buffer flush writes its
optional "dropped N records" notice and then every record while holding only the buffer
mutex, so concurrent flushes (the handler's error path and a `preTimeout` callback)
never interleave or duplicate records.

```mermaid
flowchart LR
  subgraph proc["process-wide, immutable after construction"]
    H["logger.Handler"]
    M["*metrics.Metrics"]
    T["*tracer.Tracer"]
  end
  subgraph shared["process-wide, shared with synchronisation"]
    W["syncwriter.Stdout (mutex)<br/>one Write per record"]
    CS["cold-start atomic.Bool<br/>CompareAndSwap"]
    SDK["X-Ray SDK globals<br/>never mutated by the library"]
  end
  subgraph inv["per invocation, created by begin()"]
    S["State: immutable fields"]
    SM["State.Mu: Attrs, CorrelationID,<br/>LogBuffer pointer"]
    LB["LogBuffer (own mutex)"]
    AT["State atomics:<br/>Closed, MetadataBytes"]
    B["*metrics.Batch (own mutex)"]
    SG["*xray.Segment tree (SDK locks)"]
  end
  H --> W
  M --> W
  H -.reads.-> S
  H -.reads/writes.-> SM
  H -.writes.-> LB
  LB --> W
  M --> B
  T --> SG
  T -.budget.-> AT
  classDef imm fill:#e8f4fd,stroke:#2b6cb0,color:#000;
  classDef mtx fill:#fff4d6,stroke:#b7791f,color:#000;
  class H,M,T,S imm;
  class W,CS,SDK,SM,LB,AT,B,SG mtx;
```

Package-level variables are limited to: the cold-start `atomic.Bool` (inverted, so its
zero value needs no init; root tests reset it through an `export_test.go` hook),
`syncwriter.Stdout` (the shared stdout writer), and `sync.Pool`s. There is no env
snapshot (§3) and no default instance of any utility (decision 7).

> **Changed after implementation** (decisions 33, 35). The env snapshot left this list;
> `syncwriter.Stdout` joined it. Without one shared default writer, a logger and a
> metrics instance would each own a mutex over the same `os.Stdout`, and a log line
> could interleave with an EMF blob under multi-concurrency.
>
> **Validated in implementation.** The concurrency claims were proved with `-race`
> tests that run several invocations of one wrapped handler in parallel goroutines, each
> with its own Lambda context and trace header, one test per utility plus one for
> concurrent first invocations (exactly one cold start) and one for `WrapHandler`, and
> check that each invocation's lines, blobs and segments stay its own.

## 6. Error and panic policy

- Library functions **return** errors; nothing in the library panics on bad input.
  `metrics.Batch.Add` returns `error` for an invalid unit, name, value or dimension
  budget; annotation validation is the SDK's (`(*xray.Segment).AddAnnotation` returns
  `error`; `tracer.AddAnnotation` passes it through and returns nil when there is no
  segment).
- Sentinel errors for programmatic checks (`errors.Is`): `metrics.ErrNamespaceMissing`,
  `metrics.ErrEmptyMetrics`, `metrics.ErrTooManyDimensions`, `metrics.ErrInvalidUnit`,
  `metrics.ErrInvalidValue`, `metrics.ErrInvalidName`, `metrics.ErrInvalidDimension`,
  `metrics.ErrBatchFlushed`, `logger.ErrInvalidLevel`. Annotation validation errors are
  the SDK's own.
- Wrapped errors use `%w` only where the wrapped error is part of the contract and that
  is documented. Typed errors are matched with `errors.AsType[T]` (Go 1.26) in docs and
  tests. No error type is ever wrapped through a pointer to it; Go 1.27's `go vet`
  reports that pattern.
- Values that `encoding/json/v2` cannot marshal never fail a call: the logger writes
  `"!ERROR:<err>"` in their place, metrics drops the metadata value with one `WARN` per
  batch, and the tracer skips metadata that is invalid or over the budget with a `WARN`
  (§2 JSON policy, 04 §5).
- I/O failures of the stdout writer never fail the invocation; they are logged once at
  `DEBUG` through `logger.FromContext(ctx)`. The wrapper's own diagnostics (§4.3) follow
  the same rule: `Options.Logger`, else `slog.Default()`; they are never dropped.
- UDP emission failures are handled by the SDK and logged through the SDK's logger.
  Consumers call `xray.SetLogger(tracer.NewXRayLogger(lg))` in `main`; without it the
  SDK's default logger writes plain-text lines to stdout, which ALC then treats as INFO.
  The library never calls `xray.SetLogger` itself.
- Handler panics: the wrapper recovers, performs steps 13 to 17 (the tracer closes the
  handler segment with the panic as its error), logs the panic value and
  `runtime/debug.Stack()` at `ERROR`, then re-panics with the original value so
  `aws-lambda-go` still reports a `Runtime.ExitError`-style failure with a stack (R7).
  Because the deferred function runs before unwinding, the runtime's own stack capture
  still contains the original frames.
- With `Options.PprofLabels`, the handler runs inside `pprof.Do(ctx, pprof.Labels(
  "function_name", name, "function_request_id", id), fn)`. The labels identify the
  invocation in CPU and goroutine profiles, in `SIGQUIT` dumps taken while the handler
  runs, and in crash tracebacks of goroutines started by the handler: they inherit the
  labels, and since Go 1.27 the goroutine header of a traceback prints them, so a crash
  in a multi-concurrency environment can be tied to one invocation (an unrecovered panic
  in such a goroutine crashes the whole process, every concurrent invocation included).
  A panic in the handler itself carries **no** labels: `pprof.Do` restores the parent
  labels in its own defer before the wrapper's deferred recover runs, and aws-lambda-go
  recovers every handler panic and reports it through the Runtime API without a Go
  traceback. Cost: 3 allocations per invocation in the reference implementation; off by
  default.
- `Flush` on an empty batch is a warning, not an error, unless
  `metrics.Options.RaiseOnEmptyMetrics` is set (03 §7). `Flush` returns only sentinels
  (`ErrEmptyMetrics`, `ErrNamespaceMissing`), never an I/O error, and the wrapper returns
  any `Flush` error as the handler error only when the handler succeeded. When the
  handler returned an error or panicked, that error wins and the `Flush` error is logged
  at `WARN`. A batch counts as empty only if it never held a metric value during the
  invocation, so a `DeadlineMargin` snapshot or an auto-flush that already wrote every
  value does not make the final `Flush` warn or return `ErrEmptyMetrics`; this is a
  deliberate, documented difference from Python. `FlushSnapshot` on an empty batch writes
  nothing and warns nothing.

> **Changed after implementation** (decisions 32, 35). The `Flush` error rule was
> originally written for `ErrEmptyMetrics` only; it now covers every `Flush` error, and
> `Flush` never surfaces I/O errors, so a full disk or a closed pipe cannot turn a
> successful business result into a failed invocation. The `PprofLabels` text originally
> promised labels in the handler's own panic traceback; reading `pprof.Do` and
> aws-lambda-go showed that cannot happen, so the claim was narrowed.

## 7. Deadlines, cancellation, freeze and shutdown

Lambda is not a long-running service: every invocation has a deadline, the environment
is frozen between invocations, a timeout resets the runtime process, and shutdown gives
at most two seconds. This section states the facts, the rules the library follows, and
what consumers must do with their own goroutines.

### 7.1 Facts

| # | Fact | Source |
| --- | --- | --- |
| F1 | The runtime learns the deadline per invocation (`Lambda-Runtime-Deadline-Ms`). `aws-lambda-go` wraps the handler context with `context.WithDeadline(base, deadline)` and `defer cancel()`, so `ctx.Done()` closes at the deadline **or** as soon as the invocation finishes (after the response is posted), whichever comes first. | `lambda/invoke_loop.go` v1.55.1 |
| F2 | "Lambda freezes the execution environment when the runtime and each extension have completed and there are no pending events." Goroutines still running are suspended, not killed. | Lambda execution environment lifecycle |
| F3 | "Background processes or callbacks that were initiated by your Lambda function and did not complete when the function ended resume if Lambda reuses the execution environment. Make sure that any background processes or callbacks in your code are complete before the code exits." | same |
| F4 | On a timeout or crash during Invoke, "the Lambda service performs a reset. The reset behaves like a `Shutdown` event. First, Lambda shuts down the runtime". The next invocation re-runs Init (a suppressed init). Memory is gone: anything not flushed before the deadline is lost. | same |
| F5 | Shutdown duration limit: 0 ms with no extensions, 500 ms with a registered internal extension, 2000 ms with an external extension; then `SIGKILL`. `lambda.WithEnableSIGTERM` registers an internal extension so the process receives `SIGTERM` with about 500 ms to act. | same; `lambda/handler.go` |
| F6 | Under Managed Instances several invocations run concurrently in one process, each with its own context and deadline (R4). | 01 §1 |

### 7.2 Rules the library follows

| # | Rule |
| --- | --- |
| D1 | **End-of-invocation work never waits on `ctx`.** `end()` runs deferred, also when the handler returned `ctx.Err()`, and its I/O is local and synchronous: one `write(2)` to stdout for the EMF blob, one for the buffered log lines, one UDP datagram for the trace. None of it consults `ctx.Done()`, because a cancelled context must not prevent the flush that explains the cancellation. Its cost is bounded (05 §3). |
| D2 | **Nothing is pending between invocations.** Metrics, buffered logs and the handler segment are flushed synchronously in `end()`. The library therefore needs no shutdown hook and loses nothing at freeze (F2) or at `SIGTERM` (F5). |
| D3 | **The library creates no goroutine on its own**, with one opt-in exception, D6. Nothing of ours can be left running when the environment freezes. |
| D4 | **A finished invocation is closed, not recycled.** `end()` sets `State.Closed`. Objects reachable from a late goroutine (`State`, the metrics `Batch`, the `LogBuffer`) are **never pooled**, so a goroutine that outlives the handler (F3) can only ever touch its own invocation's data, never the next one's. Late calls degrade in a defined way (7.4). |
| D5 | **Helpers for deadline-aware code.** `powertools.RemainingTime(ctx) time.Duration` (the Go form of `getRemainingTimeInMillis`) and `powertools.Deadline(ctx) (time.Time, bool)`. Both read `ctx.Deadline()`; they exist so handler code does not need to know how the runtime encodes it. `RemainingTime` is clamped at 0 once the deadline has passed, and without a deadline (local runs, unit tests) it returns `time.Duration(math.MaxInt64)`, meaning no limit, so the 7.5 pattern `context.WithTimeout(ctx, RemainingTime(ctx) - margin)` keeps working there. |
| D6 | **Opt-in pre-timeout flush** (`Options.DeadlineMargin`, default 0 = off). With a margin of, say, 100 ms, `begin()` arms `time.AfterFunc(remaining - margin, preTimeout)` when the context has a deadline and more than the margin remains; otherwise the timer is not armed, because a timer that fires at once would only race the handler. The timer lives in an unexported root-package struct local to `begin` and `end`, never in `State`. If the handler is still running when it fires, `preTimeout` writes a `WARN` line (`"invocation within <margin> of its deadline"` with `remaining_ms`), flushes the log buffer and flushes a **snapshot** of the metrics batch: `FlushSnapshot` writes the values recorded so far and then clears them, keeping dimensions, metadata and the open state, exactly like the 100-metric auto-flush (03 §3). The batch keeps accepting and `end()` flushes the rest. The trace is not touched: closing the handler segment early would misreport the invocation. `end()` stops the timer first. This is the only timer and the only goroutine the library can create; its lifetime ends at `end()`, except for a callback that has already started, which finishes on its own. Race with `end()`, defined: `preTimeout` first checks `State.Closed` and returns if set; `end()` calls `Stop()` and does not wait for a callback that already started, which is safe because every path the callback uses is mutex-protected and idempotent (`FlushSnapshot` on an empty or flushed batch writes nothing, a buffer flush writes under the buffer's mutex and on an empty buffer writes nothing, and the `WARN` goes through the same `syncwriter.Stdout`). The worst case is one extra `WARN` line and one extra EMF blob containing only metrics added between the snapshot and `Flush`, never a lost or duplicated metric value, because the snapshot clears what it wrote. Reason: after a timeout the runtime is reset (F4), so diagnostics that were not written before the deadline never reach CloudWatch. |
| D7 | **Leaks become visible.** After closing the handler segment, the tracer's end-of-handler helper checks whether the SDK emitted it (`seg.RLock(); held := !seg.Emitted; seg.RUnlock()`). The SDK emits a closed segment only when none of its descendants is open, so `held` means a child opened by `tracer.Start`, `Capture`, `xray.Client`, `awsv2` or a direct SDK call is still open, and the wrapper logs one `WARN` ("handler trace held back by open subsegments"); such a child delays or loses the whole handler trace (7.5). |

> **Changed after implementation** (decisions 24, 31, 33, 35, 44).
> - **D5:** the plan did not say what `RemainingTime` returns without a deadline.
>   Returning 0 would make `context.WithTimeout(ctx, RemainingTime(ctx) - margin)`
>   expire at once in every unit test and local run; returning the maximum duration
>   keeps the same handler code working everywhere. The API review kept this choice.
> - **D6:** the literal `AfterFunc(remaining - margin)` fires immediately when less than
>   the margin remains, so the timer is not armed then. `FlushSnapshot` was specified to
>   clear what it wrote, which is what makes the race with `end()` free of duplicates.
> - **D7:** the plan's `State.OpenSegments` counter could only see children opened
>   through `tracer`, not `xray.Client`, `awsv2` or direct SDK calls. The SDK's own
>   `Emitted` flag sees all of them and needs no bookkeeping.
>
> **Testing techniques that proved necessary.** The leak test combines three checks,
> because none is sufficient alone: the Go 1.27 `goroutineleak` profile (count 0 and no
> stack mentioning the module), which finds goroutines blocked forever; a scan of the
> full goroutine profile for frames of this module; and a `testing/synctest` test that
> proves a stopped margin timer never fires. A pending `time.AfterFunc` is not a
> goroutine, so the profiles alone cannot show it. `synctest` also made every
> deadline test deterministic: fake time advances only when every goroutine in the
> bubble is blocked, so "fires at deadline minus margin" is asserted exactly, without
> sleeps.

### 7.3 Timeline of an invocation

```mermaid
sequenceDiagram
  autonumber
  participant L as Lambda service
  participant RT as aws-lambda-go runtime
  participant W as powertools.Wrap
  participant H as handler
  participant G as handler goroutines
  participant T as margin timer (opt-in)
  L->>RT: Next: event + Lambda-Runtime-Deadline-Ms
  RT->>W: ctx = WithDeadline(base, deadline)
  W->>W: begin(): State, batch
  W->>T: arm AfterFunc(remaining - margin) when DeadlineMargin is set and more than the margin remains
  W->>W: open the handler segment when sampled
  W->>H: fn(ctx, in)
  H->>G: go work(ctx) ... (errgroup / WaitGroup.Go)
  G-->>H: Wait() before returning
  H-->>W: (out, err)
  W->>T: Stop()
  W->>W: end(): close segment (UDP) and check it was emitted, flush buffer, flush batch (stdout), State.Closed = true
  W-->>RT: (out, err)
  RT->>L: response, then Next
  RT->>RT: cancel(), ctx.Done() closes
  L->>L: freeze environment (goroutines still running are suspended, F2)
  Note over L,G: On the next invocation the environment thaws: a goroutine that was not waited for resumes with the OLD ctx (already cancelled) and the OLD State (Closed). It cannot touch the new invocation.
```

### 7.4 What a late call does after `State.Closed`

```mermaid
stateDiagram-v2
  [*] --> Open: begin()
  Open --> Open: handler and its goroutines use logger, metrics, tracer
  Open --> PreTimeout: margin timer fires (DeadlineMargin, handler still running)
  PreTimeout --> Open: WARN logged, buffer flushed, metrics snapshot flushed and cleared, batch keeps accepting
  Open --> Closed: end() (handler returned, errored or panicked)
  PreTimeout --> Closed: end()
  Closed --> Closed: late logger call is written immediately with the old invocation's keys (never buffered)
  Closed --> Closed: late Batch.Add returns ErrBatchFlushed, value dropped, one WARN per invocation
  Closed --> Closed: late tracer.Start or Capture returns (ctx, nil), fn runs untraced, one WARN
  Closed --> Closed: Header(ctx), TraceID(ctx), RemainingTime(ctx) still answer
  Closed --> [*]: State unreferenced, garbage collected (never pooled)
```

The rules are chosen so that a late goroutine is **observable** (its log lines still carry
`function_request_id` of the invocation that started it) but can never **corrupt**
another invocation: there is no object reuse to corrupt.

The `Closed` guard covers `tracer.Start` and `Capture` only. `xray.Client`, `awsv2` and
direct SDK calls in a late goroutine still open a subsegment under the already emitted
handler segment, and the SDK re-emits the handler document when that child closes.
Waiting for goroutines before returning (7.5) is the only cure.

**Streaming responses.** The lifecycle ends when `fn` returns. When `Out` is an
`io.Reader` (for example a function URL streaming response), aws-lambda-go reads the body
only after that: the margin timer is already stopped, the metrics batch flushed, the
handler segment closed and `State.Closed` set while the body is produced. Telemetry the
body producer writes later is late in the sense of this section, and `DeadlineMargin`
does not cover the streaming phase. The wrapper returns the reader untouched and never
reads it; response capture skips it (04 §5). Response streaming stays unsupported in v1
(decision 17).

> **Changed after implementation** (decision 31). The late-call guard was narrowed to
> what the tracer controls, and the SDK's re-emission behaviour was documented instead
> of promised away. The streaming note was added: when `Out` is a reader the answer is
> "late telemetry", and saying so is cheaper than supporting it in v1.

### 7.5 What consumers must do with their goroutines

```mermaid
flowchart TD
  start([handler starts goroutines]) --> q1{Must the result be in this invocation's response, logs, metrics or trace?}
  q1 -- yes --> wait["Use errgroup.WithContext(ctx) or sync.WaitGroup.Go and Wait() before returning.<br/>Derive deadlines from ctx: context.WithTimeout(ctx, RemainingTime(ctx) - margin)."]
  q1 -- no --> q2{Can it be deferred to another invocation or service?}
  q2 -- yes --> queue["Hand it off: SQS, EventBridge, Step Functions, a second function.<br/>Do not rely on it running after return (F2, F3)."]
  q2 -- no --> wait
  wait --> seg{Does the goroutine open subsegments?}
  seg -- yes --> close["Close them before the handler returns: with the X-Ray SDK an open child holds back the emission of the whole handler trace (the SDK emits only when no descendant is open).<br/>The wrapper warns when the handler segment was not emitted at end()."]
  seg -- no --> ok([ok])
  close --> ok
  queue --> ok
  wait -. never .-> bg["context.Background() or a detached ctx inside a handler:<br/>loses the deadline, the request id and the trace"]
```

Concretely, the pattern the examples use (illustrative):

```go
func handle(ctx context.Context, in Order) (Receipt, error) {
    ctx, cancel := context.WithTimeout(ctx, powertools.RemainingTime(ctx)-200*time.Millisecond)
    defer cancel()                      // leave 200 ms for end() and the runtime; also works without a deadline (D5)

    g, gctx := errgroup.WithContext(ctx)
    g.Go(func() error { return tracer.Capture(gctx, "inventory", func(ctx context.Context) error { return reserve(ctx, in) }) })
    g.Go(func() error { return tracer.Capture(gctx, "payment",   func(ctx context.Context) error { return charge(ctx, in) }) })
    if err := g.Wait(); err != nil {    // all goroutines and their subsegments are done here
        return Receipt{}, err
    }
    return receipt(in), nil
}
```

`errgroup` is `golang.org/x/sync`, a consumer choice; the standard library alternative is
`sync.WaitGroup.Go` (Go 1.25) with a shared error variable. `context.AfterFunc(ctx, cleanup)`
(Go 1.21) is the way to react to cancellation without a goroutine of your own.

### 7.6 Timeout scenario, with and without the margin

```mermaid
sequenceDiagram
  autonumber
  participant L as Lambda service
  participant W as powertools.Wrap
  participant H as handler (ignores ctx)
  participant T as margin timer
  participant CW as CloudWatch / X-Ray
  W->>T: arm at deadline - 100 ms
  W->>H: fn(ctx, in)
  H->>H: blocked on a slow downstream call
  T->>W: preTimeout()
  W->>CW: WARN "within 100ms of deadline", buffered logs, metrics snapshot
  L->>L: deadline reached: "Task timed out", runtime reset (F4)
  Note over W,CW: Lost: the handler trace (segment never closed), any metric added after the snapshot, any log after the flush.
  Note over W,CW: Without DeadlineMargin everything buffered and the whole metrics batch are lost. With it, the diagnostics up to the margin survive.
  L->>L: next invocation: suppressed Init (cold start), new State
```

### 7.7 Shutdown

The library keeps nothing between invocations (D2), so it registers no shutdown hook.
Consumers who hold resources (connections, local caches to persist) use the runtime's
own mechanism, which composes with the wrapper:

```go
lambda.StartWithOptions(
    powertools.Wrap(handle, opts),
    lambda.WithEnableSIGTERM(func() { db.Close() }),   // ~500 ms before SIGKILL (F5)
)
```

Under Managed Instances (F6) the same rules hold per invocation; the margin timer, when
enabled, is one per invocation, and `end()` of one invocation never touches another.

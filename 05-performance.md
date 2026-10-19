# 05. Performance

Performance is a requirement, not a later optimisation. This file defines the hot paths,
the budgets, how they are measured and how CI enforces them. Allocation budgets are exact
requirements; latency figures are targets on the Lambda architecture, or expected costs
relative to a baseline measured in the same run, so they hold on any machine.

## 1. Principles

1. **Nothing per record that can be done per process or per invocation.** Static attrs
   are rendered at construction; Lambda context keys are rendered once per invocation
   and cached; environment variables are read once, by constructors.
2. **No reflection, `fmt` or `regexp` on hot paths; no v1 `encoding/json` anywhere.**
   The only reflection runs once at wrap time: `runtime.FuncForPC` names a `Wrap`
   handler, and `reflect.TypeOf(h)` names a `WrapHandler` handler (04 §5). `%T` and
   `encoding/json/v2` are allowed on cold paths: error rendering, metadata of arbitrary
   type, response capture (decision 23).
3. **Pooled buffers, single write.** `internal/bufpool` hands out `[]byte` with a
   64 KiB return cap (larger buffers are dropped, not pooled, to avoid pinning memory).
4. **Zero work when disabled.** Level gate, unsampled trace, disabled metrics: no
   allocation, no lookup beyond what decides the gate.
5. **Synchronous.** No background goroutines, timers or channels in the library's own
   code: nothing to lose at freeze, nothing to drain, trivially testable (Google: prefer
   synchronous functions). The single exception is the opt-in `DeadlineMargin` timer
   (01 §7 D6), one `time.AfterFunc` per invocation, stopped in `end()`, tested under
   `synctest`; its callback runs in a goroutine only if it fires, and 01 §7 D6 defines
   the race with `end()`. Scope of the claim: `internal`, `logger`, `metrics`, `tracer`
   and the root wrapper. The X-Ray SDK, imported only by `tracer`, has paths that start
   goroutines (`xray.BeginSegment` and other root segments, `xray.CaptureAsync`, the
   centralized sampling poller on the first `ShouldTrace`); the wrapper calls none of
   them, and in Lambda facade mode `BeginSubsegment` and `Close` are synchronous
   (04 §1). Consumers who call those SDK functions directly own those goroutines.
   **The proof has three parts**, because the `goroutineleak` profile reports only
   goroutines permanently blocked on unreachable primitives, and a pending
   `time.AfterFunc` is not a goroutine at all, so no profile can show it. After 1000
   wrapped invocations, with and without `DeadlineMargin`: the `goroutineleak` profile
   reports zero; a scan of the full `goroutine` profile (`WriteTo(w, 2)`) finds no frame
   from the module's import path outside the test; and a `synctest` test returns before
   the margin, sleeps past the deadline and asserts that the stopped timer never fired
   (no pre-timeout `WARN`, no extra EMF blob). 06 §1 lists the tests.
6. **Cold start matters as much as steady state.** No `init()` work: constructors read
   the environment when called and cache the result, so nothing reads it per invocation
   and tests can `t.Setenv` before constructing (decision 35). No package-level tables
   built at init that are not constants; standard library only in `internal`, `logger`
   and `metrics`. The X-Ray SDK (whose package init reads the environment and dials
   nothing) is imported only by the `tracer` package's production code. The root wrapper
   imports `tracer` (its handler-segment plumbing lives there), so **every binary that
   calls `Wrap` links the X-Ray SDK**, even with `Options.Tracer` nil; only a binary that
   uses `logger` and `metrics` without the wrapper stays SDK-free, which is what the
   size gate checks (§6). Test files may import the SDK: they are never linked into
   consumer binaries (decision 31).
7. **Small when unavoidable.** Go 1.27's size-specialised allocators make objects under
   80 bytes up to 30 % cheaper to allocate. Per-record allocations that cannot be
   pooled away (an `*XRayLogger` message wrapper, a sanitised segment name) are kept
   under that size; per-invocation objects (`State`, the batch) are allocated once and
   may be larger. `betteralign` (§4) keeps the small ones small.

## 2. Hot-path inventory

| Path | Frequency | Design |
| --- | --- | --- |
| `Handler.Enabled` | every log call | integer compare; one `ctx.Value` only when sampling can matter |
| `Handler.Handle` scalar attrs | every emitted log | pooled buffer, appends only, one `Write` |
| `Handler.Handle` with `KindAny` | occasional | `encoding/json/v2` into the same buffer |
| `AppendKeys` | few per invocation | lock, append, bump version; rendering deferred to next `Handle` and cached |
| `Batch.Add` existing name | many per invocation | lock, map lookup, append float |
| `Batch.Add` new name | ≤ 100 per invocation | map insert, `names` append |
| `Batch.Flush` | once per invocation | one serialization, one `Write` |
| `tracer.Start`/`Close` disabled or unsampled | tens per invocation | cached boolean checks, no SDK call, 0 allocations |
| `tracer.AddAnnotation`/`AddMetadata` without a segment | tens per invocation | one `ctx.Value`, return `nil`, 0 allocations |
| `tracer.Start`/`Close` sampled | tens per invocation | SDK-bound (`xray.BeginSubsegment`, `Close`); wrapper adds name sanitisation and one `os.Getenv` for the SDK switch |
| handler segment `Close` | once per invocation | SDK serialises the tree and writes one UDP datagram, followed by one `RLock` read of the segment's `Emitted` field for the open-subsegment warning (04 §5.1) |
| wrapper `begin`/`end` | once per invocation | one `State` alloc, one `WithValue`, header parse without allocation (substrings of the header string) |

## 3. Budgets

The allocation budgets below are requirements, set with Go 1.27 defaults (Green Tea
garbage collector, size-specialised allocation, `encoding/json` backed by v2). Each row
becomes a `testing.AllocsPerRun` test in the phase that implements the path, with a
fixed **locking rule**:

- a row whose budget is 0 asserts `== 0`;
- any other row asserts `<= min(budget, measured + 1)`, where `measured` is the count the
  first implementation of the path reaches, so a budget tightens itself to the code;
- a row relative to the X-Ray SDK asserts
  `<= SDK baseline + min(delta budget, measured delta + 1)`, with the SDK baseline
  measured in the same test, so an SDK upgrade does not break it.

A count above its budget is never absorbed: the code is fixed, or the budget changes
with a recorded decision. The `ns/op` column holds targets on `linux/arm64`, the Lambda
architecture the CI benchmarks run on; the bench gate of §6 compares time only against
the merge base, never against these absolute numbers.

| Operation | allocs/op budget | ns/op target (linux/arm64) |
| --- | --- | --- |
| `log.Info("msg")` disabled level | 0 | < 10 |
| `log.LogAttrs(ctx, slog.LevelInfo, "msg", …)` with up to 5 scalar attrs, enabled, wrapper active | 0 | < 900 (excluding `Write`) |
| `log.LogAttrs` with 10 scalar attrs, enabled | ≤ 1 (the record's overflow slice) | < 1500 |
| `log.Info("msg", "k", v, …)` key-value form | ≤ 1 per boxed non-constant value, at the call site | caller's cost, documented |
| `log.Info` with one `slog.Any(struct)` | ≤ 3 | < 3000 |
| `logger.AppendKeys(ctx, 1 attr)` | ≤ 1 | < 200 |
| `Batch.Add` existing name | 0 | < 60 |
| `Batch.Add` new name | ≤ 2 | < 300 |
| `Batch.Flush` 10 metrics, 2 dims | ≤ 1 | < 4000 (excluding `Write`) |
| `tracer.Start` + `Close` disabled or unsampled | 0 | < 20 |
| `tracer.Start` + `Close` sampled leaf | SDK baseline + ≤ 1 | SDK baseline + < 200 |
| handler segment `Close`, 5 children, capture off | SDK baseline + ≤ 2 | SDK baseline (recorded, not gated) |
| `Wrap` overhead per invocation, all three on, no I/O | ≤ 12 | < 20000 |
| `Options.PprofLabels` extra cost per invocation | ≤ 3 | < 1500 |
| `Options.DeadlineMargin` extra cost per invocation (arm + stop) | ≤ 2 | < 500 |
| `end()` total, all three on, no I/O wait | counted in the `Wrap` row | < 20000, never waits on `ctx` (01 §7 D1) |
| package init of core (`go test -run XXX -bench init`) | n/a | < 1 ms |

What a correct implementation should reach, and what the tests must get right:

- **Expected counts below the budgets.** A careful implementation lands under several
  budgets, and the locking rule then holds it there: `AppendKeys` and the record after
  it at 0; `Batch.Add` of a new name at 1; `Batch.Flush` itself at 0 once the
  key-collision merge reuses the batch's preallocated slices; a sampled tracer leaf and
  the handler end with capture off at SDK baseline + 0, a few tens of ns over the SDK
  for the leaf and on the order of a hundred ns for the handler end; `Wrap` with all
  three utilities at a handful of allocations and well under a microsecond; the pprof
  labels at 3 and the margin timer at 2, each costing tens of ns.
- **The logger rows depend on the call form.** Up to 5 attrs fit inline in a
  `slog.Record`, so `LogAttrs` with 0, 3 or 5 scalar attrs is 0 allocations enabled or
  disabled, and 10 attrs cost 1. Key-value calls (`log.Info("msg", "k", v)`) box every
  non-constant value into `...any` at the call site, before `Enabled` runs; that cost is
  the caller's, budgeted at 1 per boxed value and documented, and `LogAttrs` is
  recommended on hot paths. The runtime boxes a small integer without allocating, so 3
  values typically cost 2.
- **Budget tests use non-constant values** (variables, not literals), or they pass
  vacuously on constant-folded arguments.
- **Paths outside the table are gated too:** the trace header parser, the pooled buffer,
  the synchronised writer and the attr encoder are 0 allocations; `tracer.Header` is 0
  without a header and ≤ 1 when rebuilt; a sanitised segment name costs ≤ 1; the SDK's
  `Close` on a nil segment costs 1 (recorded, not gated, 04 §9).

Comparison baselines, run in the same benchmark run and recorded, not gated, with every
side writing to `io.Discard`:

| Ours | Baseline | Target |
| --- | --- | --- |
| scalar attrs, wrapper off | `slog.JSONHandler`, same record | within 1.2x; expected at or below 1.0x |
| scalar attrs, wrapper on | `slog.JSONHandler`, same record | at most 1.5x; expected near 1.0x |
| one `slog.Any(struct)` | `slog.JSONHandler`, same record | none; expected close to 1.0x (both pay the JSON marshal) |
| EMF blob, 10 metrics, 2 dimensions, built from scratch | `encoding/json/v2` `json.Marshal` of the equivalent `map[string]any` | ours faster |

The EMF baseline uses `encoding/json/v2`, because v1 `encoding/json` is banned
everywhere, tests included (decision 23). Ours writes the Lambda keys (and, with the
wrapper on, the invocation keys) that `slog.JSONHandler` does not, so the ratios are
conservative. Both EMF sides build the blob from scratch in every iteration; the map
baseline should allocate several times as many objects and bytes per blob as the
hand-written encoder.

## 4. Techniques, by component

### Logger

- `preAttrs` rendered at `WithAttrs`; `preGroups` rendered at `WithGroup`.
- Level name lookup: `[6]string` indexed by `(level+8)/4` for the six canonical levels,
  fallback to `slog.Level.String()` otherwise.
- Timestamp: `time.Time.AppendFormat` with the fixed layout; no `Format` (allocates).
- Lambda context block rendered once per invocation into `State.lambdaJSON` on first
  `Handle` (lazy, so invocations that never log pay nothing).
- `AppendKeys` rendering cached by `attrsVersion`; `Handle` compares a `uint32` and
  reuses the cached bytes.
- Record attrs: `r.Attrs(func(a slog.Attr) bool)` iteration, no `[]slog.Attr`
  materialisation.
- Quoting via `jsontext.AppendQuote` only; the Go 1.27 floor makes it available, so
  there is no hand-rolled escaping.
- `Write` through `syncwriter`, which does **not** copy; the pooled buffer is handed to
  `os.Stdout.Write` directly under the mutex.

### Metrics

- `Batch` slices preallocated with caps 16 (names), 4 (values per metric), 8 (dims).
- Unit validation by `switch` on the string (compiler-generated jump), no map.
- Dimension key validation by a `[256]bool` printable-ASCII table.
- Serialization writes `"Name"` strings via the same quote routine; whole floats are
  written as integers.
- The key-collision merge (decision 32) reuses the batch's preallocated slices, so
  `Flush` itself allocates nothing.

### Tracer

- Disabled or unsampled: `Start` returns `(ctx, nil)` without allocating a new context
  and without calling the SDK (which would allocate a `Dummy` segment and read two
  environment variables per call).
- Name sanitisation strips Python's invalid characters (04 §2) after a
  `strings.ContainsAny` check over a constant ASCII string, so no package-level table is
  needed; the input string is returned unchanged when clean (0 allocations), and a dirty
  name costs 1.
- Response metadata is marshalled once with `encoding/json/v2` into bytes stored as a
  `jsontext.Value`. The SDK's v1 marshal re-validates, compacts and HTML-escapes it at
  emission (`<`, `>` and `&` grow to 6 bytes), so `IsValid` is checked first and the
  guard is a per-invocation budget (48 KiB across the handler and every `Capture` child,
  decision 33) counted at escaped size with one byte scan, without a second marshal. The
  SDK streams by subsegment count, not size, so an oversized document would be dropped
  whole (04 §1).
- The current segment lives only under the SDK's context key, so a sampled `Start` costs
  only the SDK's own `WithValue`.
- Everything else is the SDK's: its allocations are measured as a baseline benchmark
  (`xray.BeginSubsegment` + `Close` with the Lambda facade) and our overhead is gated
  relative to it, not in absolute terms.

### Struct layout (betteralign)

- Every struct that is allocated per invocation or per record (`invocation.State`,
  `invocation.TraceHeader`, `invocation.LogBuffer`, `logger.Handler`, `metrics.Batch`,
  `metrics.metricDef`, `metrics.dimension`, `tracer.Tracer`) has its fields ordered by
  [`betteralign`](https://github.com/dkorunic/betteralign), a drop-in replacement for
  `fieldalignment` that preserves comments, sorts stably and writes files atomically.
  Ordering rule applied by the tool: zero-sized fields first, then higher alignment,
  pointer-bearing before pointer-free, fewer trailing non-pointer bytes, larger size.
  The payoff is twofold: fewer padding bytes per allocation, and fewer pointer-bearing
  bytes for the garbage collector to scan, which matters for `State`, the long-lived
  per-invocation object. The X-Ray SDK's `Segment` is not ours to reorder.
- Expected sizes on 64-bit: the per-record and per-definition structs (trace header,
  log buffer, metric definition, dimension, `tracer.Tracer`) stay under the 80-byte
  allocation class or close to it; `State` and `Batch` are a few hundred bytes, allocated
  once per invocation.
- Exported option structs (`powertools.Options`, `logger.Options`, `metrics.Options`,
  `tracer.Options`) carry the directive `// betteralign:ignore` **on the opening-brace
  line** (`type Options struct { // betteralign:ignore`): they are built once per
  process, and their source order is their documentation order on `pkg.go.dev`. The tool
  honours the directive only there; placed inside the body it is silently ignored and the
  struct gets reordered.
- Exported structs that are not ignored get reordered, and with them their
  documentation order (the wrapper's end-of-handler struct, 04 §4, for example). Keyed
  literals make this harmless for code; only the reading order changes.
- Keyed struct literals everywhere (a Google style rule for literals of types from other
  packages, extended here to all struct literals) mean the tool's positional-literal
  guard never blocks a rewrite, and a reorder can never silently mis-assign a field.
- Build tags: every test file carries a build tag, and the tool ignores its own `-tags`
  flag, so the tags go through `GOFLAGS` (for example
  `GOFLAGS=-tags=unit betteralign -apply -test_files ./...`); otherwise tagged test files
  are never checked.
- Workflow: `make align` runs `betteralign -apply` on the module and on `examples`. The
  tool is single-pass and exits non-zero on the run that changes something, so
  `make align` runs it a second time to confirm a clean tree, and the developer then runs
  `go build ./...` and the tests (the tool's documented safety check, because positional
  literals of exported structs in other packages are not detected). CI runs the
  report-only mode and fails on any finding (§6).
- Installation: `go install github.com/dkorunic/betteralign/cmd/betteralign@<pinned version>`
  from one Makefile variable, like every other tool, so the module's `go.mod` never lists
  it. Alternative if tools accumulate: a separate `tools` module with Go 1.24+ `tool`
  directives, kept out of the published module so consumers' dependency graphs stay
  clean.
- Known caveats, all handled: files are LF (`.gitattributes`), there are no cgo, ABI or
  wire-format structs in the library (JSON output is assembled by hand, never by
  `unsafe` or offsets), and the false-sharing warning does not apply because no struct
  is written concurrently by goroutines pinned to different cores on purpose.

Once the hot structs are aligned they rarely need another change; the report-mode gate
catches a new field placed badly.

### Wrapper

- `State` is one allocation; the context chain grows by one `WithValue` (plus, when
  sampled, the SDK's own `WithValue` for the current segment, which lives only under the
  SDK's context key). Per-invocation objects (`State`, `Batch`, `LogBuffer`) are never
  pooled (01 §7 D4); only transient record buffers are.
- Trace header parsed by scanning the string once; every field, `Rest` included (the
  trailing run of other fields, 04 §2), is a substring, so the parse allocates nothing.
  The header comes only from `ctx.Value("x-amzn-trace-id")`, never from `os.Getenv` per
  invocation (decision 30).
- Cold start: `atomic.Bool.CompareAndSwap`.
- Handler name computed at wrap time.

## 5. Measurement protocol

- Benchmarks live next to the code (`*_bench_test.go`) and use `b.ReportAllocs()` and
  `for b.Loop()` (Go 1.24+), never `b.N`. They write to `io.Discard` or a counting
  `xray.Emitter` to exclude I/O. The counting emitter is defined in the `tracer`
  package's own test files; `internal/benchutil` is production code for the import rule
  and must not import the X-Ray SDK, so the SDK baseline benchmark lives in `tracer`'s
  tests. Gated benchmarks share a `Gated` name prefix, so the CI gate selects them with
  `-bench 'Gated'`.
- Allocation budget tests carry an extra `!race` build constraint
  (`//go:build unit && !race`) and run in a separate step without `-race` or coverage, on
  the pinned toolchain only: the race runtime makes `sync.Pool` drop items at random and
  changes allocation counts, so exact budgets are only meaningful without it. Anyone who
  runs the suite on Go tip excludes them too, because tip's escape analysis changes the
  counts.
- A benchmark harness in `internal/benchutil` builds a realistic invocation context
  (trace header, `lambdacontext`) so the wrapper-on numbers are representative.
- `make bench` runs `go test -run '^$' -bench . -benchmem -count 10 ./...` and writes
  `benchstat`-compatible output to an ignored build directory. No baseline file is
  committed and none is a gate input. A second target runs the same interleaved A/B
  comparison and gate script as CI (§6) on a developer machine, merge base against the
  working tree, so a laptop and CI give the same verdict. A results history document
  may record the toolchain, the locked budgets and benchmark medians for readers; it is
  never a gate input.
- Profiles: `make profile` writes CPU and allocation profiles for the gated benchmarks of
  each package into the ignored build directory; they are reviewed when a budget test
  fails. When the CI bench gate fails, the job profiles the regressed packages and
  uploads the profiles with both bench files, the CSV and the report as an ordinary job
  artifact.
- Lambda-level validation (07 Phase 5, optional jobs, decision 45): the benchmark is two
  binaries running the same deterministic work, one without any Powertools import and
  one under `Wrap` with logger, metrics and tracer, so the init comparison includes
  linking and package init of the X-Ray SDK (decision 37). The deployment templates
  deploy the prebuilt, measured binary rather than rebuilding it. Per function at 128 MB
  and 1024 MB: 1000 sequential warm invocations (the first, cold one dropped) and 20
  forced cold starts, `Duration` and `Init Duration` parsed from each `REPORT` line;
  overhead = p50(wrapped) − p50(plain), with p90, p99 and the mean reported and the
  toolchain read from the deployed binaries. **Targets, enforced at 1024 MB:** < 1 ms
  added per invocation and < 5 ms added init. The results are recorded whenever the
  optional jobs run; a miss is a `fix`.

## 6. CI gates

Two ideas shape the gates. Both sides of every time comparison come from **one runner
in one job**: the merge base and the head run interleaved, so no frozen baseline file
and no cross-machine noise enter the verdict. And **allocations are the hard gate, time
a loose one**: allocation counts are deterministic, while `ns/op` on shared runners is
noisy, so time fails only on a large, significant regression (decision 39).

```mermaid
flowchart LR
  pr["pull request"] --> t["build job<br/>go test -race"]
  t --> tb["non-race step<br/>Allocs and Budget tests<br/>pinned Go 1.27.N only"]
  pr --> b["bench job, one runner<br/>merge base and head interleaved<br/>count 10, benchtime 300ms, cpu 1"]
  pr --> s["size job<br/>hello: logger and metrics, no wrapper<br/>no X-Ray SDK in hello<br/>full and benchmark: recorded only"]
  pr --> l["lint and checks jobs<br/>forbidigo and depguard<br/>betteralign report mode, import rules"]
  b --> bs["benchstat -format csv<br/>base versus head"]
  bs --> ab{"allocs/op or B/op increase<br/>at p below 0.05, not accepted?"}
  ab -- yes --> fail([fail])
  ab -- no --> ns{"ns/op worse by more than 25 percent<br/>at p below 0.05, not accepted?"}
  ns -- yes --> fail
  ns -- no --> cmt["deltas posted<br/>as one PR comment"]
  cmt --> pass([pass])
  tb -- within budget --> pass
  tb -. over budget .-> fail
  s -- hello within 1 MiB over bare --> pass
  s -. hello 1 MiB or more over bare, or links the SDK .-> fail
  l -- clean --> pass
  l -. findings .-> fail
```

1. **Allocation budgets.** `AllocsPerRun` tests fail the build when a budget is exceeded,
   with the locking rule of §3: exact `== 0` for the 0-allocation paths,
   `<= min(budget, measured + 1)` for the others, SDK-relative rows against an SDK
   baseline measured in the same test. They run in a separate step without `-race` on
   the pinned toolchain (§5).
2. **The interleaved A/B bench gate** (decision 39). One required job checks out the
   merge base and the PR head and runs the gated subset (`-bench 'Gated'`) of both on the
   same runner, interleaved (base, head, base, head), with
   `-count 10 -benchtime 300ms -cpu 1`. `benchstat -format csv base=base.txt
   head=head.txt` feeds a small gate script, the same one developers run locally. Rules:
   - **benchstat always exits 0**, so the script must parse its output. It reads the two
     medians and the `p=` value, never the "vs base" column, so one `sync.Pool` drop in
     ten runs does not move the verdict.
   - `allocs/op` and `B/op` are the hard gate: any increase at p < 0.05 fails (counts are
     deterministic, so a 0 to 1 regression has equal samples on each side and
     p = 0.000); a non-significant increase is reported as a warning.
   - `ns/op` fails only above **25 %** at p < 0.05; smaller deltas are informational.
     Whether 20 % is safe on the runners is reviewed after 20 recorded pull-request runs.
   - A benchmark present on one side only is reported as `new` or `removed` and never
     gated, so renaming or adding a gated benchmark does not block its own pull request.
     A unit added with `b.ReportMetric` (for example `ns/add`) is reported as `info`,
     never gated, because it is a derived view of `ns/op`.
   - The escape hatch is a reviewed accept-list file: one benchmark name per line,
     exactly as benchstat prints it. A failing row whose name is listed becomes
     `accepted`. The file changes only in the pull request that needs it (for example an
     SDK bump that adds one allocation inside the SDK while the library's delta against
     the SDK baseline benchmark is unchanged), and that description says why. Once it
     merges, the merge base already holds the increase, so the next run reports the line
     as `stale accept` and the next pull request removes it.
   - The report goes to the job summary and to one PR comment updated in place; on a
     failure the regressed packages are profiled (§5).
   - The gate script has self-tests against fixture CSV files (an allocation regression,
     a time regression under and over the threshold, a non-significant change, `new` and
     `removed` rows, an accepted and a stale entry), run with the other static checks.
3. **Binary size** (decision 37), built with `CGO_ENABLED=0`, `-tags lambda.norpc` and
   `-trimpath -ldflags='-s -w'` for `linux/arm64`. `examples/hello` uses `logger` and
   `metrics` **without the wrapper**, because the wrapper imports `tracer` and so links
   the X-Ray SDK (§1), and must add less than 1 MiB (1 048 576 bytes) over
   `examples/bare` (`lambda.Start` only). Both sides are built in the same run with the
   same toolchain, so no baseline file is committed. The gate also checks that `hello`
   does not reach the SDK, the root package or `tracer` (`go list -deps` over its source
   and the `dep` lines that `go version -m` reads from the binary), so a small SDK subset
   cannot slip in under the limit. `examples/full` and the wrapped benchmark binary are
   built, recorded and reported on every run, not gated, because their delta is the
   SDK's.
   Expected orders of magnitude: a bare `aws-lambda-go` binary is a few MiB; logger plus
   metrics add on the order of a hundred KiB, a small fraction of the 1 MiB budget;
   `Wrap` with the tracer links the X-Ray SDK and its dependency tree (grpc, protobuf,
   fasthttp, `x/net`, …) and adds several MiB; a function that also links AWS SDK for Go
   v2 clients (`config`, DynamoDB) adds several MiB more. The plan accepts the SDK's
   cost for wrapped binaries; making them SDK-free would mean moving the tracer plumbing
   out of the wrapper's import graph, which this design does not do.
4. **Struct layout.** `betteralign ./...` in report mode on the module and on
   `examples`; any finding fails the `lint` job with the tool's own diagnostic
   (`8 bytes saved: struct of size 24 could be 16`). Test files are included
   (`-test_files`, with the tags passed through `GOFLAGS`, §4) so test fixtures do not
   drift either.
5. **Static rules.** `go vet`, `staticcheck` and `gofmt` as usual. The hot-path rule (no
   `fmt`, `reflect`, `regexp` in `internal/jsonenc`, `internal/syncwriter`,
   `internal/bufpool`, `internal/invocation`, the `logger` handler files and `metrics`)
   is enforced by `golangci-lint`'s `forbidigo` and `depguard`, scoped by package path,
   with `//nolint:forbidigo` on the documented cold-path functions (error rendering,
   `%T`). No in-house analyzer (11 §3 B).

Acceptance checks for this file:

- [ ] Every row of §3 has an `AllocsPerRun` test under the locking rule, in a
      `unit && !race` file, using non-constant values.
- [ ] The SDK-relative rows measure the SDK baseline in the same test.
- [ ] Every gated benchmark carries the `Gated` prefix and writes no I/O.
- [ ] The bench gate parses benchstat's CSV, fails on a significant allocation or
      `B/op` increase and on a significant `ns/op` regression above 25 %, and honours
      the accept list; its self-tests pass.
- [ ] The size gate builds `bare` and `hello` in the same run and fails when `hello`
      adds 1 MiB or more or reaches the SDK, the root package or `tracer`.
- [ ] The three-part goroutine proof of §1 passes with and without `DeadlineMargin`.

## 7. Forbidden on hot paths

`fmt.Sprintf`/`Fprintf`, `reflect`, `regexp`, v1 `encoding/json` (banned everywhere,
decision 23), `os.Getenv` per invocation (environment variables are read by constructors
only, and the trace header comes from the context), `map` iteration with ordering sorts,
`time.Format`, string concatenation in loops, `defer` in tiny leaf functions where it
measurably costs (allowed elsewhere), interface conversions of scalars that escape
(inside the library; the caller's own key-value boxing is documented in §3),
`context.WithValue` per log call, goroutines, channels.

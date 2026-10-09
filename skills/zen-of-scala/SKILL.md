---
name: zen-of-scala
description: Concrete Scala 3 and ZIO idioms — syntax, effect composition, error handling, resource/Scope management, and testing conventions. Use when writing or reviewing Scala 3 / ZIO code.
---

# Zen of Scala

Concrete, mechanical guidance for writing idiomatic Scala 3 and ZIO. This is
about *how to spell things* — the design philosophy (illegal states, no
mutability, domain types, the test onion) lives in the `zen-of-james` skill and
is not repeated here.

## Scala 3 Syntax & Style

- Use indentation-based (significant-whitespace) syntax. No braces for control
  structures, class/object/enum bodies, `given` bodies, or `match`. Reserve
  braces for the rare block where indentation is genuinely ambiguous.
- Never use `new`. Construct via `apply` (case classes, companion `apply`,
  `given` instances). If a Java type forces construction, wrap it.
- `.asInstanceOf` / casts should almost never appear. If you reach for one,
  the types are wrong upstream — fix them there.
- At a boundary you don't control, depend only on a library's strongly-typed
  surface. An untyped (`Any`) or reflectively-shaped field is a runtime
  landmine: a cast there — or a version skew that mis-shapes it — throws
  `ClassCastException` only when that path executes, often surfacing long after
  the change that armed it (e.g. when a log level is raised and a logger finally
  evaluates its message). If a library's convenience implementation consumes the
  untyped part, implement its small typed interface yourself and touch only the
  well-typed fields. (Real case: a magnum `SqlLogger` whose `params` was an
  `Any`-backed `Iterator[Iterator[Any]]` under version skew — replacing it with
  our own logger that reads only `sql`/`execTime`/`cause` made the whole failure
  class impossible.)
- Model ADTs with `enum`. Use `case` for each variant; use exhaustive `match`
  (no catch-all `case _` unless a default is genuinely intended).
- Implicits are `given` / `using`. Name `given` instances when they'll be
  referred to; leave them anonymous otherwise.
- Under `-language:strictEquality`, provide explicit `given CanEqual`
  instances for types you compare with `==`. This is the mechanism; the
  reasoning is in `zen-of-james`.
- Java interop: use `.nn` to assert non-null at the boundary, and convert
  Java nullables into `Option` immediately so `| Null` doesn't leak inward.
- `Option[T]` should almost never be collapsed with `.getOrElse(default)` to a
  sentinel (e.g. empty string, `0`). Turning `None` into a default silently
  hides the "absent" case and is a common bug source. Keep it as `Option` and
  branch, `map`, or `foreach` over it. Only collapse with explicit approval and
  a comment explaining why the default is safe.
- Prefer union types for alternatives over sealed hierarchies where a value is
  genuinely "one of these unrelated things": `def f: ZIO[R, ErrorA | ErrorB, A]`.

### Comments

- Only comment the unexpected or the genuinely complex. Do not narrate what the
  code already says. A surprising workaround, a subtle ordering requirement, or
  a deliberate deviation from a rule deserves a comment; a plain mapping does
  not.

### File & Package Layout

- When a source file grows past ~200 lines, split it — usually into a new
  package with focused files. Prefer one primary `object`/`type` per file.
- Keep dev-only and test-only code in `src/test`. `src/main` is production
  only; this prevents accidental backdoors and dev shortcuts shipping.

### Serialization: typed models, not JSON AST

Model every JSON shape the code knows (request and response bodies, API
payloads, config, files, MCP tool inputs and outputs, test fixtures) as a
`case class` or `enum` that `derives Schema`, and let the codec come from the
schema. Don't build or pick apart `zio.json.ast.Json` (`Json.Obj(...)`,
`Json.Str(...)`, `.get(JsonCursor...)`, `fields.find(_._1 == ...)`) or
string-concatenated JSON for a shape you know. The type is the documentation,
the compiler checks every field, and the wire format can't drift from the code.

```scala
import zio.schema.{ Schema, derived }
import zio.schema.annotation.fieldName
import zio.schema.codec.JsonCodec

final case class Artifact(
    groupId: String,
    artifactId: String,
    @fieldName("latest_version") latestVersion: Option[String],  // wire name differs
) derives Schema

enum Status derives Schema:
  case Ok, Missing

// zio-json codec from the schema, e.g. for files or other libraries
given zio.json.JsonCodec[Artifact] = JsonCodec.jsonCodec(Schema[Artifact])

// zio-http: bodies straight to and from the type
import zio.schema.codec.JsonCodec.schemaBasedBinaryCodec
val artifact: IO[Throwable, Artifact] = request.body.to[Artifact]
val response = Response(body = Body.from(artifact))
```

- One type per shape. Nested objects are nested case classes; optional fields
  are `Option`; closed sets of values are `enum`s, not `String`s; open maps
  are `Map[String, A]`. Keep the Scala names idiomatic and map wire names
  with `@fieldName` rather than naming fields after the wire.
- Decode at the edge and fail with a typed error there; the rest of the code
  only sees the model. Never `.toOption` or `.getOrElse(Json.Null)` a decode
  failure away.
- Tests use the same models: build the value, encode it, and assert on the
  decoded value, not on JSON strings or AST fragments. Compare against a JSON
  string only to pin a wire format on purpose (one golden test per shape).
- Libraries that take a `Schema` (zio-http endpoints and bodies, zio-http-mcp
  tool inputs/outputs, zio-schema codecs) get the derived schema; don't write
  a JSON Schema or a codec by hand next to the type.
- `Json` (AST) is for data whose shape the code genuinely doesn't know: a
  proxy or relay that passes payloads through, a user-supplied arbitrary
  object, a JSON-RPC envelope before dispatch on its `method`. Keep it at that
  boundary and convert to a typed model as soon as the shape is known.
- When you touch code that builds JSON by hand, replace it with a model in
  the same change if it's small; otherwise note it as follow-up.

## ZIO Effect Composition

- Prefer `zio-direct` (`defer:` / `.run`) as the default composition style over
  `for`-comprehensions or `flatMap` chains:

  ```scala
  defer:
    val blocker = ZIO.service[FetchBlocker].run.blocker
    val tmpDir  = ZIO.service[TmpDir].run
    doWork(tmpDir).run
  ```

- Do NOT nest `defer` blocks. If a sub-step is complex, extract it into its own
  method with its own `defer`, then `.run` that method from the caller.
- Access services with `ZIO.service[A]`, `ZIO.serviceWith`, or
  `ZIO.serviceWithZIO`. Model a service as a `case class` wrapping its
  collaborators/state, provided via `ZLayer`.
- Provide layers at the program edges (the `run` in `main`, the test's layer
  block) — not deep inside business logic. Avoid inter-service dependencies at
  startup between your own services (depending on external systems is fine).

### Prefer the terse combinator

These pairs are equivalent; use the left form — it reads better and is what
linters expect:

- `effect.as(value)`   over `effect *> ZIO.succeed(value)` or `effect.map(_ => value)`
- `effect.unit`        over `effect *> ZIO.unit`
- `effect.as(None)`    over `effect *> ZIO.none`
- `effect.mapBoth(onError, onSuccess)` over adjacent `.map(g).mapError(f)` /
  `.mapError(f).map(g)` (note: error mapper comes first). Only collapse when the
  two calls are adjacent — an intermediate combinator that needs the original
  value/error means leave them separate.
- `effect.when(cond)` / `effect.unless(cond)` over
  `if cond then effect else ZIO.unit` (and its inverse).
- `effect.orElseFail(newError)` over `effect.mapError(_ => newError)` when the
  original error is discarded.
- `effect.delay(duration)` over `ZIO.sleep(duration) *> effect`.
- `ZIO.foreachDiscard(anOption)(f)` over
  `ZIO.whenCase(anOption) { case Some(x) => f(x) }`. Reserve `ZIO.whenCase` for
  non-`Option` matches or multiple cases.

## ZIO Error Handling

- Model domain errors as data (`enum` / `case class`), not thrown exceptions.
  Carry them in the typed error channel, composing alternatives with union
  types: `ZIO[R, VerifyError | Internal, A]`.
- Avoid `.orDie`. Propagate failures through the typed error channel: declare
  `.outError[E]` on endpoints and `.mapError` underlying `Throwable`s into a
  typed `Internal(message)` variant. The only acceptable `.orDie` is inside a
  `ZIO.acquireRelease` finalizer — and even there prefer `.ignoreLogged`.
  (In a small, self-contained app, `.orDie` for a truly-unexpected failure like
  a low-level network fault can be pragmatic, but the default is: keep it typed.)
- Never let a failure go silent. Every failure must be BOTH logged AND surfaced:
  - Log with the operation name + identifying ids. Pick the level by cause:
    `logError`/`logErrorCause` for real bugs (`Internal`/unexpected),
    `logWarning`/`logWarningCause` for expected authorization/state rejections.
  - Surface a plain-language, user-facing message — never raw exception text.
  - A mutation that fails must NOT report success (no redirect-to-"Saved").
- Do not swallow errors into a silent fallback. `catchAll(_ => fallback)`,
  `.option`, `.either` (discarding the `Left`), `Try(...).toOption`, and empty
  `catch` blocks all hide the cause. Even a *legitimate* degradation must log:

  ```scala
  bestEffortLookup(id)
    .catchAllCause(c => ZIO.logWarningCause(s"byline lookup failed id=$id", c).as(None))
  ```

  Reach for `catchAllCause` (not `catchAll(_ => …)`) so the full `Cause`
  (including defects) reaches the log. A nested `.catchAll` swallows the error
  before any top-level handler can see it, so each nested site must log itself.

## ZIO Resources & `Scope`

- Use `ZLayer.scoped` (not `ZLayer.fromZIO` + `Scope.default`) for layers that
  build scoped resources. The layer owns a private `Scope` closed at layer
  finalization; the `Scope` never leaks into unrelated code.
- Don't declare a phantom `Scope` in a method's return type if the body doesn't
  use it — the requirement propagates to every caller. Likewise, don't wrap an
  already-non-scoped effect in `ZIO.scoped`.
- `Server.serve` enforces `HasNoScope` on the outermost routes; a `Handler`'s
  `R` must not expose `Scope`. Use `Handler.scoped[R]` to absorb an inner
  `Scope` requirement into the per-request scope.
- Streaming a file? Do NOT wrap `getDir`/lookup in a local `ZIO.scoped` — the
  body is opened lazily when the transport pulls it, so a local scope would
  close first and could free the resource mid-stream. Keep `Scope` on the inner
  ZIO and wrap the resulting `Handler` in `Handler.scoped`. A fully-materialized
  in-memory response may use a local `ZIO.scoped`.
- `forkDaemon` inherits the caller's environment, including its `Scope`. If the
  parent scope closes while the daemon runs, the daemon's `acquireRelease`
  finalizers run immediately. A forked daemon that needs a `Scope` must own it:

  ```scala
  def index(gav: GAV): ZIO[R, Nothing, Unit] =
    ZIO.scoped(doWork(gav)).forkDaemon.unit
  ```
- Use `forkDaemon` for background work that should outlive the current scope;
  `.ignoreLogged` for best-effort cleanup finalizers.

### Caching resources that own external state

`zio.cache.Cache` has no eviction callback — if a cached value owns disk/file
state, eviction leaks it. Use `zio.cache.ScopedCache`: each entry gets its own
`Scope`, eviction runs the finalizer, and per-entry reference counting keeps a
concurrent reader from being cut off mid-read.

```scala
def lookup(gav: GAV): ZIO[Client & TmpDir & Scope, NotFoundError, File] =
  defer:
    val dir = extractTo(gav).run
    ZIO.addFinalizer(deleteDir(dir).ignoreLogged).run
    dir

val cache = ScopedCache.makeWith(capacity, ScopedLookup(lookup)):
  case Exit.Success(_) => ttl
  case Exit.Failure(_) => Duration.Zero
```

`ScopedCache`'s internal `Pending` dedup is NOT guaranteed (a `Pending` entry
can be evicted under capacity pressure, starting a second concurrent lookup). If
concurrent construction for the same key isn't safe, put a
`ConcurrentMap[K, Promise[E, Unit]]` in front: first fiber wins `putIfAbsent`
and owns construction; others await the `Promise`; owner completes and removes
the entry via `.onExit`.

## Testing

- Use ZIO Test (`ZIOSpecDefault`). Write assertions as ordinary boolean
  expressions passed to `assertTrue` — avoid heavy matcher/assertion DSLs.
- Use the test-double clocks/sources by default: `TestClock`, `TestRandom`,
  `TestSystem` instead of the `Live` variants. Reach for
  `TestAspect.withLiveClock`/`withLiveRandom`/`withLiveSystem` only where a real
  one is actually required.
- Tests provide their own layers — no hidden shared fixtures. Use
  `TestAspect.sequential` for suites that share mutable external state.
- For integration tests that need real infrastructure, prefer Testcontainers
  (e.g. a container-backed layer yielding the config) over embedded native
  binaries, which fork/exec per test and race under parallel runs.
- Share one expensive container across a module's specs — do NOT start one per
  spec. Per-spec containers multiply boot/migration cost and, under rootless
  Docker, race the port manager (`bind: address already in use`) when specs
  start in parallel. Pattern: a JVM-singleton container (`lazy val`, reaped at
  exit) that migrates a `template` database once; each spec gets its own cloned
  database (`CREATE DATABASE … TEMPLATE`) so data is fully isolated and specs
  still run in parallel. Funnel every container start (and each `CREATE
  DATABASE`) through one lock so two ephemeral host ports are never programmed
  at the same instant. This is the "reuse the expensive resource, isolate the
  cheap one" shape — keep parallelism, don't pay 27× for it.
- Persist test output to a file when you run a suite, so results can be
  re-read without re-running.
- When something breaks in a confusing way, first reproduce it in a test, then
  fix.
- Guard expensive/paid integration paths (e.g. anything hitting a metered API
  key): run them very sparingly and keep them out of the default test task.

## Dependencies

- Don't declare transitive dependencies explicitly. Depend only on the libraries your code imports. Let the build resolve everything they pull in, and don't re-declare a transitive dependency just to pin its version. If an explicit dependency must follow a transitive one's version, use sbt-tdepver (see the `zen-of-projects` skill).

## Concurrency: Virtual Threads

- Use virtual threads (Java 21+) for blocking work instead of platform-thread pools.
- In ZIO apps, run blocking effects (`ZIO.attemptBlocking`, JDBC, blocking clients) on a Loom-based blocking executor, enabled in `bootstrap`:

  ```scala
  object App extends ZIOAppDefault:
    override val bootstrap = Runtime.enableLoomBasedBlockingExecutor // ++ other bootstrap layers
  ```

  Enable it in `AppTest` (the dev/test main) as well, so development matches production.
- Outside ZIO, use `Executors.newVirtualThreadPerTaskExecutor()` rather than a fixed thread pool. Don't pool virtual threads.
- Virtual threads don't make CPU-bound work faster. Keep that work on ZIO's default executor.

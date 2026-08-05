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

### Serialization

- Prefer `case class` + a derived `zio-schema` `Schema` and derived codecs over
  hand-rolling JSON AST marshalling/unmarshalling. Deriving keeps the wire
  format in sync with the type and removes a class of stringly-typed bugs.

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
- Persist test output to a file when you run a suite, so results can be
  re-read without re-running.
- When something breaks in a confusing way, first reproduce it in a test, then
  fix.
- Guard expensive/paid integration paths (e.g. anything hitting a metered API
  key): run them very sparingly and keep them out of the default test task.

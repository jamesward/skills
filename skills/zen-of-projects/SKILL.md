---
name: zen-of-projects
description: Whether creating new projects or bringing existing ones into compliance, do these things.
---

# Automation & Agent Guidance

- Every project should have an `AGENTS.md` with unique, project-oriented guidance, including which Skills apply and the exact build, test, and development workflow.
- Put automation guidance in a `.factory` directory. Record the daily routine in `.factory/DAILY.md`, including instructions to update dependencies, run CI, align the project with `AGENTS.md` and this Skill, and preserve existing work. Include this instruction:

```text
If there are other open PRs for this work, update that PR instead of creating a new one.
```

# Scala Projects

## Versions and Java

- Use the latest **stable, non-prerelease** sbt 2.x release. Do not select versions containing qualifiers such as `RC`, `M`, or `SNAPSHOT` merely because Maven metadata sorts them as latest. Resolve versions through [javadocs.dev](https://www.javadocs.dev/org.scala-sbt/sbt), then confirm ambiguous results against the [official sbt download page](https://www.scala-sbt.org/download/).
- Use the latest stable Scala release from `org.scala-lang:scala-library`. Reject prerelease versions and confirm ambiguity against the [official Scala download page](https://www.scala-lang.org/download/).
- Resolve the latest stable version of every plugin and dependency, then pin the exact version in the build. Never leave `<latest version>` or an open version range in a finished project.
- Use Java 21 LTS by default for local development and CI unless the project documents a reason to require a newer version.

## Build structure and launchers

- Write settings in flat `build.sbt` style, for example top-level `scalaVersion := "<version>"` rather than wrapping ordinary settings in `projectRef.settings(...)`. Multi-project builds may still use project declarations for topology, aggregation, dependencies, and plugin enablement.
- Keep both `sbt` and `sbt.bat` launcher scripts in the project root. Commit the POSIX executable bit on `sbt`; verify `sbt.bat` is present and runnable on Windows.

## sbt-mcp

- Add the latest stable sbt-mcp plugin to `project/plugins.sbt`:

```scala
addSbtPlugin("com.jamesward" % "sbt-mcp" % "<latest stable sbt-mcp version>")
```

  Resolve it from [the sbt 2 / Scala 3 plugin artifact](https://www.javadocs.dev/com.jamesward/sbt-mcp_sbt2_3/latest).
- Configure sbt-mcp with flat project settings. Choose one available port in the 5000–5999 range, check it does not conflict with another local project, and commit that stable choice rather than selecting a new random port on each run. Keep the server loopback-only because its tools can execute build tasks:

```scala
mcpEnabled := true
mcpHost := "127.0.0.1"
mcpPort := 5015 // Replace once with this project's chosen available port.
```

- Run the initialization task once to print client-specific setup guidance:

```bash
./sbt --server --no-colors --supershell=false mcpInstall
```

- `mcpInstall` prints guidance; it does **not** inspect or edit the MCP client configuration. Follow its output:
  1. Check the target config for the same server key or URL and reconcile duplicates.
  2. Add the emitted HTTP server entry to the applicable project-local config, such as `.kiro/settings/mcp.json`, `.mcp.json`, or `.cursor/mcp.json`.
  3. Start a long-lived sbt session and keep it running while the MCP client uses the server. A one-shot `mcpInstall` process does not keep the server alive.
  4. Reconnect or reinitialize the MCP client after the server is running so it reloads the tool list.
- Record the MCP server name and expected tool workflow in `AGENTS.md`. Agents should use MCP for sbt tasks and Scala/classpath symbol inspection when it is available, and clearly state when they must fall back to the project launcher.
- sbt 2.x uses a persistent daemon. After changing environment variables or JVM `-D` properties, stop it before the next build so the new process receives those values:

```bash
./sbt shutdown
```

## Compiler defaults

Use at least:

```scala
scalacOptions ++= Seq(
  "-language:strictEquality",
  "-deprecation",
  "-Werror",
)
```

When enabling strict equality:

- Derive or define `CanEqual` only for domain types that are intentionally comparable, for example `enum Status derives CanEqual`.
- Do not add a universal `CanEqual[Any, Any]` escape hatch.
- For foreign types without suitable equality evidence, compare a typed identifying property when that preserves the assertion's meaning.

## SkillsJars

- Add the latest stable SkillsJars sbt plugin to `project/plugins.sbt` before using the `Skills` configuration:

```scala
addSbtPlugin("com.skillsjars" % "skillsjars-sbt-plugin" % "<latest stable version>")
```

- Configure extraction and add the latest stable James Ward Skills dependency in `build.sbt`:

```scala
skillsJarsOutputDir := Some(file(".kiro/skills"))
libraryDependencies += "com.jamesward" % "skills" % "<latest stable version>" % Skills
```

- Follow [the SkillsJars setup instructions](https://www.skillsjars.com/setup), including:
  - Add `.kiro/skills/` (or the selected generated directory) to `.gitignore`.
  - Run `./sbt extractSkillsJars` after setup and whenever the pinned Skills dependency changes.
  - Read the relevant extracted `SKILL.md` files before working.
  - Record this repeatable workflow in `AGENTS.md`.
- Vet third-party Skills before adding them. Automated scanning is not a substitute for reviewing instructions that an agent will execute.

## Dependencies

- Explicit dependencies should normally be only the outermost dependencies the project directly uses. Do not repeat transitive dependencies merely to synchronize versions.
- If an explicit dependency must use the version supplied by a transitive dependency, use [sbt-tdepver](https://github.com/jamesward/sbt-tdepver).

## New Bootstrap

1. Create `project/build.properties` with an exact stable sbt version:

   ```properties
   sbt.version=<latest stable sbt 2.x version>
   ```

2. Add the root `sbt` and `sbt.bat` launchers and make `sbt` executable.
3. Create `project/plugins.sbt` with pinned stable sbt-mcp and SkillsJars plugin versions.
4. Create a flat `build.sbt` containing at least:

   ```scala
   name := "<project name>"
   scalaVersion := "<latest stable Scala version>"

   // sbt-mcp settings
   mcpEnabled := true
   mcpHost := "127.0.0.1"
   mcpPort := <chosen available 5000-5999 port>
   ```

5. Add the compiler defaults and Skills dependency described above.
6. Load sbt, run `mcpInstall`, manually register the emitted server configuration, start a long-lived sbt session, and reconnect the MCP client.
7. Run `extractSkillsJars`, read the relevant Skills, and apply all remaining project instructions.

## Server Projects

- Add the latest stable sbt-reload plugin:

  ```scala
  addSbtPlugin("com.jamesward" % "sbt-reload" % "<latest stable version>")
  ```

- If the project consumes frontend assets from WebJars, add the latest stable sbt-webjars plugin. Do not add it merely because the server exposes HTTP or serves independently managed static files:

  ```scala
  addSbtPlugin("org.webjars" % "sbt-webjars" % "<latest stable version>")
  ```

- Add the latest stable sbt-native-packager plugin and enable Java application packaging:

  ```scala
  addSbtPlugin("com.github.sbt" % "sbt-native-packager" % "<latest stable version>")
  ```

  ```scala
  enablePlugins(JavaAppPackaging)
  ```

### Testcontainers and test-scoped development

- Use Testcontainers for local development and integration tests that need real service dependencies such as databases or queues. Exercise the same production interfaces and repository/service layers; swap only configuration and resource-providing ZIO Layers rather than adding dev branches to production logic.
- Keep Testcontainers libraries, `AppTest`, seed data, unauthenticated dev routes, and test resource layers in `Test` scope (`% Test` dependencies and `src/test/scala`). They must not enter the production classpath or packaged application.
- Pin both the Testcontainers library version and every container image tag. Do not use floating tags such as `latest`, and do not expose fixed host ports unless the test contract requires them.
- Make resource ownership explicit:
  - By default, build the container in `ZLayer.scoped` with `ZIO.acquireRelease`; start and stop it with blocking effects, and keep the layer's scope alive for the full test or development-server lifetime.
  - A deliberately shared container per forked test JVM is valid when startup/migration cost is significant. Document that exception, rely on Testcontainers/Ryuk or an explicit JVM-level owner for container cleanup, and still return per-spec isolated resources from a scoped layer.
  - Close connection pools, drop cloned databases/namespaces, and release other per-spec resources in scope finalizers. Cleanup failures should be logged rather than silently discarded.
- Preserve test isolation under parallel execution. Do not let specs share mutable database/schema state. For databases, prefer a fresh database/schema per spec; when setup is expensive, a migrated template may be prepared once and cloned for each spec. Synchronize only startup or clone operations known to be unsafe in the target Docker environment rather than serializing the whole suite.
- Wrap blocking container, JDBC, and administrative calls in `ZIO.attemptBlocking`. Surface startup failures clearly; do not silently skip Docker-backed tests unless the project explicitly defines and documents an availability gate.
- Fork application/test JVMs when the container lifecycle or JVM options require process isolation, and select separate production and development mains:

  ```scala
  fork := true
  Compile / mainClass := Some("App")
  Test / mainClass := Some("AppTest")
  ```

- Put `AppTest` in `src/test/scala`. It should compose the normal application routes/services with Testcontainer-backed layers and safe dev-only configuration. Run it through Test scope so production `App` remains unchanged:

  ```bash
  ./sbt api/Test/run
  ./sbt '~api/Test/runReload'
  ```

  In a single-module server, use `./sbt Test/run` or a project alias such as `dev` that expands to `~Test/runReload`. Document the exact module commands in `AGENTS.md`.

## Library Projects

- Add the latest stable sbt-ci-release plugin and declare semantic versioning:

  ```scala
  addSbtPlugin("com.github.sbt" % "sbt-ci-release" % "<latest stable version>")
  ```

  ```scala
  versionScheme := Some("semver-spec")
  ```

## Validation

After creating or changing the build:

1. Run `./sbt shutdown` first when environment variables, JVM `-D` properties, or daemon-sensitive configuration changed.
2. Run `./sbt extractSkillsJars` and verify the generated Skills are ignored.
3. Compile main and test sources with the strict compiler options enabled.
4. Run the complete non-metered test suite. Prefer `testFull` in CI when a guaranteed full sbt 2 test run is required rather than an incremental cached run.
5. For server applications, run `stage` and a minimal startup or health-check smoke test.
6. Keep paid, metered, or destructive integration suites out of the default validation path; gate them explicitly and run them only when requested.
7. Fix all validation failures before declaring the project compliant. If a required check cannot run, document the blocker and the closest successful check.

A typical application validation sequence is:

```bash
./sbt shutdown
./sbt extractSkillsJars
./sbt "Test / compile; testFull; stage"
```

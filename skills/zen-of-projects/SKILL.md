---
name: zen-of-projects
description: Whether creating new projects or bringing existing ones into compliance, do these things.
---

# Automation & Agent Guidance

## One source of truth

Say each thing once and reference it elsewhere:

- **This Skill** owns general conventions, compatibility rules, the validation sequence, and the daily routine.
- **`AGENTS.md`** owns project facts only: what the project is, which Skills apply, exact build/test/dev commands, the MCP server name and port, gated or paid tests, and every **exception** to this Skill together with its reason (for example a required Java version, an allowed prerelease, or an intentional pin). Do not restate this Skill's general rules in `AGENTS.md`; write "Follow the `zen-of-projects` Skill" and record only what differs or is unique.
- **`.factory/DAILY.md`** is the bootstrap below, verbatim. It contains no project-specific content.

When they disagree, this Skill wins for general conventions and `AGENTS.md` wins for its documented exceptions. Fix the drift in the same change rather than leaving two versions.

## `.factory/DAILY.md`

Use exactly this content:

````markdown
# Daily Routine

If there are other open PRs for this work, update that PR instead of creating a new one.

0. Load the project's MCP tools before anything else. `AGENTS.md` names the sbt-mcp server
   (`sbt-mcp-<project>`). In Claude Code its tools are deferred, so load them with ToolSearch
   (search for the server name). They include `sbt-task` for sbt commands and the javadocs.dev
   tools such as `get_latest_version`. Use them for the rest of the run, and fall back to `./sbt`
   and `curl` only when they are unavailable. Say which one you used.
1. Update the Skills dependency. It is pinned in `build.sbt` as
   `"com.jamesward" % "skills" % "<version>" % Skills`. List every pin (some projects also pin it
   in an `example/` build) with:

   ```bash
   grep -rn '"com.jamesward" % "skills"' --include='*.sbt' . | grep -v -e /target/ -e /src/sbt-test/
   ```

   Get the latest release with `get_latest_version` (group `com.jamesward`, artifact `skills`). Without
   MCP, ask Maven Central itself, not a mirror (mirrors lag new releases):

   ```bash
   curl -fsS --retry 5 --retry-delay 10 --retry-all-errors https://repo.maven.apache.org/maven2/com/jamesward/skills/maven-metadata.xml | sed -n 's:.*<release>\(.*\)</release>.*:\1:p'
   ```

   Maven Central can rate-limit cloud sessions (HTTP 429); the retries cover that. Set every pin
   to the version it prints.
2. Run `reload; extractSkillsJars` with the sbt-mcp `sbt-task` tool, or `./sbt extractSkillsJars`. `.kiro/skills/` is gitignored, so it does not exist until this
   runs. If sbt cannot download artifacts (for example HTTP 429 or a proxy 403), stop and report the
   error instead of changing resolvers.
3. Read `.kiro/skills/*zen-of-projects*/SKILL.md` and follow its "Daily Routine" section, using
   `AGENTS.md` for this project's commands and documented exceptions.
````

The skills dependency is updated first so the rest of the run follows the newest version of this Skill.

## Daily Routine

1. **Find the rolling PR first.** Look for an open PR whose title starts with `Daily maintenance:`. Also match the legacy prefix `Agent alignment:`:

   ```bash
   gh pr list --state open --search 'in:title "Daily maintenance:" OR in:title "Agent alignment:"'
   ```

   Some cloud environments proxy GitHub and allow only a limited set of GraphQL operations. If a `gh` command fails with a GraphQL error, use the REST API instead, for example `gh api 'repos/{owner}/{repo}/pulls?state=open' --jq '.[] | select(.title | test("^(Daily maintenance|Agent alignment):")) | [.number, .head.ref, .created_at] | @tsv'`.

   Reuse the oldest match and check out its branch. Close any other matches as duplicates. Create a branch only when there is no match. New PRs use the `Daily maintenance:` prefix and the `daily-maintenance` label, and target the repository's default branch.
2. **Preserve existing work.** Build on the PR branch and any uncommitted changes. Never reset, force-push over, or discard them.
3. **Update dependencies.** Move sbt, Scala, every plugin, every dependency, and the GitHub Actions versions in `.github/workflows` to the latest stable version. Apply the compatibility rules below and the exceptions in `AGENTS.md`. Fold in any open dependency-bump PRs, then close them.
   - **Check MCP first.** At the start of the run, note whether the project's sbt-mcp server (named in `AGENTS.md`) and the javadocs.dev tools are available, and say so in the PR description. In Claude Code, MCP tools can be deferred and won't appear in your tool list until you load them with ToolSearch. Search for the server name before concluding that they are missing. If the sbt-mcp server is configured but missing, read `/tmp/sbt-mcp-stdio.log` and `/tmp/sbt-mcp-server.log` and include the cause.
   - **Find versions with javadocs.dev MCP.** Use `get_latest_version`, which skips prereleases, through the project's sbt-mcp server or the javadocs.dev connector (`https://www.javadocs.dev/mcp`). Use `list_javadoc_symbols`, `get_javadoc_symbol`, `list_source_files` and `get_source_file` to read a new API when migrating code.
   - **Fallback without MCP: Maven metadata.** Maven Central rate-limits shared cloud IPs (HTTP 429), so make few requests, retry, and filter out prereleases. Never use a mirror for this, because mirrors lag new releases.

     ```bash
     latest_stable() { local g="${1%%:*}" a="${1##*:}"; curl -fsS --retry 5 --retry-delay 10 --retry-all-errors "https://repo.maven.apache.org/maven2/${g//.//}/$a/maven-metadata.xml" | grep -o '<version>[^<]*' | sed 's/<version>//' | grep -E '^[0-9]' | grep -viE -- '[-.]?(m|rc|alpha|beta|snapshot|cr|ea|preview|dev|pre)[-.]?[0-9]*([-.]|$)' | sort -V | tail -1; }
     latest_stable org.scala-sbt:sbt   # prints e.g. 2.0.9; empty output means only prereleases exist
     ```

   - **GitHub Actions versions.** Use `git ls-remote --tags https://github.com/<owner>/<action>`, not the GitHub API. Cloud sessions can only call the API for the repositories attached to the session.
4. **Align.** Bring the project into line with this Skill, using the version extracted in the bootstrap step. Update `AGENTS.md` where code or workflow has drifted, and remove any text that restates this Skill.
5. **Validate.** Run the Validation sequence below with the project's commands from `AGENTS.md`. If a bump fails, try to fix it: migrate to the new API, apply the compatibility rules, and re-run validation.
6. **Publish and merge.** Commit to the rolling PR branch, push, and rewrite the PR description to summarize all changes on the branch. If nothing changed, take no action. Then:
   - **Merge** (`gh pr merge --squash --delete-branch`, or `gh api -X PUT 'repos/{owner}/{repo}/pulls/<number>/merge' -f merge_method=squash` if GraphQL is blocked) when validation and the PR's CI checks pass, and the branch contains only dependency bumps plus the fixes they needed.
   - **Request human review** and do not merge when the branch also changes public APIs, behavior, or alignment beyond version bumps, or when `AGENTS.md` requires human review for the kind of change involved. Add the `needs-human` label and say what needs a decision.
   - **Escalate** when a failure cannot be fixed. Leave the PR open and unmerged with the `needs-human` label. Comment with the failing bump, the error, and what was tried. Do not revert the bump just to get a green build.

# Scala Projects

## Versions and Java

- Use the latest **stable, non-prerelease** sbt 2.x release. Do not select versions containing qualifiers such as `RC`, `M`, or `SNAPSHOT` merely because Maven metadata sorts them as latest. Resolve versions through [javadocs.dev](https://www.javadocs.dev/org.scala-sbt/sbt), then confirm ambiguous results against the [official sbt download page](https://www.scala-sbt.org/download/).
- Use the latest stable Scala release from `org.scala-lang:scala-library`. Reject prerelease versions and confirm ambiguity against the [official Scala download page](https://www.scala-lang.org/download/).
- Resolve the latest stable version of every plugin and dependency, then pin the exact version in the build. Never leave `<latest version>` or an open version range in a finished project.
- Use Java 21 LTS by default for local development and CI unless the project documents a reason to require a newer version.

## Compatibility rules for upgrades

"Latest stable" is constrained by these rules. Automated version bumps often break them.

- **sbt plugins use sbt's Scala.** An sbt 2 plugin's Scala 3 `scalaVersion` must equal the `scala3-library_3` version in the pinned `org.scala-sbt:sbt` POM (3.8.4 for sbt 2.0.9). Never bump a plugin's Scala on its own: newer TASTy cannot be read by sbt's build compiler. Plugins cross-built for sbt 1 use the latest Scala 2.12.x with `-deprecation -Xfatal-warnings`, because 2.12 has no strict equality. Scripted test fixtures follow the same rule.
- **Java 25 bytecode forces Java 25.** Some libraries publish class file version 69, for example Kyo `1.0.0-RC*` and `html-to-markdown` 3.x. On Java 21 they fail with `UnsupportedClassVersionError`. Projects that depend on them use Java 25 and record the reason in `AGENTS.md`. After a failed build on the wrong JDK, run `clean` before validating again.
- **JDK 24+-only JVM flags.** Flags such as `--sun-misc-unsafe-memory-access=allow` stop Java 21 from starting. Keep them out of `.sbtopts` and `.jvmopts`. Prefer `-Dsun.misc.unsafe.memory.access=allow`, which works on JDK 21 through 25, including in native-packager's `application.ini`. Otherwise add the flag conditionally inside `Def.uncached { ... }`, because sbt 2 caches JDK-dependent task results across JDK switches.
- **Prereleases only when there is no stable release.** Examples are `dev.zio:zio-direct` 1.0.0-RC7 and Kyo 1.0.0-RC*. Take the newest such release only if the tests pass, and record the exception in `AGENTS.md`.
- **Intentional pins.** Keep a version that `AGENTS.md` documents as intentionally pinned, such as one used by a bug reproducer.
- **Deprecations break the build under `-Werror`.** Migrate to the replacement API rather than suppressing the warning. For example, replace `ZIO.done(exit)` with `exit`, and replace a library's deprecated alias with its new name.
- **Archived or renamed artifacts.** When a dependency is archived or superseded (for example `zio-bedrock-converse` replaced by `zio-bedrock`), migrate to the successor. Do not keep bumping the old artifact.
- If a bump breaks the build and cannot reasonably be fixed, follow the escalation step in the Daily Routine.

## Build structure and launchers

- Write settings in flat `build.sbt` style, for example top-level `scalaVersion := "<version>"` rather than wrapping ordinary settings in `projectRef.settings(...)`. Multi-project builds may still use project declarations for topology, aggregation, dependencies, and plugin enablement.
- Keep both `sbt` and `sbt.bat` launcher scripts in the project root. Commit the POSIX executable bit on `sbt`; verify `sbt.bat` is present and runnable on Windows.

## Resolvers

- Use sbt's default resolvers: Maven Central plus the `local` Ivy repository. Never commit resolver configuration to a project. That rules out a `project/repositories` file, `-Dsbt.repository.config` or `-Dsbt.override.build.repos` in `.sbtopts` or `.jvmopts`, mirror URLs, and `Resolver.mavenLocal`, `Resolver.file`, or `file://` resolvers in the build.
- To test against an unpublished artifact, use `publishLocal`. The `local` Ivy repository is already a default resolver, so no resolver change is needed. If a test really needs another resolver, keep it ephemeral: pass it for that one invocation (for example `./sbt 'set resolvers += Resolver.mavenLocal' test`) and make sure it is not in the committed diff.
- Mirrors are environment configuration. An automated environment that gets rate-limited by Maven Central can put a mirror first in the user-level `~/.sbt/repositories`, with Maven Central as the fallback, and add `-Dsbt.override.build.repos=true` to the user-level `~/.config/sbt/sbtopts`:

  ```text
  [repositories]
    local
    google-maven-central: https://maven-central.storage-download.googleapis.com/maven2/
    maven-central
  ```

  The sbt launcher and the build then download from the Google mirror and fall back to Maven Central for anything the mirror lacks, such as a release from the last few hours. With `sbt.override.build.repos=true`, sbt ignores `resolvers` declared in the build. Add any extra repository a project needs to the environment's list instead.

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

## Maven Central Badge

Every project that publishes an artifact to Maven Central, including libraries and sbt plugins, puts a javadocs.dev badge directly under the title in `README.md`:

```markdown
[![javadocs.dev](https://www.javadocs.dev/<groupId>/<artifactId>/badge.svg)](https://www.javadocs.dev/<groupId>/<artifactId>/latest)
```

- Use the published artifactId including its suffix: `_3` for Scala 3 libraries (for example `com.jamesward/zio-mavencentral_3`), and `_sbt2_3` for sbt 2 plugins (for example `com.jamesward/sbt-reload_sbt2_3`). For plugins cross-built for sbt 1 and sbt 2, use the `_sbt2_3` artifact.
- Check that `https://www.javadocs.dev/<groupId>/<artifactId>/latest` resolves before committing the badge.
- When a build publishes several modules, add one badge per primary published module.

## Validation

After creating or changing the build:

1. Run `./sbt shutdown` first when environment variables, JVM `-D` properties, or daemon-sensitive configuration changed.
2. Run `./sbt extractSkillsJars` and verify the generated Skills are ignored.
3. Compile main and test sources with the strict compiler options enabled.
4. Run the complete non-metered test suite. Testcontainers needs a Docker daemon. If `docker info` fails in a cloud session (where Docker is installed but not running), start it with `dockerd > /tmp/dockerd.log 2>&1 &` and wait until `docker info` succeeds. Don't skip Docker-backed tests because of it. Prefer `testFull` in CI when a guaranteed full sbt 2 test run is required rather than an incremental cached run.
5. For server applications, run `stage` and a minimal startup or health-check smoke test.
6. Keep paid, metered, or destructive integration suites out of the default validation path; gate them explicitly and run them only when requested.
7. Fix all validation failures before declaring the project compliant. If a required check cannot run, document the blocker and the closest successful check.

A typical application validation sequence is:

```bash
./sbt shutdown
./sbt extractSkillsJars
./sbt "Test / compile; testFull; stage"
```

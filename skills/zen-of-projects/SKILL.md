---
name: zen-of-projects
description: Whether creating new projects or bringing existing ones into compliance, do these things.
---

# Scala Projects

- Use latest release sbt 2.x from https://www.javadocs.dev/org.scala-sbt/sbt
- build.sbt should be written in a flat style (i.e. top level `scalaVersion := "<version>"` instead of a `projectref.settings`)
- The sbt and sbt.bat launcher scripts should be in the project root and be executable
- All projects need the sbt-mcp plugin: `addSbtPlugin("com.jamesward" % "sbt-mcp" % "<latest sbt-mcp version>")` Latest version from: http://javadocs.dev/com.jamesward/sbt-mcp_sbt2_3/latest
- sbt-mcp needs to be configured in build.sbt with:
```
mcpEnabled := true
mcpPort    := <random 5000-5999 integer>
```
- sbt-mcp needs to be initialized once with: `./sbt --server --no-colors --supershell=false mcpInstall`
- IMPORTANT: sbt 2.x uses a persistent daemon so env vars and `-D` params do not take effect until the sbt server is restarted i.e. `./sbt shutdown`
- Minimum Scala defaults:
```
scalacOptions ++= Seq(
  "-language:strictEquality",
  "-deprecation",
  "-Werror",
)
``` 
- All projects need James Ward's Skills:
```
libraryDependencies += "com.jamesward" % "skills" % "<latest version>" % Skills
```
- Initialize the SkillsJars by following instructions at: https://www.skillsjars.com/setup

## New Bootstrap

- Create a `project/build.properties` containing `sbt.version=<latest sbt version>` (with the actual latest sbt version)
- Create the `build.sbt` with at least:
```
name := "<project name>"

# sbt-mcp settings
```
- Then using the sbt-mcp server, get the latest release scala `groupId=org.scala-lang artifactId=scala-library` and set it in `build.sbt` `scalaVersion := "<latest version>"`
- Apply all of the project instructions above

## Server Projects

- Use the sbt-reload plugin: `addSbtPlugin("com.jamesward" % "sbt-reload" % "<latest version>")`
- If the server has web stuff, use the sbt-webjars plugin: `addSbtPlugin("org.webjars" % "sbt-webjars" % "<latest version>")`
- Use sbt-native-packager `addSbtPlugin("com.github.sbt" % "sbt-native-packager" % "<latest version>")` and `enablePlugins(JavaAppPackaging)` in the `build.sbt`

## Library Projects

- Use sbt-ci-release: `addSbtPlugin("com.github.sbt" % "sbt-ci-release" % "<latest version>")` with `versionScheme := Some("semver-spec")` in `build.sbt`


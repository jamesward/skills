# James Ward's Agent Skills

![javadocs.dev](https://www.javadocs.dev/com.jamesward/skills/badge.svg)

[Agent Skills](https://agentskills.io/) that tell coding agents how James Ward writes code and runs projects.

## Skills

| Skill | Use it for |
| --- | --- |
| [`zen-of-james`](skills/zen-of-james/SKILL.md) | Design principles for any language: make illegal states unrepresentable, no mutability, domain types, effects as values, and the test onion. |
| [`zen-of-scala`](skills/zen-of-scala/SKILL.md) | Scala 3 and ZIO idioms: syntax and style, effect composition, error handling, resources and `Scope`, and testing. |
| [`zen-of-projects`](skills/zen-of-projects/SKILL.md) | Setting up a project or bringing one into line: agent guidance (`AGENTS.md`, MCP, Skills), builds for sbt, Maven, Gradle and Node.js, servers, libraries, buildpacks, and the maintenance and incident routines. |
| [`zen-of-websites`](skills/zen-of-websites/SKILL.md) | Maintained static websites: spec-driven content, contribution triage, publishing, SEO, agent readiness and Lighthouse checks. |

## Use them

### Over MCP

The MCP server at `https://mcp.jamesward.com/` ([jamesward/mcp](https://github.com/jamesward/mcp)) serves the Skills on this repo's `main` branch (read when the server starts, so a change shows up within about a day) as `skill://jamesward/skills/<name>/SKILL.md`. For example, in Kiro's `.kiro/settings/mcp.json`:

```json
{
  "mcpServers": {
    "jamesward": {
      "type": "http",
      "url": "https://mcp.jamesward.com/"
    }
  }
}
```

Or in Claude Code:

```bash
claude mcp add --transport http jamesward https://mcp.jamesward.com/
```
### As a SkillsJar

The Skills are published to Maven Central as `com.jamesward:skills` (the badge above shows the latest version; use it for `VERSION`). Add the SkillsJars plugin for your build as the [SkillsJars setup guide](https://www.skillsjars.com/setup) describes, then add the dependency.

sbt (`build.sbt`):

```scala
skillsJarsOutputDir := Some(file(".kiro/skills"))
libraryDependencies += "com.jamesward" % "skills" % "VERSION" % Skills
```

Maven, inside the `com.skillsjars:maven-plugin` plugin's `<dependencies>` in `pom.xml`:

```xml
<dependency>
    <groupId>com.jamesward</groupId>
    <artifactId>skills</artifactId>
    <version>VERSION</version>
</dependency>
```

Gradle (`build.gradle.kts`), with the `com.skillsjars.gradle-plugin` plugin:

```kotlin
dependencies {
    testRuntimeOnly("com.jamesward:skills:VERSION")
}
```

Without a JVM build, use [sjm](https://github.com/skillsjars/skillsjars-cli):

```bash
sjm --dir .kiro/skills install com.jamesward:skills
```

Then extract them into your agent's skills directory (`./sbta extractSkillsJars`, `./mvnw skillsjars:extract -Ddir=.kiro/skills` or `./gradlew extractSkillsJars -Pdir=.kiro/skills`).

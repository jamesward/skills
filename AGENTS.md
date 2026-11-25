# AGENTS.md

James Ward's Agent Skills (`skills/<name>/SKILL.md`). [`README.md`](README.md) describes the Skills and how users get them. They reach users two ways:

- **SkillsJar:** the Maven build (`pom.xml`) publishes them to Maven Central as `com.jamesward:skills`.
- **Skills over MCP:** [jamesward/mcp](https://github.com/jamesward/mcp), at `https://mcp.jamesward.com/`, reads `skills/` on this repo's `main` branch when it starts and serves them as `skill://jamesward/skills/<name>/SKILL.md`. A skill it can't serve (invalid frontmatter, over the size limits) fails that server's startup, so a broken skill merged to `main` takes it down at its next restart.

When adding, renaming or removing a Skill, update its row in the README's Skills table.

## Skills

Follow the `zen-of-projects` Skill.

- For Scala 3 and ZIO code (writing or reviewing it), follow `zen-of-scala`.
- For design, domain modeling, effects and testing strategy, follow `zen-of-james`.

They are this repo's own `skills/` directory (the unreleased source of `com.jamesward:skills`), linked as `.kiro/skills` and `.claude/skills` (symlinks to `../skills`). Read the relevant ones before working.

## MCP

The MCP server is `javadocs` (`https://www.javadocs.dev/mcp`), configured in `.kiro/settings/mcp.json` and `.mcp.json` (enabled in `.claude/settings.json`). In Claude Code its tools are deferred: load them with ToolSearch.

## Build, Test & Dev Workflow

Full validation (CI runs the same):

```bash
./mvnw -q -B package
```

- The build packages only `skills/`. Check the jar with `unzip -l target/skills-*.jar` after changing it.
- No tests need Docker or paid services.

## Maintenance routine

`.factory/MAINTENANCE.md`.

## Exceptions

- No Skills dependency: this repo is the source of `com.jamesward:skills`, so the routine's steps to update and extract `com.jamesward:skills` don't apply. Agents read `skills/` through the `.kiro/skills` and `.claude/skills` symlinks.

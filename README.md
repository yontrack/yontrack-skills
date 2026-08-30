# Yontrack skills

Agent skills for [Yontrack](https://docs.yontrack.com/yontrack/ref/index.html), packaged as a Claude Code plugin.

## Installing

```
/plugin marketplace add yontrack/yontrack-skills
/plugin install yontrack-skills@yontrack
```

## Skills

- **[`yontrack-github-actions`](skills/yontrack-github-actions/SKILL.md)** — reporting a repository's CI to Yontrack from GitHub Actions:
  installing and configuring the CLI, declaring validations and promotions in `.yontrack/ci.yaml`,
  recording validation runs with their run info, and reading build data back.
- **[`yontrack-auto-versioning`](skills/yontrack-auto-versioning/SKILL.md)** — propagating a version between repositories on a promotion: declaring
  auto-versioning on the target branch so a promoted build updates a version somewhere else.

Agents reach these on their own; you can also invoke one by name.

## Adding a skill

One directory per skill under `skills/`, each with a `SKILL.md` carrying `name` and `description`
frontmatter, and an entry in the `skills` array of `.claude-plugin/plugin.json`.

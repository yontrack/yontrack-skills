# Yontrack skills

Agent skills for [Yontrack](https://doc.yontrack.com), packaged as a Claude Code plugin.

## Installing

```
/plugin marketplace add yontrack/yontrack-skills
/plugin install yontrack-skills@yontrack
```

## Skills

- **`yontrack-github-actions`** — reporting a repository's CI to Yontrack from GitHub Actions:
  installing and configuring the CLI, declaring validations and promotions in `.yontrack/ci.yaml`,
  recording validation runs, and reading build data back.

Agents reach these on their own; you can also invoke one by name.

## Adding a skill

One directory per skill under `skills/`, each with a `SKILL.md` carrying `name` and `description`
frontmatter, and an entry in the `skills` array of `.claude-plugin/plugin.json`.

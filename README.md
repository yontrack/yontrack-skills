# Yontrack skills

Agent skills for [Yontrack](https://yontrack.com), packaged as a Claude Code plugin. The skills follow the
[Yontrack documentation](https://docs.yontrack.com/yontrack/ref/index.html).

## Installing

```
/plugin marketplace add yontrack/yontrack-skills
/plugin install yontrack-skills@yontrack
```

## Skills

- **[`yontrack-github-actions`](skills/yontrack-github-actions/SKILL.md)** — reporting a repository's CI to Yontrack from GitHub Actions:
  installing and configuring the CLI, declaring validations and promotions in `.yontrack/ci.yaml`,
  recording validation runs with their run info, and reading build data back.
- **[`yontrack-bitbucket-pipelines`](skills/yontrack-bitbucket-pipelines/SKILL.md)** — the same from Bitbucket Pipelines:
  installing the pinned CLI in each step, registering the build once and handing it to later steps as an
  artifact, and recording validations from `after-script` with their run info.
- **[`yontrack-gitlab-ci`](skills/yontrack-gitlab-ci/SKILL.md)** — the same from GitLab CI/CD: installing the
  pinned CLI in each job, registering the build once and handing it to later jobs as a `dotenv` artifact, and
  recording validations from `after_script` with their run info.
- **[`yontrack-auto-versioning`](skills/yontrack-auto-versioning/SKILL.md)** — propagating a version between repositories on a promotion: declaring
  auto-versioning on the target branch so a promoted build updates a version somewhere else.
- **[`yontrack-changelogs`](skills/yontrack-changelogs/SKILL.md)** — rendering a change log in a template: on a promotion
  notification, in a workflow node, or in an auto-versioning pull request body. Which renderable measures which
  interval, plain versus semantic (conventional-commit) form, and why one comes out empty.
- **[`yontrack-agent-conduct`](skills/yontrack-agent-conduct/SKILL.md)** — how an AI agent behaves on a Yontrack delivery
  record, whatever the agent: identifying itself with its agent token and session, reading its own policy and a
  build's readiness before proposing a merge, a promotion or a deployment, recording evidence as validation runs,
  and stopping at the gates a person holds. Plain Markdown with nothing Claude-specific: point to it from an
  `AGENTS.md` to give the same conduct to Codex, Copilot or any other agent.

Agents reach these on their own; you can also invoke one by name.

## Adding a skill

One directory per skill under `skills/`, each with a `SKILL.md` carrying `name` and `description`
frontmatter, and an entry in the `skills` array of `.claude-plugin/plugin.json`.

---
name: yontrack-github-actions
description: Report a repository's CI to Yontrack from GitHub Actions — register the build once, link every job to it, declare validations and promotions in .yontrack/ci.yaml, and record validation runs. Use when wiring a workflow to Yontrack, adding a validation stamp or a promotion level, or reading Yontrack build data from CI.
---

# Reporting GitHub Actions to Yontrack

Yontrack records what a delivery pipeline did: each **build**, the **validations** that ran against it,
and the **promotions** it earned. A workflow wired to Yontrack answers "is this version fit to deploy?"
without anyone reading a log.

One build per workflow run. **One job registers it; every other job links to it.**

- [`ontrack-github-actions-cli-config`](https://github.com/nemerosa/ontrack-github-actions-cli-config)
  registers the build — exactly once per run.
- [`ontrack-github-actions-cli-library/ontrack-cli-workflow-build`](https://github.com/nemerosa/ontrack-github-actions-cli-library)
  links a job to the build already registered, finding it by commit.
- The bare `yontrack` CLI records everything after that.

## Steps

### 1. Confirm the credentials

```bash
gh variable list   # expect YONTRACK_URL
gh secret list     # expect YONTRACK_TOKEN
```

Both are needed by the registering job and by every linking job. When either is missing, stop and ask — a
token is minted in the Yontrack UI and only the human can do it.

### 2. Compute the version once, at workflow level

```yaml
env:
  VERSION: "1.0.${{ github.run_number }}"
```

The build is registered with this version, and the artefacts the run publishes carry the same one. A job
that computes its own version describes a build nobody else is reporting to.

### 3. Declare validations and promotions

Write `.yontrack/ci.yaml` (see [Configuration file](#configuration-file)). Every validation the workflow
records is declared here, and at least one promotion gates on them — a validation with no promotion above
it is a stamp nobody reads.

### 4. Register the build, in a job of its own

```yaml
jobs:
  yontrack:
    name: Yontrack
    runs-on: ubuntu-latest
    outputs:
      project: ${{ steps.config.outputs.projectName }}
    steps:
      - uses: actions/checkout@v4

      - name: Register the build
        id: config
        uses: nemerosa/ontrack-github-actions-cli-config@v1.3.0
        env:
          YONTRACK_URL: ${{ vars.YONTRACK_URL }}
          YONTRACK_TOKEN: ${{ secrets.YONTRACK_TOKEN }}
        with:
          version: 5.1.0
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

This job runs first and does nothing else. It creates the project, the branch and the build, and exports
the project name for the jobs that link to it.

Pin the CLI `version`. An unpinned CLI lets a Yontrack release turn a green pipeline red in a repository
nobody touched. The token keeps the download off the anonymous GitHub rate limit.

### 5. Link every other job to that build

```yaml
  build:
    needs: yontrack
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Link to the Yontrack build
        uses: nemerosa/ontrack-github-actions-cli-library/ontrack-cli-workflow-build@v1.0.3
        with:
          project: ${{ needs.yontrack.outputs.project }}
          yontrack-url: ${{ vars.YONTRACK_URL }}
          yontrack-token: ${{ secrets.YONTRACK_TOKEN }}
          version: 5.1.0
```

The action installs the CLI, finds the build registered against `github.sha`, and exports
`YONTRACK_PROJECT_NAME`, `YONTRACK_BRANCH_NAME` and `YONTRACK_BUILD_NAME` for the rest of that job — so
every later `yontrack` command needs no `--project`, `--branch` or `--build`.

`needs: yontrack` is what makes it work: the action searches for a build that must already exist. Calling
`-config` in more than one job instead registers a *different* build per job, and validations then fail with
`Build not found`.

### 6. Record a validation after each meaningful step

Give the step an `id`, then validate on its outcome:

```yaml
      - name: Build
        id: build
        run: ./gradlew build

      - if: ${{ steps.build.outcome != '' }}
        run: |
          yontrack validate \
            --validation build \
            junit --pattern "build/test-results/**/*.xml" --fail-when-no-results
```

`steps.<id>.outcome` is empty only when the step was skipped, so this records `FAILED` when the build
broke — which is the case worth seeing in Yontrack. Prefer typed data over a bare status wherever the step
produces it (see [Validation data](#validation-data)).

### 7. Record the version, and verify

Yontrack names the build itself. Attach the version that was published as a property, so the build can be
traced back to a deployable artefact:

```bash
yontrack build set-property release "$VERSION"
```

Then read it back after a real run:

```bash
yontrack build search --with-promotion BRONZE --count 1 --output json
```

Done when every declared validation appears on one build and the promotion is earned.

## Configuration file

`.yontrack/ci.yaml`, processed as a template then parsed. `defaults` applies to every branch; `custom`
entries add to it per branch.

```yaml
version: v1
configuration:
  defaults:
    branch:
      validations:
        build:
          tests: {}      # typed stamp: accepts test counts
        docker: {}       # plain stamp: PASSED / FAILED
      promotions:
        BRONZE:
          validations:
            - build
            - docker
  custom:
    - conditions:
        branch: main
      branch:
        validations:
          release: {}
        promotions:
          SILVER:
            validations:
              - release
```

The minimal file is `version: v1` with `configuration: {}` — Yontrack then creates project, branch and
build from the CI context and nothing else.

Both the file and the CLI carry more than this: properties, notifications, workflows, auto-versioning,
`@path` file inclusion, `--var` and `--env` template variables, Sprig functions. Reach for the
[Yontrack reference documentation](https://docs.yontrack.com/yontrack/ref/index.html) rather than guessing
at the schema.

## Validation data

A typed stamp shows a trend in Yontrack where a bare status shows a light. The subcommand follows the
common flags:

```bash
yontrack validate --validation build junit --pattern "build/test-results/**/*.xml" --fail-when-no-results
yontrack validate --validation tests tests --passed 20 --skipped 2 --failed 1
yontrack validate --validation security chml --critical 0 --high 2 --medium 25 --low 1214
yontrack validate --validation coverage percentage --value 87
```

`junit` reads the XML itself and is the one to reach for with a JVM, Node or Python test run. The stamp
must be declared with a matching type in `.yontrack/ci.yaml` — `tests: {}` above.

## Beyond validation

Once a job is linked, the same job can read and annotate the build:

```bash
yontrack build set-property release "1.2.3"
yontrack build search --with-promotion RELEASE --count 1 --output json --accept-not-found
yontrack build changelog semantic --from-promotion RELEASE --emojis --issues
```

This is why the workflow calls the CLI directly rather than the
[`ontrack-github-actions-cli-validation`](https://github.com/nemerosa/ontrack-github-actions-cli-validation)
action: one vocabulary covers validation and everything past it, where the action stops at validation and
leaves the workflow mixing two idioms.

Likewise `ontrack-github-actions-cli-setup` is the previous generation — `node16`, an `ontrack-cli`
executable, and a dozen action inputs in place of a configuration file. New workflows use `-config`
to register and `ontrack-cli-workflow-build` to link; keep it that way.

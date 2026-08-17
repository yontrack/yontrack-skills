---
name: yontrack-github-actions
description: Report a repository's CI to Yontrack from GitHub Actions — install and configure the Yontrack CLI, declare validations and promotions in .yontrack/ci.yaml, record validation runs, and query builds by promotion. Use when wiring a workflow to Yontrack, adding a validation stamp or a promotion level, or reading Yontrack build data from CI.
---

# Reporting GitHub Actions to Yontrack

Yontrack records what a delivery pipeline did: each **build**, the **validations** that ran against it,
and the **promotions** it earned. A workflow wired to Yontrack answers "is this version fit to deploy?"
without anyone reading a log.

Two moving parts: the [`ontrack-github-actions-cli-config`](https://github.com/nemerosa/ontrack-github-actions-cli-config)
action installs and configures the `yontrack` CLI and registers the build, then the bare CLI records
everything after that.

## Steps

### 1. Confirm the credentials

```bash
gh variable list   # expect YONTRACK_URL
gh secret list     # expect YONTRACK_TOKEN
```

Both are read by the config action. When either is missing, stop and ask — a token is minted in the
Yontrack UI and only the human can do it.

### 2. Give every job the same build name

`yontrack ci config` derives the build name from the environment, and `VERSION` wins when it is set.
Put it at workflow level so every job agrees:

```yaml
env:
  VERSION: "0.1.${{ github.run_number }}"
```

Jobs that previously computed a version now consume `$VERSION`. Without this, jobs that each configure
the CLI register *different builds* and the validations scatter across them.

### 3. Declare validations and promotions

Write `.yontrack/ci.yaml` (see [Configuration file](#configuration-file)). Every validation the workflow
records is declared here, and at least one promotion gates on them — a validation with no promotion above
it is a stamp nobody reads.

### 4. Configure the CLI in each validating job

```yaml
      - name: Yontrack configuration
        uses: nemerosa/ontrack-github-actions-cli-config@v1.3.0
        env:
          YONTRACK_URL: ${{ vars.YONTRACK_URL }}
          YONTRACK_TOKEN: ${{ secrets.YONTRACK_TOKEN }}
        with:
          version: 5.1.0
```

Pin `version`. An unpinned CLI lets a Yontrack release turn a green pipeline red in a repository nobody
touched, and pinning also drops the GitHub release lookup — which matters once several jobs each install
the CLI.

The action exports `YONTRACK_PROJECT_NAME`, `YONTRACK_BRANCH_NAME` and `YONTRACK_BUILD_NAME` for the rest
of *that job*, and every later `yontrack` command reads them instead of `--project`, `--branch` and
`--build`. Their scope is the job, so each job that validates configures the CLI itself. Concurrent jobs
registering the same build is safe.

### 5. Record a validation after each meaningful step

Give the step an `id`, then validate on its outcome:

```yaml
      - name: Build
        id: build
        run: ./gradlew build

      - if: ${{ steps.build.outcome != '' }}
        run: |
          yontrack validate \
            --validation build \
            --status ${{ steps.build.outcome == 'success' && 'PASSED' || 'FAILED' }}
```

`steps.<id>.outcome` is empty only when the step was skipped, so this records `FAILED` when the build
broke — which is the case worth seeing in Yontrack. Prefer typed data over a bare status wherever the
step produces it (see [Validation data](#validation-data)).

### 6. Verify against a real run

Push, then read the build back:

```bash
yontrack build search --with-promotion BRONZE --count 1 --output json
```

Done when every declared validation appears on the build and the promotion is earned.

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
`@path` file inclusion, `--var` and `--env` template variables, Sprig functions. Reach for
[doc.yontrack.com](https://doc.yontrack.com) rather than guessing at the schema.

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

Once the CLI is configured, the same job can read and annotate the build:

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
executable, and a dozen action inputs in place of a configuration file. New workflows use
`-config`; keep it that way.

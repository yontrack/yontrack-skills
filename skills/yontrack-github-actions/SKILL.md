---
name: yontrack-github-actions
description: Report a repository's CI to Yontrack from GitHub Actions — register the build once, link every job to it, declare validations and promotions in .yontrack/ci.yaml, and record validation runs with their run info. Use when wiring a workflow to Yontrack, adding a validation stamp or a promotion level, recording how long a step took, or reading Yontrack build data from CI.
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
        uses: nemerosa/ontrack-github-actions-cli-config@v2.0.0
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
    steps:
      - name: Record the job start
        id: job_start
        run: echo "started=$(date +%s)" >> "$GITHUB_OUTPUT"

      # ... checkout, toolchain setup, caches ...

      - name: Build
        id: build
        run: ./gradlew build

      - if: ${{ steps.build.outcome != '' }}
        env:
          STARTED: ${{ steps.job_start.outputs.started }}
        run: |
          run_time_flag=""
          if [ -n "${STARTED:-}" ]; then
            run_time_flag="--run-time $(( $(date +%s) - STARTED ))"
          fi
          yontrack validate \
            --validation build \
            junit --pattern "build/test-results/**/*.xml" --fail-when-no-results \
            $run_time_flag $YONTRACK_RUN_INFO
```

`steps.<id>.outcome` is empty only when the step was skipped, so this records `FAILED` when the build
broke — which is the case worth seeing in Yontrack. Prefer typed data over a bare status wherever the step
produces it (see [Validation data](#validation-data)).

The `started` output and `$YONTRACK_RUN_INFO` are what make the run measurable and traceable — do not
drop them from a validation step (see [Run info](#run-info)).

**Run `yontrack` from the root of the workspace.** The CLI reads its configuration from
`./.yontrack-config.yaml`, resolved against the current directory, and the linking step writes it at the
workspace root. In a job carrying a `defaults.run.working-directory`, a validation step must opt out or it
fails with `No current configuration`:

```yaml
    defaults:
      run:
        working-directory: client
# ...
      - if: ${{ steps.test.outcome != '' }}
        working-directory: ${{ github.workspace }}
        run: yontrack validate --validation client --status PASSED $YONTRACK_RUN_INFO
```

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

`.yontrack/ci.yaml`, processed as a template then parsed. `defaults` applies to every branch; each
`custom.configs` entry adds to it when all its `conditions` hold.

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
    configs:
      - conditions:
          - name: branch
            config: main
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

## Run info

Every validation should carry **run info**: how long the step took, and where it ran. Without it the stamp
is a light with no history — `ontrack_run_VALIDATION_RUN_time_seconds` is never emitted, the validation
history charts an empty series, the branch tooltip drops the duration, and nothing links back to the CI run.

The CLI is **purely declarative**. `GetRunInfo()` reads five flags and returns nothing when all five are
unset, so a `yontrack validate` that does not spell them out sends `runInfo: null`. Nothing is inferred
from `GITHUB_RUN_ID`, `GITHUB_EVENT_NAME` or `GITHUB_SHA`. This is silent: the validation still passes.

### Where it ran

The same four values everywhere, so set them once at workflow level:

```yaml
env:
  YONTRACK_RUN_INFO: >-
    --source-type github-workflow
    --source-uri ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
    --trigger-type ${{ github.event_name }}
    --trigger-data ${{ github.sha }}
```

`github-workflow` is the source type Yontrack's own GitHub ingestion records, so a run reported by a
workflow and one reported by a webhook read the same way in the UI. The values are a URL, an event name
and a SHA — no whitespace — so expanding `$YONTRACK_RUN_INFO` unquoted at each call site is safe.

### How long it took

GitHub exposes **no step or job duration** to expressions — there is no `steps.<id>.duration` and no
`job.duration`. You have to take the clock yourself. Record the start in a **first `run:` step of the
job** and subtract at each reporting step:

```yaml
    steps:
      - name: Record the job start
        id: job_start
        run: echo "started=$(date +%s)" >> "$GITHUB_OUTPUT"

      # ... checkout, toolchain setup, caches, the work ...

      - name: Integration tests
        id: it
        run: ./gradlew integrationTest

      - if: ${{ !cancelled() && steps.it.outcome != '' }}
        env:
          STARTED: ${{ steps.job_start.outputs.started }}
        run: |
          run_time_flag=""
          if [ -n "${STARTED:-}" ]; then
            run_time_flag="--run-time $(( $(date +%s) - STARTED ))"
          fi
          yontrack validate --validation integration --status "${{ steps.it.outcome == 'success' && 'PASSED' || 'FAILED' }}" \
            $run_time_flag $YONTRACK_RUN_INFO
```

**Measure the job, not one step of it.** Checkout, toolchain setup, cache restore, starting a database or
pulling images are all part of what a stamp costs; timing only the `./gradlew` line reports a number well
under what the pipeline actually spends, and the gap can be large. Timing one step instead is worth it
only when a single job carries several stamps for genuinely separate pieces of work.

Two spans stay outside the measurement no matter what, and are worth a comment rather than a pretence of
exactness: the runner's own *Set up job*, which precedes every step a workflow can observe, and whatever
the job does **after** the validation has been recorded.

**Guard the subtraction.** When there is no recorded start, `STARTED` arrives empty, and bash evaluates an
empty operand as `0` — `$(( $(date +%s) - STARTED ))` then quietly reports the Unix epoch, some 56 years,
into the very metric the run time exists to feed. Build the flag conditionally as above so an unknown
start omits it instead.

### Rules

- **Run-info flags go after the data subcommand.** `junit` and its siblings redeclare all five flags on
  themselves, so `--validation build junit --pattern ... --run-time 42` is the form to write. With a bare
  `--status` there is no subcommand and they follow it directly.
- **`--run-time` is in seconds**, an integer.
- **Measure the work, not the reporting.** A job that validates work done in *another* job — a matrix of
  tests collected into one stamp — has no span of its own worth reporting. Have each leg write its own job
  duration into its artefact and take the slowest in the reporting job; that is the wall clock of the
  parallel legs.
- **Omit `--run-time` rather than guess.** A fabricated duration poisons the metric the stamp exists to
  feed; the other four flags still go out without it.

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

---
name: yontrack-bitbucket-pipelines
description: Report a repository's CI to Yontrack from Bitbucket Pipelines — install the pinned CLI in each step, register the build once with ci config, hand it to later steps as an artifact, declare validations and promotions in .yontrack/ci.yaml, and record validation runs from after-script with their run info. Use when wiring a bitbucket-pipelines.yml to Yontrack, adding a validation stamp or a promotion level to a Bitbucket pipeline, recording how long a step took, or reading Yontrack build data from Bitbucket Pipelines.
---

# Reporting Bitbucket Pipelines to Yontrack

Yontrack records what a delivery pipeline did: each **build**, the **validations** that ran against it,
and the **promotions** it earned. A pipeline wired to Yontrack answers "is this version fit to deploy?"
without anyone reading a log.

One build per pipeline run. **The first step registers it; every later step picks it up from a file.**

There is no Yontrack pipe in the Bitbucket marketplace: the `yontrack` CLI is installed and driven
directly. Two traits of Pipelines shape everything below:

- **Each step runs in its own container.** Nothing a step installs, configures or exports survives into
  the next one — the CLI is installed again in every step that talks to Yontrack.
- **Only `artifacts` cross from a step to the next ones.** The registered build travels as a file.

Everything here needs **CLI 5.7.0 or later** — the release that knows Bitbucket Pipelines.

## Steps

### 1. Confirm the credentials

The pipeline needs these [repository or workspace variables](https://support.atlassian.com/bitbucket-cloud/docs/variables-and-secrets/):

| Variable | |
|---|---|
| `YONTRACK_URL` | URL of the Yontrack installation. Secured. |
| `YONTRACK_TOKEN` | A Yontrack API token. Secured. |
| `YONTRACK_CLI_VERSION` | The CLI release to install, like `5.7.0`. Optional, not a secret. |

There is no `gh variable list` for Bitbucket that the user can be assumed to have. **Ask the user** whether
they are defined. When Bitbucket API credentials are at hand, the variable names (never the secured values)
can be listed instead:

```bash
curl -fsS -u "<user>:<api-token>" \
  "https://api.bitbucket.org/2.0/repositories/<workspace>/<repo>/pipelines_config/variables/" | jq -r '.values[].key'
curl -fsS -u "<user>:<api-token>" \
  "https://api.bitbucket.org/2.0/workspaces/<workspace>/pipelines-config/variables" | jq -r '.values[].key'
```

When `YONTRACK_URL` or `YONTRACK_TOKEN` is missing, stop and ask — a token is minted in the Yontrack UI and
only the human can do it. The CLI does not read these variables itself; the scripts below hand them to
`yontrack config create`.

Independently of the pipeline, and only once, the Yontrack project is associated with its Bitbucket Cloud
repository so change logs and commit searches work. The workspace slug is required:

```bash
yontrack project set-property --project <project> bitbucket-cloud \
  --configuration <bitbucket-cloud-config> --workspace <workspace> --repository <repo>
```

The `bitbucket-cloud` configuration in Yontrack holds credentials only; the workspace belongs to the project.

### 2. Derive the version from the build number

```text
1.0.$BITBUCKET_BUILD_NUMBER
```

`BITBUCKET_BUILD_NUMBER` is the same in every step of a run, so each step derives the same version without
passing it along. The build is registered with this version as its release label (step 5), and the
artefacts the run publishes carry the same one.

Hand it to Yontrack as `--env VERSION=...`, **never as an exported shell `VERSION`**: `install.sh` reads
`VERSION` as the *CLI* release to install, so an exported `VERSION=1.0.42` makes the next install look for
CLI 1.0.42 and fail.

### 3. Declare validations and promotions

Write `.yontrack/ci.yaml` (see [Configuration file](#configuration-file)). Every validation the pipeline
records is declared here, and at least one promotion gates on them — a validation with no promotion above
it is a stamp nobody reads.

### 4. Install the CLI, pinned, in every step that uses it

```bash
export PATH="$HOME/.local/bin:$PATH"
curl -fsSL https://raw.githubusercontent.com/yontrack/yontrack-cli/main/install.sh | VERSION="${YONTRACK_CLI_VERSION:-5.7.0}" sh
yontrack config create prod "$YONTRACK_URL" --token "$YONTRACK_TOKEN"
```

- **Pin the release.** With `VERSION` empty the script installs the latest one, and a Yontrack release can
  then turn a green pipeline red in a repository nobody touched. The `:-5.7.0` fallback keeps it pinned even
  when `YONTRACK_CLI_VERSION` is not defined.
- The script needs `curl` in the image and installs into `$HOME/.local/bin` — hence the `PATH` first.
- **Keep the three lines in one `- |` block**, so the `export` and the commands that need it run in the
  same shell.
- `config create` writes `./.yontrack-config.yaml` in the current directory, **token included**. Never
  list it in `artifacts`, and run `yontrack` from the clone directory (the default working directory) —
  from anywhere else it fails with `No current configuration`.

### 5. Register the build, once, in the first step

```bash
yontrack ci config \
  --env-all BITBUCKET_ \
  --env VERSION="1.0.$BITBUCKET_BUILD_NUMBER" \
  --output env > yontrack.env
```

`ci config` creates the project, the branch and the build from the environment. `--env-all BITBUCKET_`
sends every variable with that prefix, and Yontrack recognises Bitbucket Pipelines and Bitbucket Cloud from
them on its own — `--ci bitbucket-pipelines` and `--scm bitbucket-cloud` exist only to force it. From
those variables Yontrack also:

- names the project after `BITBUCKET_REPO_SLUG` and the branch after `BITBUCKET_BRANCH`;
- names the build `<timestamp>-<BITBUCKET_BUILD_NUMBER>`;
- sets the Git commit property from `BITBUCKET_COMMIT` and links the build to the pipeline run.

`VERSION` becomes the build's **release label** — the version the build is known by, and what
auto-versioning's `versionSource: labelOnly` reads. Pass it with `--env`, not `--var`: `--var` only feeds
the template, and does not label anything.

Two pipelines it refuses or cannot name:

- **Pull-request pipelines are rejected** (`BITBUCKET_PR_ID` set). Register from `default` or `branches`,
  not from `pull-requests`.
- **Tag pipelines have no `BITBUCKET_BRANCH`.** Pass the branch explicitly with `--env BRANCH_NAME=...`.

Only the `export` lines go to standard output — `ci config` traces to standard error — so `yontrack.env`
holds the identity of the build and nothing else.

`--env-all` also echoes every selected variable to the build log. If the pipeline defines a sensitive
`BITBUCKET_*` variable of its own, list the needed ones with `--env KEY=VALUE` instead.

### 6. Hand the build to every later step

Declare the file as an artifact of the registering step:

```yaml
        artifacts:
          - yontrack.env
```

Bitbucket restores it in every later step, where sourcing it exports `YONTRACK_PROJECT_NAME`,
`YONTRACK_BRANCH_NAME`, `YONTRACK_BUILD_NAME` and `YONTRACK_BUILD_ID` — so every later `yontrack` command
needs no `--project`, `--branch` or `--build`:

```bash
source yontrack.env
```

**Prefer this route.** One call to Yontrack for the whole pipeline, and it depends on nothing the build
carries.

The alternative looks the build up again in each step, by commit:

```bash
eval "$(yontrack build search --project my-project --commit "$BITBUCKET_COMMIT" --count 1 --output env)"
```

It prints the same exports, but costs a query per step, needs the project name in every step, and finds the
build through the Git commit property `ci config` set on it. Take it only when artifacts are not an option — a
separate pipeline reporting against a build registered by another one, for instance.

### 7. Record a validation from `after-script`

`script` stops at its first failing command, so a validation placed there is skipped exactly when it matters.
`after-script` runs once `script` is over, pass or fail, and Bitbucket sets `BITBUCKET_EXIT_CODE` to the
outcome:

```yaml
      - step:
          name: Unit tests
          script:
            - date +%s > .yontrack-started
            - |
              export PATH="$HOME/.local/bin:$PATH"
              curl -fsSL https://raw.githubusercontent.com/yontrack/yontrack-cli/main/install.sh | VERSION="${YONTRACK_CLI_VERSION:-5.7.0}" sh
              yontrack config create prod "$YONTRACK_URL" --token "$YONTRACK_TOKEN"
            - ./gradlew test
          after-script:
            - |
              export PATH="$HOME/.local/bin:$PATH"
              source yontrack.env
              run_time_flag=""
              if [ -s .yontrack-started ]; then
                run_time_flag="--run-time $(( $(date +%s) - $(cat .yontrack-started) ))"
              fi
              yontrack validate --validation unit-tests \
                junit --pattern "build/test-results/**/*.xml" \
                $run_time_flag
```

`junit` derives the status from the test results, so a failing test records `FAILED` without looking at
the exit code. With a bare status, map `BITBUCKET_EXIT_CODE` instead:

```bash
if [ "$BITBUCKET_EXIT_CODE" = "0" ]; then status=PASSED; else status=FAILED; fi
yontrack validate --validation docker --status "$status" $run_time_flag
```

Prefer typed data over a bare status wherever the step produces it (see [Validation data](#validation-data)).

What to know about `after-script`:

- **It runs in a shell of its own.** Variables from `script` are gone — hence the `PATH` again, and the
  start time kept in a file. The container, the installed binary and `./.yontrack-config.yaml` are still
  there.
- **Its failure does not fail the step.** A `yontrack validate` that errors — undeclared stamp, wrong
  project — leaves the step green and the stamp missing. Check the log of the first real run.
- **Install the CLI first in `script`.** If an earlier command fails before the install, `after-script`
  finds no `yontrack`.
- `BITBUCKET_EXIT_CODE` exists only in `after-script`.

### 8. Verify

The version is already on the build — `VERSION` set its release label in step 5, so there is no
`build set-property release` to add. Read it back after a real run:

```bash
yontrack build search --project my-project --with-promotion BRONZE --count 1 --output json
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
        unit-tests:
          tests: {}      # typed stamp: accepts test counts
        docker: {}       # plain stamp: PASSED / FAILED
      promotions:
        BRONZE:
          validations:
            - unit-tests
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

Build settings sit under `defaults.build` (or a `custom` entry's `build`), beside `branch`. To name builds
otherwise than `<timestamp>-<build number>`, set `buildName` there — a template over the `env` context
described in the reference. There is no `build.name` key, and no top-level `configuration.build`.

Both the file and the CLI carry more than this: properties, notifications, workflows, auto-versioning,
`@path` file inclusion, `--var` and `--env` template variables, Sprig functions. Reach for the
[Yontrack reference documentation](https://docs.yontrack.com/yontrack/ref/index.html) rather than guessing
at the schema.

## Validation data

A typed stamp shows a trend in Yontrack where a bare status shows a light. The subcommand follows the
common flags:

```bash
yontrack validate --validation unit-tests junit --pattern "build/test-results/**/*.xml" --fail-when-no-results
yontrack validate --validation tests tests --passed 20 --skipped 2 --failed 1
yontrack validate --validation security chml --critical 0 --high 2 --medium 25 --low 1214
yontrack validate --validation coverage percentage --value 87
```

`junit` reads the XML itself and is the one to reach for with a JVM, Node or Python test run. The stamp
must be declared with a matching type in `.yontrack/ci.yaml` — `tests: {}` above.

`--fail-when-no-results` records `FAILED` when no report is found. In `after-script` that is usually
right: no report means the build broke before the tests ran.

## Run info

Every validation should carry **run info**: how long the step took, and where it ran. Without it the stamp
is a light with no history — the run-time metric is never emitted, the validation history charts an empty
series, and nothing links back to the pipeline.

### Where it ran — filled in for you

Inside Bitbucket Pipelines (detected by `BITBUCKET_BUILD_NUMBER`), CLI 5.7.0 and later default every flag
that is not given:

| Flag | Default |
|---|---|
| `--source-type` | `bitbucket-pipeline` |
| `--source-uri` | `https://bitbucket.org/$BITBUCKET_WORKSPACE/$BITBUCKET_REPO_SLUG/pipelines/results/$BITBUCKET_BUILD_NUMBER` |
| `--trigger-type` | `commit` |
| `--trigger-data` | `$BITBUCKET_COMMIT` |

So pass none of them. Explicit flags still win, but hand-assembling the URL only risks getting it wrong.
An older CLI fills in nothing and sends `runInfo: null` — silently, the validation still passes — which is
one more reason to pin 5.7.0 or later.

### How long it took — yours to measure

`--run-time` is **never** defaulted: only the caller knows. Bitbucket exposes no step duration to the
script, so take the clock yourself — **as the first command of `script`**, written to a file because
`after-script` does not share the shell (see step 7).

**Measure the step, not the command.** Pulling the image, the install, caches, starting services are all
part of what a stamp costs; timing only `./gradlew test` reports a number well under what the pipeline
spends. The image pull and clone happen before `script` and stay outside the measurement whatever you do.

**Guard the subtraction.** If the start file is missing, `$(( $(date +%s) - ))` either breaks the script or,
with an empty variable, reports the Unix epoch — some 56 years — into the very metric the run time feeds.
Build the flag conditionally, as in step 7, so an unknown start omits it.

### Rules

- **Run-info flags go after the data subcommand.** `junit` and its siblings redeclare them, so
  `--validation unit-tests junit --pattern ... --run-time 42` is the form to write. With a bare `--status`
  they follow it directly.
- **`--run-time` is in seconds**, an integer.
- **Omit `--run-time` rather than guess.** A fabricated duration poisons the metric; the defaulted flags
  still go out without it.
- **Parallel steps each report their own stamp** with their own duration. A step that validates work done
  by *other* steps has no span of its own worth reporting — have each leg write its duration into an
  artifact and take the slowest.

## A complete `bitbucket-pipelines.yml`

```yaml
image: atlassian/default-image:4

pipelines:
  default:
    - step:
        name: Yontrack build
        script:
          - |
            export PATH="$HOME/.local/bin:$PATH"
            curl -fsSL https://raw.githubusercontent.com/yontrack/yontrack-cli/main/install.sh | VERSION="${YONTRACK_CLI_VERSION:-5.7.0}" sh
            yontrack config create prod "$YONTRACK_URL" --token "$YONTRACK_TOKEN"
            yontrack ci config \
              --env-all BITBUCKET_ \
              --env VERSION="1.0.$BITBUCKET_BUILD_NUMBER" \
              --output env > yontrack.env
        artifacts:
          - yontrack.env
    - step:
        name: Unit tests
        script:
          - date +%s > .yontrack-started
          - |
            export PATH="$HOME/.local/bin:$PATH"
            curl -fsSL https://raw.githubusercontent.com/yontrack/yontrack-cli/main/install.sh | VERSION="${YONTRACK_CLI_VERSION:-5.7.0}" sh
            yontrack config create prod "$YONTRACK_URL" --token "$YONTRACK_TOKEN"
          - ./gradlew test
        after-script:
          - |
            export PATH="$HOME/.local/bin:$PATH"
            source yontrack.env
            run_time_flag=""
            if [ -s .yontrack-started ]; then
              run_time_flag="--run-time $(( $(date +%s) - $(cat .yontrack-started) ))"
            fi
            yontrack validate --validation unit-tests \
              junit --pattern "build/test-results/**/*.xml" --fail-when-no-results \
              $run_time_flag
    - step:
        name: Docker image
        services:
          - docker
        script:
          - date +%s > .yontrack-started
          - |
            export PATH="$HOME/.local/bin:$PATH"
            curl -fsSL https://raw.githubusercontent.com/yontrack/yontrack-cli/main/install.sh | VERSION="${YONTRACK_CLI_VERSION:-5.7.0}" sh
            yontrack config create prod "$YONTRACK_URL" --token "$YONTRACK_TOKEN"
          - docker build -t "my-app:1.0.$BITBUCKET_BUILD_NUMBER" .
        after-script:
          - |
            export PATH="$HOME/.local/bin:$PATH"
            source yontrack.env
            run_time_flag=""
            if [ -s .yontrack-started ]; then
              run_time_flag="--run-time $(( $(date +%s) - $(cat .yontrack-started) ))"
            fi
            if [ "$BITBUCKET_EXIT_CODE" = "0" ]; then status=PASSED; else status=FAILED; fi
            yontrack validate --validation docker --status "$status" $run_time_flag
```

The first step registers the build and publishes `yontrack.env`; each later step installs the CLI, does its
work, and records its outcome against that build from `after-script`.

## Beyond validation

Once a step has sourced `yontrack.env`, it can read and annotate the build:

```bash
yontrack build search --project my-project --with-promotion RELEASE --count 1 --output json --accept-not-found
yontrack build changelog semantic --from-promotion RELEASE --emojis --issues
```

One CLI covers validation and everything past it, so a pipeline never mixes two idioms.

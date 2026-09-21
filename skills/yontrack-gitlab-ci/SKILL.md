---
name: yontrack-gitlab-ci
description: Report a repository's CI to Yontrack from GitLab CI/CD — install the pinned CLI in each job, register the build once with ci config, hand it to later jobs as a dotenv artifact, declare validations and promotions in .yontrack/ci.yaml, and record validation runs from after_script with their run info. Use when wiring a .gitlab-ci.yml to Yontrack, adding a validation stamp or a promotion level to a GitLab pipeline, recording how long a job took, or reading Yontrack build data from GitLab CI.
---

# Reporting GitLab CI to Yontrack

Yontrack records what a delivery pipeline did: each **build**, the **validations** that ran against it,
and the **promotions** it earned. A pipeline wired to Yontrack answers "is this version fit to deploy?"
without anyone reading a log.

One build per pipeline. **The first job registers it; every later job receives it as variables.**

There is no Yontrack component in the GitLab CI/CD catalog: the `yontrack` CLI is installed and driven
directly. Two traits of GitLab CI shape everything below:

- **Jobs do not share a shell.** Each job runs in a container of its own, often on another runner. Nothing a
  job installs, configures or exports survives into the next one — the CLI is installed again in every job
  that talks to Yontrack.
- **Only `artifacts` cross from a job to the later ones** — files, or variables through a `dotenv` report.
  The registered build travels that way.

Everything here needs **Yontrack 5.5 or later**, whose `gitlab-ci` CI engine and `gitlab` SCM engine read
the pipeline, and **CLI 5.7.0 or later**, which fills in the run info inside GitLab CI.

## Steps

### 1. Confirm the credentials

The pipeline needs these [CI/CD variables](https://docs.gitlab.com/ci/variables/), on the project or on a
parent group when several projects feed the same Yontrack:

| Variable | |
|---|---|
| `YONTRACK_URL` | URL of the Yontrack installation. Protected. |
| `YONTRACK_TOKEN` | A Yontrack API token. **Masked** and protected. |
| `YONTRACK_CLI_VERSION` | The CLI release to install, like `5.7.0`. Optional, not a secret. |

List the names the project and its group already carry:

```bash
glab variable list
glab variable list --group <group>
```

When `YONTRACK_URL` or `YONTRACK_TOKEN` is missing, stop and ask — a token is minted in the Yontrack UI and
only the human can do it. The CLI does not read these variables itself; the scripts below hand them to
`yontrack config create`.

**Masked** keeps the token out of the job logs. **Protected** means the variable reaches only pipelines on
protected branches and tags: every other pipeline sees an empty `YONTRACK_TOKEN` and fails to register its
build. That is the right trade when only protected branches feed Yontrack. If feature branches must report
too, say so to the user and leave the token unprotected — do not make that call silently.

Yontrack itself needs a **GitLab configuration** whose URL is the GitLab instance — gitlab.com or a
self-managed host. `ci config` matches the pipeline's project URL against the configured instances and sets
the project's GitLab property on its own, the full project path included, subgroups and all. When several
configurations name the same instance, it fails and asks for `scmConfig` (see
[Configuration file](#configuration-file)).

### 2. Compute the version once, from `CI_PIPELINE_IID`

```yaml
variables:
  VERSION: "1.0.$CI_PIPELINE_IID"
```

At the top level of `.gitlab-ci.yml`, every job of the pipeline sees the same value. `CI_PIPELINE_IID` is
the project's own pipeline counter, 1, 2, 3...; its sibling `CI_PIPELINE_ID` is unique across the whole
instance and jumps by arbitrary amounts — a poor version number.

The build is registered with this version as its release label (step 5), and the artefacts the pipeline
publishes carry the same one.

`VERSION` is also what `install.sh` reads as the *CLI* release to install. The install line in step 4 sets
it inline, which overrides the pipeline's `VERSION` for that one command. **Never shorten it to a bare
`| sh`**: the script would then look for CLI `1.0.42` and fail.

### 3. Declare validations and promotions

Write `.yontrack/ci.yaml` (see [Configuration file](#configuration-file)). Every validation the pipeline
records is declared here, and at least one promotion gates on them — a validation with no promotion above
it is a stamp nobody reads.

### 4. Install the CLI, pinned, in every job that uses it

Put it in the pipeline's `default:before_script`, so every job gets it:

```yaml
default:
  before_script:
    - date +%s > .yontrack-started
    - apt-get update -qq && apt-get install -y -qq --no-install-recommends curl ca-certificates
    - export PATH="$HOME/.local/bin:$PATH"
    - curl -fsSL https://raw.githubusercontent.com/yontrack/yontrack-cli/main/install.sh | VERSION="${YONTRACK_CLI_VERSION:-5.7.0}" sh
    - yontrack config create prod "$YONTRACK_URL" --token "$YONTRACK_TOKEN"
```

- **Pin the release.** With `VERSION` empty the script installs the latest one, and a Yontrack release can
  then turn a green pipeline red in a repository nobody touched. The `:-5.7.0` fallback keeps it pinned even
  when `YONTRACK_CLI_VERSION` is not defined.
- The script needs `curl`. The `apt-get` line is for Debian and Ubuntu images; an Alpine image uses
  `apk add --no-cache curl`, and an image that already has `curl` drops the line.
- The script installs into `$HOME/.local/bin` — hence the `PATH`. `before_script` and `script` run in one
  shell, so the `export` reaches the whole job; `after_script` does not (step 7).
- **A job's own `before_script` replaces the default one entirely**, it does not add to it. A job that needs
  one repeats the install lines, or pulls them in with `!reference [default, before_script]`.
- `config create` writes `./.yontrack-config.yaml` in the clone directory, **token included**. Keep it out
  of every `artifacts` and `cache` path, and run `yontrack` from `$CI_PROJECT_DIR` — from anywhere else it
  fails with `No current configuration`.
- The first line records when the job started, for the run time (step 7).

### 5. Register the build, once, in the first job

```bash
yontrack ci config \
  --env-all CI_ \
  --env-all GITLAB_CI \
  --env VERSION="$VERSION" \
  --output env | sed 's/^export //' > yontrack.env
```

`ci config` creates the project, the branch and the build from the environment. `--env-all` sends every
variable with the given prefix. GitLab's predefined variables are the `CI_` ones, but the variable the
`gitlab-ci` engine is detected by is `GITLAB_CI`, which has no such prefix — hence the second `--env-all`,
which matches that one variable. Drop it and Yontrack does not recognise the pipeline. `--ci gitlab-ci` and
`--scm gitlab` exist only to force the detection.

From those variables Yontrack:

- names the project after `CI_PROJECT_NAME` and the branch after `CI_COMMIT_REF_NAME`;
- names the build `<timestamp>-<CI_PIPELINE_IID>`;
- takes the repository from `CI_PROJECT_URL` — **never** from `CI_REPOSITORY_URL`, which embeds the job
  token;
- sets the Git commit property from `CI_COMMIT_SHA` and links the build to the pipeline (`CI_PIPELINE_URL`).

`VERSION` becomes the build's **release label** — the version the build is known by, and what
auto-versioning's `versionSource: labelOnly` reads. Pass it with `--env`, not `--var`: `--var` only feeds
the template, and does not label anything. It has no `CI_` prefix either, so `--env-all CI_` does not carry
it.

`--env-all CI_` also sends — and echoes to the job log — `CI_JOB_TOKEN`, `CI_REPOSITORY_URL` and, where the
container registry is on, `CI_REGISTRY_PASSWORD`. GitLab masks them in the log and Yontrack reads none of
them, but they do reach it. Where that matters, list what the engine reads instead, each with its value:

```bash
yontrack ci config \
  --env GITLAB_CI="$GITLAB_CI" \
  --env CI_PROJECT_URL="$CI_PROJECT_URL" \
  --env CI_PROJECT_PATH="$CI_PROJECT_PATH" \
  --env CI_PROJECT_NAME="$CI_PROJECT_NAME" \
  --env CI_COMMIT_SHA="$CI_COMMIT_SHA" \
  --env CI_COMMIT_REF_NAME="$CI_COMMIT_REF_NAME" \
  --env CI_MERGE_REQUEST_IID="${CI_MERGE_REQUEST_IID:-}" \
  --env CI_PIPELINE_ID="$CI_PIPELINE_ID" \
  --env CI_PIPELINE_IID="$CI_PIPELINE_IID" \
  --env CI_PIPELINE_URL="$CI_PIPELINE_URL" \
  --env VERSION="$VERSION" \
  --output env | sed 's/^export //' > yontrack.env
```

`--env` takes `KEY=VALUE`; a bare `--env GITLAB_CI` is rejected.

Pipelines it refuses or cannot name:

- **Merge request pipelines are rejected**: with `CI_MERGE_REQUEST_IID` set the branch is `PR-<iid>`, which
  Yontrack refuses. Register from branch pipelines only — if the project runs merge request pipelines, keep
  every Yontrack job out of them with
  `rules: [{ if: '$CI_PIPELINE_SOURCE == "merge_request_event"', when: never }, { when: on_success }]`.
- **Tag pipelines** have the tag in `CI_COMMIT_REF_NAME`. Pass the branch explicitly with
  `--env BRANCH_NAME=...`.

Only the `export` lines go to standard output — `ci config` traces to standard error — so `yontrack.env`
holds the identity of the build and nothing else. The `sed` strips their `export ` prefix, which a `dotenv`
report does not accept.

### 6. Hand the build to every later job

Declare the file as a `dotenv` report of the registering job:

```yaml
  artifacts:
    reports:
      dotenv: yontrack.env
```

GitLab turns it into variables of every later job — `YONTRACK_PROJECT_NAME`, `YONTRACK_BRANCH_NAME`,
`YONTRACK_BUILD_NAME`, `YONTRACK_BUILD_ID` and their ids — in `script` and in `after_script` alike, so
every later `yontrack` command needs no `--project`, `--branch` or `--build`, and nothing has to be
sourced.

A job receives the artifacts of every job in earlier stages by default, or only those of the jobs it lists
in `needs`. A job with `needs: []` or `dependencies: []` receives no build: list the registering job in its
`needs`.

**Prefer this route.** One call to Yontrack for the whole pipeline, and it depends on nothing the build
carries.

The alternative looks the build up again in each job, by commit:

```bash
eval "$(yontrack build search --project my-project --commit "$CI_COMMIT_SHA" --count 1 --output env)"
```

It prints the same exports, but costs a query per job, needs the project name in every job, and finds the
build only through the Git commit property `ci config` set on it. In a merged results pipeline it finds
nothing: `CI_COMMIT_SHA` is then the temporary merge commit, not the commit the build was registered with.
Take it only when artifacts are not an option — a separate pipeline reporting against a build registered by
another one, for instance.

### 7. Record a validation from `after_script`

`script` stops at its first failing command, so a validation placed there is skipped exactly when it matters.
`after_script` runs once `script` is over, pass or fail, with `CI_JOB_STATUS` set to `success`, `failed` or
`canceled`:

```yaml
unit-tests:
  stage: test
  script:
    - ./gradlew test
  after_script:
    - |
      export PATH="$HOME/.local/bin:$PATH"
      run_time_flag=""
      if [ -s .yontrack-started ]; then
        run_time_flag="--run-time $(( $(date +%s) - $(cat .yontrack-started) ))"
      fi
      yontrack validate --validation unit-tests \
        junit --pattern "build/test-results/**/*.xml" --fail-when-no-results \
        $run_time_flag
```

`junit` derives the status from the test results, so a failing test records `FAILED` without looking at
the job status. With a bare status, map `CI_JOB_STATUS` instead:

```bash
if [ "$CI_JOB_STATUS" = "success" ]; then status=PASSED; else status=FAILED; fi
yontrack validate --validation docker --status "$status" $run_time_flag
```

A cancelled job lands in `FAILED` here. Test for `failed` alone and skip the rest if a cancellation should
leave no run at all.

Prefer typed data over a bare status wherever the job produces it (see [Validation data](#validation-data)).

What to know about `after_script`:

- **It runs in a shell of its own.** What `before_script` and `script` exported is gone — hence the `PATH`
  again, and the start time kept in a file. The container, the installed binary, `./.yontrack-config.yaml`,
  the pipeline's `variables` and the dotenv variables are all still there.
- **Its failure does not fail the job.** A `yontrack validate` that errors — undeclared stamp, wrong
  project — leaves the job green and the stamp missing. Check the log of the first real run.
- **The install comes first, in `before_script`.** If `before_script` fails before it, `after_script`
  finds no `yontrack`.
- `CI_JOB_STATUS` is only meaningful in `after_script`; in `script` it is always `running`.

### 8. Verify

The version is already on the build — `VERSION` set its release label in step 5, so there is no
`build set-property release` to add. Read it back after a real run:

```bash
yontrack build search --project my-project --with-promotion BRONZE --count 1 --output json
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
        unit-tests:
          tests: {}      # typed stamp: accepts test counts
        docker: {}       # plain stamp: PASSED / FAILED
      promotions:
        BRONZE:
          validations:
            - unit-tests
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

Project settings sit under `defaults.project`, beside `branch`. `scmConfig` names the GitLab configuration
to use when more than one is registered for the instance; `scmIndexationInterval` sets the indexation
interval of the project's GitLab property.

Build settings sit under `defaults.build` (or a `custom` entry's `build`). To name builds otherwise than
`<timestamp>-<pipeline IID>`, set `buildName` there — a template over the `env` context described in the
reference. There is no `build.name` key, and no top-level `configuration.build`.

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

`--fail-when-no-results` records `FAILED` when no report is found. In `after_script` that is usually
right: no report means the job broke before the tests ran.

## Run info

Every validation should carry **run info**: how long the job took, and where it ran. Without it the stamp
is a light with no history — the run-time metric is never emitted, the validation history charts an empty
series, and nothing links back to the pipeline.

### Where it ran — filled in for you

Inside GitLab CI (detected by `GITLAB_CI=true`), CLI 5.7.0 and later default every flag that is not given:

| Flag | Default |
|---|---|
| `--source-type` | `gitlab-pipeline` |
| `--source-uri` | `$CI_PIPELINE_URL` |
| `--trigger-type` | `commit` |
| `--trigger-data` | `$CI_COMMIT_SHA` |

So pass none of them. Explicit flags still win, but there is nothing to gain by spelling them out. An
older CLI fills in nothing and sends `runInfo: null` — silently, the validation still passes — which is one
more reason to pin 5.7.0 or later.

### How long it took — yours to measure

`--run-time` is **never** defaulted: only the caller knows. Take the clock yourself — **as the first line of
`before_script`**, written to a file because `after_script` does not share the shell (see step 7).

**Measure the job, not the command.** Installing `curl` and the CLI, restoring caches, starting services
are all part of what a stamp costs; timing only `./gradlew test` reports a number well under what the
pipeline spends. The image pull and the clone happen before `before_script` and stay outside the
measurement whatever you do.

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
- **Parallel jobs each report their own stamp** with their own duration. A job that validates work done by
  *other* jobs has no span of its own worth reporting — have each leg write its duration into an artifact
  and take the slowest.

## A complete `.gitlab-ci.yml`

```yaml
stages:
  - yontrack
  - test

variables:
  VERSION: "1.0.$CI_PIPELINE_IID"

default:
  image: ubuntu:24.04
  before_script:
    - date +%s > .yontrack-started
    - apt-get update -qq && apt-get install -y -qq --no-install-recommends curl ca-certificates
    - export PATH="$HOME/.local/bin:$PATH"
    - curl -fsSL https://raw.githubusercontent.com/yontrack/yontrack-cli/main/install.sh | VERSION="${YONTRACK_CLI_VERSION:-5.7.0}" sh
    - yontrack config create prod "$YONTRACK_URL" --token "$YONTRACK_TOKEN"

yontrack-build:
  stage: yontrack
  script:
    - |
      yontrack ci config \
        --env-all CI_ \
        --env-all GITLAB_CI \
        --env VERSION="$VERSION" \
        --output env | sed 's/^export //' > yontrack.env
  artifacts:
    reports:
      dotenv: yontrack.env

unit-tests:
  stage: test
  image: eclipse-temurin:21-jdk
  script:
    - ./gradlew test
  after_script:
    - |
      export PATH="$HOME/.local/bin:$PATH"
      run_time_flag=""
      if [ -s .yontrack-started ]; then
        run_time_flag="--run-time $(( $(date +%s) - $(cat .yontrack-started) ))"
      fi
      yontrack validate --validation unit-tests \
        junit --pattern "build/test-results/**/*.xml" --fail-when-no-results \
        $run_time_flag

docker:
  stage: test
  image: docker:27
  services:
    - docker:27-dind
  before_script:
    - date +%s > .yontrack-started
    - apk add --no-cache curl
    - export PATH="$HOME/.local/bin:$PATH"
    - curl -fsSL https://raw.githubusercontent.com/yontrack/yontrack-cli/main/install.sh | VERSION="${YONTRACK_CLI_VERSION:-5.7.0}" sh
    - yontrack config create prod "$YONTRACK_URL" --token "$YONTRACK_TOKEN"
  script:
    - docker build -t "my-app:$VERSION" .
  after_script:
    - |
      export PATH="$HOME/.local/bin:$PATH"
      run_time_flag=""
      if [ -s .yontrack-started ]; then
        run_time_flag="--run-time $(( $(date +%s) - $(cat .yontrack-started) ))"
      fi
      if [ "$CI_JOB_STATUS" = "success" ]; then status=PASSED; else status=FAILED; fi
      yontrack validate --validation docker --status "$status" $run_time_flag
```

The first job registers the build and publishes it as a `dotenv` report; each later job installs the CLI,
does its work, and records its outcome against that build from `after_script`. The `docker` job runs on an
Alpine image, so it overrides `before_script` with the `apk` form of the install.

## Beyond validation

Any job after the registering one can read and annotate the build:

```bash
yontrack build search --project my-project --with-promotion RELEASE --count 1 --output json --accept-not-found
yontrack build changelog semantic --from-promotion RELEASE --emojis --issues
```

One CLI covers validation and everything past it, so a pipeline never mixes two idioms.

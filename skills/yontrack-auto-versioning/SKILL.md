---
name: yontrack-auto-versioning
description: Propagate a version between repositories on a Yontrack promotion — declare auto-versioning on the target branch so a promoted build updates a version in another repository's file, and Yontrack pushes or opens a PR for it. Use when wiring up auto-versioning, making a GitOps repository follow a promotion, or debugging a bump that did not happen.
---

# Auto-versioning on promotion

A promotion in one project updates a version in another repository. Yontrack watches the source project;
when a build is promoted it creates an upgrade branch in the target repository, rewrites the version, and
either merges it or opens a pull request.

This is the target-side counterpart to reporting builds. It suits GitOps repositories with pinned versions
and modular monoliths — not ordinary library dependencies, which lockfiles handle better.

**The configuration lives on the target branch, not the source.** The repository being *updated* declares
what it consumes. A repository can be an auto-versioning target while reporting no builds of its own; it
only has to exist in Yontrack.

## Steps

### 1. Find the version and how it is expressed

Locate the exact file and property the target repository pins, and pick the matching type (see
[Target file types](#target-file-types)). Confirm the current value by reading the file — the property path
is the thing most often wrong, and a wrong one fails at promotion time, not now.

### 2. Decide where the version comes from

`versionSource` decides what gets written, and its default is a trap:

| Value | Behaviour |
|---|---|
| `default` | The build's **label** if the source project is configured for label display, otherwise the build **name**. Rejected if configured for labels and the label is missing. |
| `name` | Always the build name. |
| `labelOnly` | Always the label; the request is **rejected** when there is no label. |
| `metaInfo/<category>/<name>` | A meta-information item on the build. |

When the source project's real version lives on a label — a `release` property set by its pipeline, with
Yontrack naming builds itself — set `labelOnly`. Leaving it `default` writes the build *name*, so a
timestamp lands in the target file and the deployment references an artefact that does not exist. `labelOnly`
turns that into a rejection, which leaves the previous version in place.

### 3. Declare the configuration

Append to `autoVersioning.configurations` on the target branch, in the target repository's
`.yontrack/ci.yaml`:

```yaml
version: v1
configuration:
  defaults:
    branch:
      autoVersioning:
        configurations:
          - sourceProject: rpg-notes
            sourceBranch: main
            sourcePromotion: BRONZE
            targetPath: apps-helmfile.d/helmfile/defaults.yaml
            targetPropertyType: yaml-path
            targetProperty: helmfile.rpgNotes.version
            versionSource: labelOnly
            validationStamp: rpg-notes-av
    build:
      autoVersioningCheck: true
```

`sourceProject`, `sourceBranch` and `sourcePromotion` are the only required fields beyond `targetPath`.
`sourceBranch` is a regular expression.

`validationStamp` creates a stamp on the *target's* builds recording whether it is up to date — a name, or
`auto` for `auto-versioning-<project>`. `autoVersioningCheck: true` on the build defaults is what makes that
check run.

### 4. Choose how the change lands

`pushMode` is `PR` by default: an upgrade branch, then a pull request, merged automatically unless
`autoApproval: false`. `PUSH` merges the upgrade branch straight into the target branch and deletes it.

Take `PUSH` only when nothing validates a pull request in that repository — for example when its CI
triggers on `push` alone, so a PR would collect no checks and exist only as a record. Be explicit about the
cost: with no pull request, **a rejected request is silent**, and the target quietly keeps its old version.
Under `PUSH`, `autoApproval`, `reviewers` and the PR templates have no effect.

The target repository can be on GitHub, Bitbucket Server or **Bitbucket Cloud**. On Bitbucket Cloud, two
differences for pull requests:

- **Auto-approval needs the auto-merge identity.** The Bitbucket Cloud configuration used by the target
  project must carry `autoMergeEmail` and `autoMergeToken` — an Atlassian account *other than* the
  configuration's own, with write access to the target repository, since nobody can approve their own pull
  request. Without it, an order with `autoApproval` fails before any pull request is created. Yontrack then
  approves, waits for the pull request's builds, and merges it itself.
- **Only `autoApprovalMode: CLIENT`** (the default). Bitbucket Cloud cannot schedule a merge for when checks
  pass, so `SCM` mode is rejected.

### 5. Report back to the source, if useful

`backValidation: <stamp>` puts a validation on the *source* build once its version has reached the target.
It is the cheapest way for the source project to record that a version was actually deployed.

Keep that stamp out of the promotion that triggers the auto-versioning. Gating the triggering promotion on
its own result is circular, and nothing would ever promote.

### 6. Verify against a real promotion

Auto-versioning fires on *new* promotions only, so configuring it changes nothing until the next one. When
the target is behind, bump it by hand in the same change — that separates a broken deployment from a broken
automation.

Done when a promotion in the source project produces a commit in the target repository carrying the
promoted version.

## Target file types

`targetPropertyType` with `targetProperty`, or `targetRegex` for anything not covered:

| Type | `targetProperty` is |
|---|---|
| *(unset)* | A key in a Java properties file |
| `yaml-path` | A JSON Path into a YAML file — the straightforward choice for Helm values |
| `yaml` | A Spring Expression Language path into a YAML file, for selecting list elements by predicate |
| `maven` | A `<properties>` element of a `pom.xml` |
| `toml` | A key path in a TOML file |

`targetRegex` replaces `targetProperty` entirely: the first matching group is the version. Reach for it when
the version sits in a line no property path addresses, such as `^appVersion: "(.*)"$` in a `Chart.yaml`.

`additionalPaths` updates further files in the same change, each with its own `property`, `propertyType` and
`versionSource`.

## Diagnosing a bump that did not happen

- **The target's `*-av` validation is `FAILED`** — the pin is stale; the configuration is being evaluated
  and the request is being rejected or has not run.
- **A timestamp or a build name landed instead of a version** — `versionSource` is `default` where it
  should be `labelOnly`.
- **Nothing at all happened** — check that the promotion actually occurred on a branch matching
  `sourceBranch`, and that the target repository exists as a project in Yontrack.
- **Requests disappear under load** — expected. Queued requests for the same source and target are
  cancelled except the latest; `cronSchedule` delays processing deliberately.

Yontrack emits `auto-versioning-error`, `auto-versioning-post-processing-error`,
`auto-versioning-pr-merge-timeout-error` and `auto-versioning-success`. Subscribe with `notifications` on
the configuration when silent failure is not acceptable — the alternative to noticing late.

Post-processing, branch expressions (`&regex`, `&same`, `&most-recent`, `&same-release`), approval modes and
audit cleanup are all real and all out of scope here. On Bitbucket Cloud, `postProcessing: bitbucket-cloud`
triggers a `custom:` pipeline that clones the upgrade branch, runs the command and pushes back to it. See the
[auto-versioning reference](https://docs.yontrack.com/yontrack/ref/latest/content/integrations/auto-versioning/auto-versioning.html).

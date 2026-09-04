---
name: yontrack-changelogs
description: Render a change log inside a Yontrack template — in a promotion notification, a workflow node, or an auto-versioning pull request body. Covers the plain and the semantic (conventional-commit) forms, which interval each one measures, and every way one renders empty. Use when adding a change log to a notification or PR body, when choosing between `changelog` and `semanticChangelog`, or when a rendered change log came out empty or lost commits.
---

# Change logs in templates

Yontrack renders a change log wherever it renders a template: a notification's `contentTemplate`, a
workflow node's `template`, an auto-versioning `prBodyTemplate`. The expression looks like a variable
with query parameters:

```
${promotionRun.semanticChangelog?issues=true&emojis=true}
```

Two things decide what you get. **Which renderable** you name fixes the interval — the two builds the
change log spans — and you never pass those builds yourself. **Plain or semantic** fixes the shape:
a flat list, or sections grouped by conventional-commit type.

The semantic form has a prerequisite that is easy to miss and fails silently. Read step 3 before
committing to it.

## Steps

### 1. Pick the renderable — it *is* the interval

| Expression | From | To |
|---|---|---|
| `${promotionRun.changelog}` | last build previously promoted to **the same level** | the build being promoted |
| `${promotionRun.semanticChangelog}` | same | same |
| `${build.changelog?from=<id>}` | the build whose **numeric internal ID** is `from` | this build |
| `${av.changelog}` | the build matching the version **currently in the target file** | the build being upgraded to |
| `${av.semanticChangelog}` | same | same |

`promotionRun` is the one you want on a promotion notification, and its interval is the useful part:
"since the last build that reached this level". On a `SILVER` promotion run that is SILVER-to-SILVER;
on `RELEASE` it is release-to-release. There is no parameter to say so and none is needed.

`build.changelog` takes `from` as a **required** parameter and it is Yontrack's internal build ID —
an integer, not a build name and not a version. There is rarely a template context that knows one, so
this renderable is far less useful than it looks.

`av.*` is only available in auto-versioning templates (`prTitleTemplate`, `prBodyTemplate`), where
`promotionRun` and `build` are not in scope.

### 2. Choose plain or semantic

Plain (`changelog`) renders issues and, depending on `commitsOption`, commits. Semantic
(`semanticChangelog`) renders an optional issues section followed by one section per
conventional-commit type, each commit shown as its subject with the scope in bold.

They do **not** take the same parameters, and the gaps matter:

| Parameter | `changelog` | `semanticChangelog` | Notes |
|---|---|---|---|
| `empty` | yes | **`promotionRun`: no**; `av`: yes | Text to render for an empty change log. See step 5. |
| `title` | yes | **no** | Semantic sections carry their own headers. |
| `commitsOption` | yes | **no** | `NONE` (default), `OPTIONAL` (only when there are no issues), `ALWAYS`. |
| `issues` | no | yes | Adds a `📋 Issues` section above the type sections. |
| `sections` | no | yes | `type=Title` mapping. See the trap in step 6. |
| `exclude` | no | yes | Types to drop entirely. |
| `emojis` | no | yes | Emoji in the section titles. |
| `commitsMaxLength` | yes | yes | Default `100`. |
| `dependencies` | yes | yes | Comma-separated project links to follow for a deep change log. |
| `acrossBranches` | `promotionRun` only | `promotionRun` only | Default `true`. |

Semantic is the better read *when the commits are typed*. When they are not, it is strictly worse
than plain — see the next step.

### 3. Confirm the commits are typed, or the semantic form will lie

**`SemanticChangelogRenderingServiceImpl` drops every commit that carries no conventional-commit type,
silently.** No "other" section, no count, no marker. A repository writing `Fix the retry loop` and
`#1683 Document the pipeline view` produces a semantic change log that is *empty*, and it looks like
there were simply no changes.

Check before you commit to it:

```bash
# How many of the last 60 subjects carry a type?
git log --oneline -60 --format='%s' | grep -cE '^[a-z]+(\([^)]+\))?!?: '
```

If that number is low, you have three honest options:

1. **`?issues=true`** and let the issues section carry the content. This works well when the
   repository references an issue on nearly every commit — the issues come from the issue service,
   not from commit types, so they survive. This is what `yontrack/yontrack` itself does.
2. **Use `changelog?commitsOption=ALWAYS` instead.** Nothing is grouped, but nothing is lost.
3. **Fix the commits first.** Adopt typed subjects, then switch. This is the only option that makes
   the semantic form genuinely better than the plain one.

The parser is `^(\w+)(?:\(([^)]+)\))?!?: (.+)$` against the **first line only**. Note what that
accepts: any word followed by `: ` is a type. `Demo: workflow-backed promotion` parses as type
`Demo`, and gets its own section titled `Demo`.

### 4. Write the expression

Parameters are query-style, repeated for list values:

```yaml
contentTemplate: |
  ${build} reached ${promotionLevel}.

  ${promotionRun.semanticChangelog?issues=true&emojis=true}
```

```yaml
prBodyTemplate: |
  Deployment of ${sourceProject} version ${VERSION}.

  ${av.semanticChangelog?issues=true&emojis=true}
```

### 5. Decide what an empty change log looks like

Every one of these renders empty, and most are normal operation rather than errors:

- **`promotionRun.*`** — no previous build at that level. The first promotion ever, and the first
  after any gap.
- **`av.*`** — the order carries no source build; or the target file's current version does not
  resolve to a build in the source project (a hand-edited version, or one predating Yontrack).
- **`semanticChangelog`** — every commit in the interval is untyped, per step 3.

`changelog` and `av.semanticChangelog` accept `?empty=No changes` to say something instead.
**`promotionRun.semanticChangelog` does not** — it has no `empty` parameter and returns an empty
string. So on a promotion notification, **put the semantic change log last**, after your call to
action. Anywhere higher and an empty render leaves a hole in the middle of the message.

### 6. Only then, tune the sections

Do this last, and only if the output is genuinely noisy.

`?exclude=chore&exclude=ci` drops types outright. `?sections=chore=Misc` retitles one.

**The `sections` mapping can silently drop commits.** `getTypeTitle` gives a mapped title an emoji
only if the *original type* is a known one, and the sort that follows is
`.toSortedMap { a, b -> a.title.compareTo(b.title) }` — comparing titles while **ignoring emojis**.
Map an unknown type onto a known type's title and you get two keys that compare equal, so one
overwrites the other in the `TreeMap` and its commits vanish. Concretely, with `emojis=true`:

```
?sections=doc=Documentation      # `doc`  -> "Documentation"    (no emoji)
                                 # `docs` -> "📝 Documentation" (emoji)
                                 # titles compare equal -> one group is dropped
```

There is no template-side fix. Fix the commit subjects instead.

## Reference

### Known types

Anything outside this table is accepted and rendered as a raw section title — a typo makes its own
section rather than an error.

| Type | Title | Emoji |
|---|---|---|
| `build` | Build | 🏗️ |
| `chore` | Misc. | 🧹 |
| `ci` | CI | 👷 |
| `docs` | Documentation | 📝 |
| `feat` | Features | ✨ |
| `fix` | Fixes | 🐛 |
| `style` | Style | 🎨 |
| `refactor` | Refactoring | ♻️ |
| `perf` | Performance | ⚡ |
| `test` | Tests | ✅ |

The issues section uses 📋.

### Traps

- **`sections` splits on `=`, never on `:`.** `?sections=fix:Bugs` does not create a "Bugs" section —
  with no `=` present, the whole string becomes both type and title, so you get a section whose type
  is the literal `fix:Bugs`, matching no commit. Published examples using `:` are wrong. Always
  `type=Title`.
- **`av.*` only reads the first target path.** A configuration with several `targetPath` entries
  computes its change log from `paths.first()` alone. If that file's version differs from the others,
  the interval is wrong — with no warning.
- **`commitsOption` is `NONE` by default.** A plain `${promotionRun.changelog}` with no parameters
  and no issues in the interval renders nothing at all. Use `OPTIONAL` to fall back to commits only
  when there are no issues, or `ALWAYS`.
- **`acrossBranches` defaults to `true`.** With no previous promotion on the current branch, Yontrack
  looks across every branch of the project. Set it `false` when a per-branch interval is what you
  meant.
- **Commit bodies are never rendered.** Only the first line of a message is used. Detail in a commit
  body will not appear in any change log.

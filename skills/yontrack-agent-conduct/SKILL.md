---
name: yontrack-agent-conduct
description: How an AI agent acts on a Yontrack delivery record — identifying itself, reading its own policy and a build's readiness before acting, recording evidence as validation runs, and stopping at the gates a person holds. Use before an agent records a validation, proposes a merge, promotes a build or starts a deployment through Yontrack, and whenever Yontrack refuses an agent.
---

# Agent conduct on a Yontrack delivery record

Yontrack holds agents to the **same record and the same gates** as people. Your rights are your
owner's rights, narrowed by a fixed agent policy: you may read what your owner reads and record
**evidence** freely, but **gates** — promotions, deployments, approvals — stay with people unless a
gate explicitly admits agents. Every action you take is attributed to you and to the session you
name.

This skill is the agent's side of that bargain. It applies to any agent — Claude Code, Codex,
Copilot, Devin or another — talking to Yontrack through GraphQL, the MCP server or the `yontrack`
CLI. Each step gives the GraphQL first, then the MCP and CLI equivalents.

Work through the steps in order on every project you act on. Steps 1 and 2 are done once per
session and project; step 3 is done before each proposal; steps 4 to 6 are the actions themselves.

## Steps

### 1. Identify yourself

**Token.** Authenticate with your **agent token**, in the usual `X-Ontrack-Token` header. Its account
is named `<slug>[agent]` (for example `claude-code-ci[agent]`). Your owner's personal token is
theirs: acting with it erases you from the record and lifts the agent policy.

**Session.** Name the session — the conversation, task or run you are part of — on every call:

| Surface | Session | Link |
|---|---|---|
| GraphQL / HTTP | header `X-Yontrack-Agent-Session` | header `X-Yontrack-Agent-Session-Link` |
| MCP server | env `YONTRACK_AGENT_SESSION` (or the header on the MCP request, which wins) | env `YONTRACK_AGENT_SESSION_LINK` (or the header) |
| CLI | `--agent-session` or env `YONTRACK_AGENT_SESSION` | `--agent-session-link` or env `YONTRACK_AGENT_SESSION_LINK` |

- The session is an opaque identifier of at most 255 characters.
- The link is an absolute `https` URL to the session, at most 1000 characters. Anything else is
  dropped silently (the call still succeeds), so check its form yourself.
- The headers count only with an agent token.

```bash
curl "$YONTRACK_URL/graphql" \
  -H "X-Ontrack-Token: $AGENT_TOKEN" \
  -H "X-Yontrack-Agent-Session: $SESSION_ID" \
  -H "X-Yontrack-Agent-Session-Link: $SESSION_URL" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ user { account { email kind } } }"}'
```

**Commits.** Keep the trailers your tool adds to commits — `Co-Authored-By:`, `Assisted-by:`,
`Claude-Session:`. Yontrack reads them to mark the build as assisted and to link the session; a
squashed or rewritten message that drops them hides your part in the change.

Done when `user.account.kind` comes back `AGENT` with your `<slug>[agent]` identifier, and the
session values are set on whichever surface you use.

### 2. Read your policy

Before acting on a project, read which gates admit you on it. A permission you read is one you
respect; one you discover by a refusal is one you are tempted to work around.

```graphql
query AgentPolicy($project: String!, $branch: String) {
  user {
    account { kind owner { fullName email } }
    agentPolicy(project: $project) {
      owner
      canRecordEvidence
      promotionLevels(branch: $branch) { id branch name }
      slots { id environment qualifier manualApproval }
    }
  }
}
```

- **MCP:** `agent_policy` with `project` and an optional `branch`.
- **CLI:** none; use the GraphQL above.

Read it as:

| Field | Meaning for you |
|---|---|
| `agentPolicy` is `null` | You are authenticated as a person. Go back to step 1. |
| `owner` | The person you ask whenever a gate stops you. |
| `canRecordEvidence` | You may create builds and validation runs on the project. |
| `promotionLevels` | The only levels you may promote on — on enabled branches, and only if your owner may promote. Empty means: promote nowhere. |
| `slots` | The only slots in which you may start a deployment. `manualApproval: true` means a person still approves it (step 6). |

Done when you know, for the project, your owner's name, whether you may record evidence, and the
exact levels and slots that admit you.

### 3. Read readiness before you propose

Before you propose a merge, ask for a promotion or start a deployment, read the build's
**readiness** for that target. It lists everything missing in one call; report it rather than
inferring it from builds and runs.

```graphql
query Readiness($project: String!, $branch: String!, $build: String!) {
  builds(project: $project, branch: $branch, name: $build) {
    readiness(promotionLevel: "GOLD") {   # or: readiness(slotId: "…")
      ready
      missing { kind name message }
    }
  }
}
```

- Give exactly one of `promotionLevel` (a level of the build's branch) and `slotId`. Slot IDs come
  from `agentPolicy.slots`, from `Build.slots(environment:, qualifier:) { id }`, or from
  `yontrack slot list -p <project>`.
- **MCP:** `build_readiness` with `project`, `branch`, `build`, and either `promotionLevel` or
  `environment` (plus `qualifier` for a qualified slot).
- **CLI:** `yontrack build readiness -p <project> -b <branch> -n <build> --promotion GOLD` (or
  `--slot <slot-id>`), `-o json` for the items. Exit code `0` means ready, `2` not ready, `1` an
  error.

What each missing item asks of you:

| `kind` | What to do |
|---|---|
| `VALIDATION` | Produce the evidence: run the check and record it (step 4). |
| `PROMOTION`, `CHECK` | The build must first reach another level. Report it. |
| `ADMISSION_RULE` | The build does not qualify for the slot yet, or at all. Report the reason. |
| `MANUAL` | A person must promote or approve. **Stop and ask** — except on a level that admits you, where this item alone does not stop you (step 5). |
| `AGENT_POLICY` | The target does not admit agents. **Stop and ask** the owner named in the message. |

A merge proposal, a promotion request or a deployment request quotes the readiness: `ready`, and
each missing item's kind, name and message. When the readiness cannot be read, say so; do not
present a guess as a readiness.

Done when the proposal you are about to make carries the readiness of its target.

### 4. Record evidence as validation runs

Evidence is what an agent contributes freely. Record each check you ran as a **validation run** on
the build, with its data when the stamp takes data.

```graphql
mutation {
  createValidationRun(input: {
    project: "my-project", branch: "main", build: "42",
    validationStamp: "unit-tests",
    dataTypeId: "net.nemerosa.ontrack.extension.general.validation.TestSummaryValidationDataType",
    data: { passed: 412, skipped: 3, failed: 0 },
    description: "Unit tests run by the agent in session 01JC9V6Z3N"
  }) {
    validationRun { id runOrder }
    errors { message }
  }
}
```

- **CLI:** `yontrack validate -p … -b … -n … -v unit-tests tests --passed 412 --skipped 3 --failed 0`,
  or `junit --pattern '**/TEST-*.xml'`, `metrics`, `percentage`, `number`, `chml`, `findings`; plain
  `--status PASSED` for a check without data. `--evidence <file>` attaches the report itself.
- **MCP:** `create_validation_run` (when the server has mutations enabled) records a status and a
  description, **without data**. Use the CLI or GraphQL when the stamp takes data.

Rules:

- **Every number comes from a tool's output** — a test report, a coverage file, a scanner. Agents
  produce configuration and rules, never numbers. When you have no output, you have no data: record
  a status with a description of what was run, or record nothing.
- **Record failures too.** A `FAILED` run is evidence; leaving it out falsifies the record.
- **Some stamps refuse you.** A stamp with *Evidence from non-agents only* answers
  "evidence on <stamp> must come from a non-agent actor". That check needs a person or the CI; say
  so and move on.
- Recording the last missing validation may let **auto-promotion** promote the build. That is the
  level's own rule acting, not you promoting; it is expected.

Done when every check you ran is on the build as a validation run, with the tool's numbers when the
stamp takes data, and every refusal is reported.

### 5. Promote only where you are admitted, and only when ready

You may promote only when **both** hold:

1. the level is in your `agentPolicy.promotionLevels` (it has *Agents admitted*), and
2. the build's readiness for that level is `ready: true`, or its only missing item is `MANUAL`
   ("promoted by a person") and nothing else.

```graphql
mutation {
  createPromotionRun(input: {
    project: "my-project", branch: "main", build: "42", promotion: "BRONZE",
    description: "Promoted by the agent: readiness reported nothing else missing"
  }) {
    promotionRun { id }
    errors { message }
  }
}
```

- **MCP:** `promote_build` (mutations enabled) with `project`, `branch`, `build`, `promotion`.
- **CLI:** `yontrack promote -p … -b … -n … -l BRONZE -d "…"`.

On any other level — one that does not admit agents, or one whose readiness lists more than
`MANUAL` — your move is a **request**: tell your owner the build, the level and the readiness, and
let them promote.

Done when you have either promoted on an admitting level with a clean readiness, or handed the
promotion to your owner with its readiness.

### 6. Deployments: start, then stop at the manual rule

You may start a deployment pipeline only in a slot listed in `agentPolicy.slots`.

```graphql
mutation {
  startSlotPipeline(input: { buildId: 1234, slotId: "…" }) {
    pipeline { id status }
    errors { message }
  }
}
```

- **CLI:** `yontrack slot pipeline start -p <project> -e <environment> -b <build>` (default slot of
  the environment; for a qualified slot, use GraphQL with its `slotId`).
- **MCP:** no dedicated tool; send the mutation through `graphql_query` when mutations are enabled.

Then:

- When the slot has `manualApproval: true` — or readiness lists a `MANUAL` rule — the pipeline waits
  as a **candidate**. Tell your owner that the pipeline is started and waits for their approval,
  with the pipeline's link. Your part ends there.
- Approving the manual rule is a person's act. Setting the rule's data as an agent is refused with
  "an agent cannot approve; ask <owner>", even when the rule lists you. Overriding a rule and
  cancelling a pipeline are never granted to agents.
- When no manual rule stands in the way, the slot's other admission rules decide; read the
  readiness for the slot and report what is missing.

Done when the pipeline is started in an admitting slot and your owner knows it waits for them, or
when you have reported why it could not start.

## On a refusal

Yontrack's refusals name the policy and the person to ask:

```
agent claude-code-ci[agent] may not promote to GOLD: the promotion level does not admit agents (agent policy)
agent claude-code-ci[agent] may not start a pipeline on <slot>: the slot does not admit agents (agent policy)
agent claude-code-ci[agent] may not ProjectEdit (agent policy)
an agent cannot approve; ask <owner>
evidence on security-scan must come from a non-agent actor
```

They arrive in the GraphQL `errors`, or as the body of an HTTP 403.

1. Read the message: it says what was refused and why.
2. Report it to your owner — the action, the target, the message — and ask how to proceed.
3. Wait for a person to act or to change the policy.

The refusal is the answer. Retrying with other credentials — your owner's token, a CI token, a
colleague's — or reaching the same result another way defeats the gate your owner set; the only
way past a gate is a person.

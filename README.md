# Issue Orchestrator

A reusable coding-agent skill for completing a feature from a specification and local Markdown issues. A long-lived controller dynamically selects one ready issue, delegates it to a fresh isolated implementation agent, independently verifies the result, saves durable state, and repeats.

The controller edits tracking metadata only. Workers implement exactly one issue each. Every correction also gets a new worker. v1 deliberately runs sequentially in one checkout, without worktrees, a merge agent, GitHub Issues, or automatic PRs/pushes.

## Install

From any project directory, install interactively:

```sh
npx skills@latest add Anuise/issue-orchestrator
```

Select only this skill:

```sh
npx skills@latest add Anuise/issue-orchestrator --skill issue-orchestrator
```

Choose an agent explicitly (project scope is the default):

```sh
npx skills@latest add Anuise/issue-orchestrator --skill issue-orchestrator --agent claude-code
npx skills@latest add Anuise/issue-orchestrator --skill issue-orchestrator --agent codex
```

For global installation, add `--global`:

```sh
npx skills@latest add Anuise/issue-orchestrator --skill issue-orchestrator --agent claude-code --global
npx skills@latest add Anuise/issue-orchestrator --skill issue-orchestrator --agent codex --global
```

Use `--yes` to skip installer prompts and `--copy` to copy instead of linking. Discover this repository's skills without installing, list installed skills, or search the skills directory:

```sh
npx skills@latest add Anuise/issue-orchestrator --list
npx skills@latest list
npx skills@latest find issue-orchestrator
```

Update installed skills, optionally selecting this skill and its scope:

```sh
npx skills@latest update
npx skills@latest update issue-orchestrator --project --yes
npx skills@latest update issue-orchestrator --global --yes
```

Commands use the space-separated `--skill issue-orchestrator` syntax checked against `skills` **1.7.0**. Both agent identifiers were exercised with real local installations. Global/update syntax was checked against CLI help; validation does not change your global agent configuration. `@latest` can change; consult `npx skills@latest --help` if a future version differs. See the [validation record](docs/validation.md) and [upstream CLI documentation](https://github.com/vercel-labs/skills).

## Use

Prepare the feature workspace:

```text
.scratch/aidms-v2.1.1/
├── spec.md
└── issues/
    ├── 01-field-verify-downloads.md
    ├── 02-field-verify-writes.md
    └── 03-list-both-job-families.md
```

Give each issue acceptance criteria and explicit dependencies where known:

```markdown
# 03 List both job families
Status: pending
Depends on: 01-field-verify-downloads.md

## Acceptance criteria
- The list includes both job families with their verified download fields.
- A behavioral test covers one record from each family.
```

In a runtime with slash-command skill invocation:

```text
/issue-orchestrator .scratch/aidms-v2.1.1
```

The portable natural-language invocation is:

```text
Use issue-orchestrator on .scratch/aidms-v2.1.1.
Continue until all executable issues are complete or genuinely blocked.
```

Paths resolve relative to the current repository root unless absolute. All issue files are required unless the spec or user explicitly says otherwise. Existing repository instructions still govern tests, commit permissions, and implementation conventions.

## Architecture and recovery

```text
spec + issue files + Git evidence
             ↓
recover → recompute dependencies → choose one ready issue
             ↑                            ↓
     persist verified state ← verify ← fresh one-issue worker
             ↓ when all issues are verified
     full verification + fresh read-only integration review
```

The controller retains a small set of paths, scheduling decisions, and evidence summaries. Workers reconstruct the necessary state from disk in a disposable context. Context isolation is mandatory; filesystem isolation is intentionally absent in v1. Use the host's native fresh-agent capability with parent-conversation inheritance disabled. This package does not enable delegation in a host that lacks it.

| Durable status | Meaning |
| --- | --- |
| `pending` | Unfinished; eligible only after dependencies and safety checks pass |
| `in_progress` | An attempt was persisted and is active or awaiting recovery |
| `blocked` | Evidence identifies a prerequisite, access, ownership, or retry blocker |
| `completed` | Acceptance criteria and implementation evidence independently verified |

`ready` and `invalid` are scheduling classifications, not extra status values. Numeric filenames break ties; dependency and risk analysis determine the next issue after every worker result. Blocked issues are reconsidered when their release conditions change. Other safe ready work continues.

Issue files retain attempt history, commit SHAs, acceptance evidence, and correction counts. Local `.orchestration/` evidence retains baselines, worker handles/results, exclusive-controller ownership, and integration results. Keep these local snapshots out of commits; they can contain pre-existing user changes. The controller can commit its own concise issue metadata separately when repository rules permit.

Resume by invoking the skill again with the same feature path. It first resolves worker liveness and reconciles interrupted attempts against Git. An old timestamp never authorizes duplicate work. Uncertain liveness or change ownership stops writes until resolved. Require exclusive checkout ownership through a runtime lease or a guaranteed single-controller environment; the feature-local recovery lock cannot prevent simultaneous invocations on different features. Local recovery requires the original filesystem evidence; it does not promise recovery after disk loss. The skill cannot keep running after its host exits or while usage limits prevent execution.

An initial attempt gets at most two correction attempts, counted durably across restarts and final integration. Exhaustion blocks that issue until explicitly authorized otherwise. The controller never fixes worker code itself.

## Completion and stops

Completion requires every required issue verified completed, no active attempt or unresolved blocker, passing final relevant tests/type checking/lint/integration review, committed implementation work, and preserved unrelated user changes. Repository instructions that prohibit commits are an explicit recorded exception.

Stop with actionable evidence when no ready work remains, the spec materially contradicts issues, required external access is missing, an irreversible operation requires authorization, writer/Git ownership cannot be resolved, or isolated delegation is unavailable. Missing inputs and dependency cycles cannot count as success. Unsafe partial worker edits can block other work in the shared checkout.

The skill does not invoke `/loop`, `/implement`, or `/implement-spec`, and does not depend on another implementation skill. Future parallel/worktree/tracker adapters are outside v1.

## Contents and validation

- [SKILL.md](skills/engineering/issue-orchestrator/SKILL.md): control loop and hard boundaries.
- [Worker contract](skills/engineering/issue-orchestrator/references/worker-contract.md): one-issue execution and compact result schema.
- [Scheduling](skills/engineering/issue-orchestrator/references/scheduling.md): dependency graph and ready frontier.
- [State machine](skills/engineering/issue-orchestrator/references/state-machine.md): durable transitions, recovery, and retry limits.
- [Verification](skills/engineering/issue-orchestrator/references/verification.md): independent acceptance and final integration gates.
- [Validation record](docs/validation.md): installation checks, scenario exercises, and runtime limitations.

No runtime dependency or executable orchestration service is bundled. Installation compatibility is verified separately from host-specific execution capability.

Licensed under [MIT](LICENSE).

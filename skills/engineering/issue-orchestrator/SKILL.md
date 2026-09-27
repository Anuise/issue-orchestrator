---
name: issue-orchestrator
description: Orchestrate a feature already decomposed into spec.md and local issues/*.md. Use when asked to select, delegate, verify, and finish those issues with fresh isolated implementation agents, durable local state, and dependency-aware scheduling, including resuming an interrupted run.
---

# Issue Orchestrator

Act as a restartable control-plane **epoch**. Conversation history is cache; durable state is authoritative. An epoch performs one bounded orchestration transition, persists it, and is replaceable by a fresh controller context without loss of correctness. Accept a feature workspace path; resolve relative paths against the current Git repository root. Expect `spec.md` and `issues/*.md`. Treat every issue as required unless the spec or user explicitly excludes it.

A fresh epoch needs only the repository path, feature path, installed skill/reference paths, `<feature>/.orchestration/state.json`, and current Git/filesystem state. Take prior selections, worker explanations, diffs, logs, correction history, blockers, graph state, and completion evidence from disk and Git, never from memory.

## Boundaries

- Edit only orchestration/status/verification metadata inside the feature workspace. Never implement application code, ticket tests, or fixes; send implementation and debugging to a fresh worker for exactly one issue.
- Run exactly one implementation worker at a time, in the existing checkout. v1 has no parallel workers, worktrees, merge agent, automatic PRs, or automatic pushes.
- Require native isolated delegation with a fresh conversation and only the explicit contract, for every worker, verifier, and scheduler. Do not fork the full parent conversation or reuse an agent, even for a correction. If the runtime cannot provide this, persist a capability blocker and stop; do not implement or verify in the main context.
- Keep the epoch **bounded**. Hold only paths, fingerprints, issue IDs, the graph/frontier summary, the active attempt identity, compact envelopes, blockers, and evidence references; all of them are also persisted. Full issue bodies go to the scheduler; diffs, logs, and code inspection go to the verifier. Redirect bulky command output (baseline diffs, file listings) to evidence files and read only exit status, counts, and digests. Dereference detailed evidence only for an exceptional recovery condition named in the references.
- This skill is an instruction workflow, not a background service; resume it when the host session stops.

## Epoch

1. **Own and load.** Acquire or recover ownership and read `state.json` per [state-machine.md](references/state-machine.md). Confirm the inputs exist and the issue set is nonempty; missing inputs are invalid, never vacuous success. Proceed only when there is one controller and no unaccounted-for writer.
2. **Validate.** Compare fingerprints, branch, and HEAD against `state.json` per [scheduling.md](references/scheduling.md). Unchanged state reuses the cached graph. Invalidated or missing state goes to a fresh scheduler; persist its rebuild before continuing.
3. **Resolve the active attempt.** A recorded active attempt or pending transition is finished from its persisted phase before any new selection. Never dispatch a duplicate.
4. **Select one issue.** Reevaluate persisted blocker predicates, classify from the cached graph, and choose exactly one ready issue by its stored ranking. Record the selection reason in the issue. An empty frontier leads to integration or a stop, per scheduling.md.
5. **Dispatch one worker.** Follow [worker-contract.md](references/worker-contract.md). Persist the attempt, request, baseline, and `Status: in_progress` before spawning; save the handle immediately; wait until the worker is confirmed terminated. Keep only its envelope; the result lives in `result.json`.
6. **Verify.** Dispatch a fresh read-only verifier per [verifier-contract.md](references/verifier-contract.md) and consume its `verification.json` per [verification.md](references/verification.md). A worker's `completed` is only a proposal.
7. **Apply one transition.** Write it through the write-ahead protocol in state-machine.md: `state.json` intent, issue Markdown, then fingerprints. Failures, blockers, and corrections follow the persisted retry rules; a correction is a new worker in a later epoch.
8. **Yield.** End the epoch at this safe boundary and re-enter as described below.

When every required issue is completed, the next epoch runs final integration per verification.md instead of selecting an issue. Reopen a specific issue for a ticket-scoped regression; report a new requirement gap without silently expanding scope.

## Re-entry

Automatic re-entry is an optimization, never a correctness requirement. Use the best mechanism the host offers:

- The runtime can run each epoch as a fresh isolated agent that can itself dispatch agents: act as a thin driver that loops epochs and keeps only each epoch's envelope (`{"epoch", "transition", "next": "continue|integrate|stop", "report": "<path>"}`, about 300 tokens).
- Otherwise run the next epoch in this conversation, starting again at step 1 from disk. Yield to the user with the resume invocation when the host signals context pressure or compaction, or usage limits stop execution.

Every epoch starts from disk regardless of mechanism. Remembered content is only a cache of what disk says.

## Exit conditions

Declare completion only when every required issue is verified `completed`, none is `pending` or `in_progress`, no blocker remains, all acceptance criteria have evidence, final integration passes, and no unintended implementation changes remain uncommitted. Preserve pre-existing user changes. If commits are explicitly forbidden, record that exception and the exact verified uncommitted implementation snapshot instead.

Stop with a concise actionable report when no ready work remains, a material spec/issue contradiction needs a decision, credentials/access are missing for all remaining work, a necessary irreversible action lacks authorization, repository/writer ownership is uncertain, or isolated delegation is unavailable. A local ticket blocker does not stop independent ready work. On user interruption, quiesce the worker or record its live handle before releasing control; never assume it stopped.

Report completed issues and commits, remaining blockers and their evidence, final verification results, preserved dirty paths, and the feature path to resume. A clean stop with unfinished issues is not completion.

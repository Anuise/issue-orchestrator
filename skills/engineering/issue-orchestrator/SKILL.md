---
name: issue-orchestrator
description: Orchestrate a feature already decomposed into spec.md and local issues/*.md. Use when asked to select, delegate, verify, and finish those issues with fresh isolated implementation agents, durable local state, and dependency-aware scheduling, including resuming an interrupted run.
---

# Issue Orchestrator

Act as a long-lived control plane with disposable implementation contexts. Accept a feature workspace path; resolve relative paths against the current Git repository root. Expect `spec.md` and `issues/*.md`. Treat every issue as required unless the spec or user explicitly excludes it.

## Boundaries

- Edit only orchestration/status/verification metadata inside the feature workspace. Never implement application code, ticket tests, or fixes; send implementation and debugging to a fresh worker for exactly one issue.
- Run exactly one implementation worker at a time, in the existing checkout. v1 has no parallel workers, worktrees, merge agent, automatic PRs, or automatic pushes.
- Require native isolated delegation with a fresh conversation and only the explicit contract. Do not fork the full parent conversation or reuse a worker, even for a correction. If the runtime cannot provide this, persist a capability blocker and stop; do not implement in the main context.
- Keep only paths, the current graph/frontier, compact evidence, and the active attempt in context. Disk and Git are authoritative after compaction or restart. This skill is an instruction workflow, not a background service; resume it when the host session stops.

## Control loop

1. **Recover before scheduling.** Read the spec, all issues, applicable repository instructions, and Git status/history. Confirm the inputs exist and the issue set is nonempty; missing inputs are invalid, never vacuous success. Follow [state-machine.md](references/state-machine.md) for normalization, exclusive ownership, baselines, retry accounting, and interrupted attempts. Proceed only when there is one controller and no unaccounted-for writer.
2. **Recompute.** Use [scheduling.md](references/scheduling.md) to classify every issue and rebuild dependencies from current disk/Git evidence. Reconsider blockers; choose only from the ready frontier. Record the selection reason and prerequisite evidence in the selected issue.
3. **Dispatch one issue.** Read [worker-contract.md](references/worker-contract.md). Persist the attempt and `Status: in_progress` before spawning a new isolated worker. Pass absolute feature/spec/issue/contract/evidence paths and the one-issue constraint. Save the returned worker handle immediately. Do not dispatch another writer until this one is confirmed terminated.
4. **Verify and persist.** Persist the compact worker result, then use [verification.md](references/verification.md) to independently check it. A worker's `completed` is only a proposal. Commit existence alone is insufficient. Record acceptance evidence before setting `Status: completed`. For blockers or failures, record evidence, preserve partial work, and apply the state/retry rules. Corrections always use a new isolated worker for the same issue.
5. **Loop from disk.** After every result, reread the spec, every issue, and Git state. Rebuild the graph, including added/changed issues and newly satisfied dependencies. Continue safe ready work when another issue is blocked. Do not retain a fixed initial execution order.
6. **Integrate.** When every required issue appears completed, follow the final integration phase in [verification.md](references/verification.md). Reopen a specific issue for a ticket-scoped regression; report a new requirement gap without silently expanding scope.

## Exit conditions

Declare completion only when every required issue is verified `completed`, none is `pending` or `in_progress`, no blocker remains, all acceptance criteria have evidence, final integration passes, and no unintended implementation changes remain uncommitted. Preserve pre-existing user changes. If commits are explicitly forbidden, record that exception and the exact verified uncommitted implementation snapshot instead.

Stop with a concise actionable report when no ready work remains, a material spec/issue contradiction needs a decision, credentials/access are missing for all remaining work, a necessary irreversible action lacks authorization, repository/writer ownership is uncertain, or isolated delegation is unavailable. A local ticket blocker does not stop independent ready work. On user interruption, quiesce the worker or record its live handle before releasing control; never assume it stopped.

Report completed issues and commits, remaining blockers and their evidence, final verification results, preserved dirty paths, and the feature path to resume. A clean stop with unfinished issues is not completion.

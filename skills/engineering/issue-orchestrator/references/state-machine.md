# Durable state and recovery

## Authority and ownership

Issue Markdown is the durable ticket tracker. Use only `pending`, `in_progress`, `blocked`, and `completed` in a single top-level `Status:` field near each issue's title. `ready` and `invalid` are computed classifications; worker `failed` is a result, not a fifth durable status.

Preserve valid statuses and all original requirements. Add `Status: pending` when missing or unrecognized, recording the original value. Treat multiple conflicting status fields as invalid and request resolution; never guess which one wins. Verify existing completion evidence before using it as a prerequisite. Code presence by itself is insufficient.

Use `<feature>/.orchestration/` for local run evidence. Require exclusive checkout ownership for the entire run: use a runtime-provided checkout-scoped exclusive lease if available, otherwise run only in an environment that guarantees a single controller session for this checkout. If exclusivity cannot be established, record an ownership blocker and stop. A scan of jobs or feature locks alone cannot establish exclusivity: two different features could start simultaneously. Never run concurrent invocations for different features in one checkout.

Within that exclusive session, acquire same-feature recovery ownership using an atomic create-directory operation at `.orchestration/controller.lock` (creation must fail if it exists; do not use force/recursive mkdir as a lock). Store controller/session identity, repository root, branch, initial HEAD, and active issue/worker handle inside it. If a pre-existing lock or another writer cannot be explained, stop safely. This feature-local lock is not a checkout-wide mutex; do not claim that it prevents cross-feature races or arbitrary external writes.

On recovery, establish whether the recorded controller and worker are running. Reattach only to observe/wait for the SAME active attempt; do not resume it with a different ticket or dispatch a duplicate. A timeout, old timestamp, lost UI, or lost conversation is not proof of death. If runtime liveness cannot be established, request confirmation that the old writer has terminated. Once all writers are confirmed stopped, archive the old lock's evidence and acquire ownership atomically. On a normal exit, release only the lock you own, after the worker has stopped and state is persisted.

Keep evidence local; do not stage snapshots, result logs, or lock contents. They may contain user changes. Durability survives a local restart, not loss of the filesystem. Completed issue metadata should contain enough concise evidence for another checkout; local evidence paths support recovery on this checkout.

## Attempt record

Before normalizing or otherwise editing feature files, record staged/unstaged/untracked paths and baseline HEAD. Save binary-capable staged/unstaged diffs plus copies/hashes of pre-existing untracked files in the local evidence directory; exclude `.orchestration/` itself from snapshots to avoid recursively copying evidence. Include user-authored spec/issue changes in the protected baseline. Never put credentials or file contents in worker prompts. Preserve those baselines for the entire run. Record enough file content evidence to compare user changes after a commit, not just their filenames.

Before each attempt, append the following to an orchestrator-owned section of its issue, preserving previous attempt records:

```text
## Orchestration
Dependencies: issues/01-foundation.md (or none; include inferred-edge evidence)
Selection: prerequisite for 03; acceptance evidence at <path>
Attempt: <unique id>
Kind: initial | correction
Corrections used: 0
Baseline HEAD: <sha>
Baseline evidence: .orchestration/<attempt>/baseline/
Worker: dispatch_pending
Result: .orchestration/<attempt>/result.json
Blocker: none
```

These are metadata examples; substitute actual values. Persist all fields together with `Status: in_progress` using a same-directory temporary file and atomic replacement where supported; reread to confirm. Never overwrite a concurrently changed issue. Immediately replace `dispatch_pending` with the runtime handle after spawning. A crash in that gap requires runtime inspection; it never authorizes a second worker by default.

The worker writes its result to the provided local path before returning. The controller persists verification evidence and the next status together. If the host fails after a code commit but before a status update, recover from the attempt baseline, Git history/diff, and result file; do not implement twice.

## State transitions

| From | Evidence/event | To/action |
| --- | --- | --- |
| `pending` | Ready, baseline saved, dispatch starting | `in_progress` |
| `in_progress` | Result and independent acceptance verification pass | `completed` |
| `in_progress` | Genuine prerequisite/access blocker | `blocked`; record evidence and a concrete unblock predicate |
| `in_progress` | Failed attempt; another correction remains | `pending`; preserve diff and failure evidence |
| `in_progress` | Failed attempt; correction budget exhausted | `blocked`; record retry exhaustion |
| `blocked` | Recorded unblock predicate is now demonstrably true | `pending`; retain history/counter |
| `completed` | Verification demonstrates invalid completion or ticket regression | `pending` for a correction, or `blocked` if budget exhausted |

For inferred dependencies, a pending issue can remain `pending` but classify as blocked; persist the edge and reason. Do not relabel a completed ticket just because a prerequisite is later reopened; reverify its affected acceptance criteria and reopen only on evidence.

Allow an initial attempt plus **at most two correction attempts per issue**. Increment and persist `Corrections used` BEFORE dispatching each correction, including corrections after final integration. Worker failure, insufficient verification, and an interrupted attempt with uncertain effects use this same budget; restarting the session does not reset it. A confirmed genuine prerequisite blocker does not consume a correction by itself: after it clears, resume with a fresh initial-kind attempt unless a correction was already being attempted. Count any dispatched correction conservatively if its outcome was lost. Reset an exhausted budget only on explicit user authorization, recorded in the issue; merely completing a dependency cannot reset exhaustion.

For a dead worker with no result: inspect the recorded Git interval and partial diff. If acceptance can be independently verified, record recovered completion; otherwise use a fresh same-issue correction within budget. Never reset a stale `in_progress` blindly. If HEAD/branch diverged unexpectedly, baseline evidence is missing, or changes cannot be attributed, mark an evidence blocker and stop unsafe writes.

## Partial work and completion records

Preserve failed/blocked worker edits and report them. A correction worker may reconcile edits attributable to the same issue. Continue other ready issues only when partial edits cannot contaminate their touched files, prerequisites, tests, or commits. If isolation cannot be established in this shared checkout, block the affected frontier and request resolution; never hide the problem by reset/clean/stash or committing unfinished work.

On verified completion, write concise durable evidence in the issue itself:

```text
Status: completed

## Implementation
Commit: <full sha, or null with repository prohibition>
Acceptance: <criterion -> inspected behavior/test evidence>
Verification:
- <exact command>: passed; <scope/result summary>
- Typecheck: skipped; <specific reason it does not apply>
- Lint: passed; <exact command>
Verified at HEAD: <sha>
```

Store all implementation/correction SHAs when an issue spans multiple commits. Record test exit status and meaningful summaries; `skipped` without an accepted reason cannot satisfy a required check. Keep blocker history and attempt counters even after completion. Metadata may remain dirty during the run. At a safe boundary, the controller may commit only its own feature-tracking metadata when repository rules permit; never stage the entire feature directory or mix user-authored spec edits into that commit.

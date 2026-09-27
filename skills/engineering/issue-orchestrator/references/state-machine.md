# Durable state and recovery

## Authority

Authority runs in this order; a lower layer never overrides a higher one:

1. Git and source files: implementation state.
2. Issue Markdown: human-visible requirements, ticket status, attempt history, and correction counters.
3. `<feature>/.orchestration/state.json`: the compact orchestration snapshot (fingerprints, graph, frontier, active attempt, evidence paths).
4. Conversation memory: never authoritative.

When `state.json` disagrees with issue Markdown or Git, rebuild the affected state from the higher layer, except for a recorded `pending_transition`, which is the controller's own in-flight write (see the transition protocol). For correction counters, take the larger of the two values.

## Issue status

Issue Markdown is the durable ticket tracker. Use only `pending`, `in_progress`, `blocked`, and `completed` in a single top-level `Status:` field near each issue's title. `ready` and `invalid` are computed classifications; worker `failed` is a result, not a fifth durable status.

Preserve valid statuses and all original requirements. Add `Status: pending` when missing or unrecognized, recording the original value. Treat multiple conflicting status fields as invalid and request resolution; never guess which one wins. Verify existing completion evidence before using it as a prerequisite. Code presence by itself is insufficient, and so is a pre-ticked `- [x]` acceptance checkbox.

## `state.json`

The controller is its only writer. Canonical issue and evidence paths are feature-relative (`issues/03-auth.md`, `.orchestration/attempts/03-a1/worker.log`) in `state.json`, agent results, and issue records; payloads add absolute paths for reading. Shape, with actual values substituted:

```json
{
  "schema_version": 2,
  "repo_root": "/abs/repo",
  "feature_path": "/abs/repo/.scratch/example",
  "branch": "main",
  "initial_head": "<sha>",
  "current_head": "<sha>",
  "epoch": 7,
  "spec": {"path": "spec.md", "fingerprint": "sha256:..."},
  "instructions": {"fingerprint": "sha256:...", "commits_allowed": true, "notes": "<rules the controller needs>"},
  "issue_inventory": {"fingerprint": "sha256:...", "count": 12},
  "issues": {
    "issues/02-api.md": {
      "fingerprint": "sha256:...",
      "status": "pending",
      "classification": "ready",
      "dependencies": [{"issue": "issues/01-foundation.md", "evidence": "<explicit, or inferred-edge reason>"}],
      "rank": {"unblocks": 3, "risk": "<short note>", "order": 2},
      "commits": [],
      "corrections_used": 0,
      "blocker": null,
      "verification": null,
      "last_attempt": null
    }
  },
  "ready_frontier": ["issues/02-api.md"],
  "active_attempt": null,
  "pending_transition": null,
  "integration": {"status": "pending", "run_id": null, "verifier": null, "verifier_retries": 0, "phase": null, "result": null},
  "last_rebuild": ".orchestration/scheduler/r3/graph.json"
}
```

A blocker is `{"reason", "evidence", "unblock_predicate"}` with an observable predicate. `commits` lists every implementation/correction SHA. `verification` is the path of the accepting `verification.json`. `last_attempt` is `{"id", "kind", "outcome", "verification"}` for the most recent finished attempt, so a correction finds its evidence by path. `integration.status` is `pending`, `running`, `passed`, `failed`, or `blocked`; `integration.verifier` has the same `{id, handle}` shape as an attempt's. Reopening any issue resets `integration` to `pending` with a null `run_id` and zero retries.

An active attempt is:

```json
{
  "id": "03-a2",
  "issue": "issues/03-auth.md",
  "kind": "correction",
  "phase": "worker_running",
  "dir": ".orchestration/attempts/03-a2/",
  "baseline_head": "<sha>",
  "worker": "<runtime handle, or dispatch_pending>",
  "worker_result": {"status": "completed", "result": "<result.json path>"},
  "verifier": {"id": "v1", "handle": "<runtime handle>"},
  "verifier_retries": 0
}
```

Phases, in order: `dispatch_pending`, `worker_running`, `worker_returned`, `verifying`, `verdict_accepted`. Integration uses `verifying` and `verdict_accepted` too. `pending_transition` is `{"issue", "change", "evidence", "fingerprint_before", "fingerprint_after"}`.

Write every update to a same-directory temporary file, replace atomically where supported, and reread to confirm. Refuse a `schema_version` newer than this document knows and stop for an upgrade. A missing, unparsable, or older state file (including a run started before `state.json` existed) triggers a full scheduler rebuild from issue Markdown, Git, and existing attempt evidence; the old issue records are carried forward, not reset. For an issue left `in_progress`, the controller then reconstructs `active_attempt` from the latest `Attempt:`/`Worker:` record and the attempt directory: a `verifications/*/verification.json` that passes the acceptance checks gives `verifying` with that verifier; a `result.json` gives `worker_returned`; otherwise `worker_running` with the recorded handle, or `dispatch_pending` without one. Epoch recovery then resolves that phase.

## Filesystem mailbox

Disposable agents communicate through files. Each returns only a small envelope: status, result path, and a critical blocker when it could not write the file. Verbose material stays on disk, where the controller reads metadata first and dereferences detail only when needed.

```text
<feature>/.orchestration/
├── state.json
├── controller.lock/
├── baseline/                 run-start user baseline
├── scheduler/<rebuild-id>/   request.json, graph.json, scheduler.log
├── attempts/<attempt-id>/
│   ├── request.json          dispatch payload, as written
│   ├── baseline/             attempt-start staged/unstaged/untracked evidence
│   ├── result.json           worker result
│   ├── worker.log, tests.log
│   └── verifications/<verifier-id>/
│       ├── request.json      verifier payload
│       ├── verification.json verifier result
│       ├── verifier.log, diff.patch
│       └── logs/
└── integration/<run-id>/     one per integration verifier: request.json, logs/, result.json, verifier.log
```

Keep evidence local; do not stage snapshots, result logs, or lock contents. They may contain user changes. Durability survives a local restart, not loss of the filesystem. Completed issue metadata should contain enough concise evidence for another checkout; local evidence paths support recovery on this checkout.

## Ownership

Require exclusive checkout ownership for the entire run: use a runtime-provided checkout-scoped exclusive lease if available, otherwise run only in an environment that guarantees a single controller session for this checkout. If exclusivity cannot be established, record an ownership blocker and stop. A scan of jobs or feature locks alone cannot establish exclusivity: two different features could start simultaneously. Never run concurrent invocations for different features in one checkout.

Within that exclusive session, acquire same-feature recovery ownership using an atomic create-directory operation at `.orchestration/controller.lock` (creation must fail if it exists; do not use force/recursive mkdir as a lock). Store controller/session identity, epoch, repository root, branch, HEAD, and active issue/worker handle inside it. If a pre-existing lock or another writer cannot be explained, stop safely. This feature-local lock is not a checkout-wide mutex; do not claim that it prevents cross-feature races or arbitrary external writes.

Hold the lock for the epoch. Release it at the yield boundary when the next epoch may run in another context, after the worker/verifier has stopped and state is persisted; an epoch continuing in the same session may keep it and update its epoch number. Release only a lock you own.

On recovery, establish whether the recorded controller and worker are running. Reattach only to observe/wait for the SAME active attempt; do not resume it with a different ticket or dispatch a duplicate. A timeout, old timestamp, lost UI, or lost conversation is not proof of death, and conversation loss alone is not data loss. A driver that observed its epoch agent terminate may confirm that controller's death; otherwise, if runtime liveness cannot be established, request confirmation that the old writer has terminated. Once all writers are confirmed stopped, archive the old lock's evidence and acquire ownership atomically.

## Baselines and attempt records

Before normalizing or otherwise editing feature files, record staged/unstaged/untracked paths and baseline HEAD in `baseline/`. Save binary-capable staged/unstaged diffs plus copies/hashes of pre-existing untracked files; exclude `.orchestration/` itself to avoid recursively copying evidence. Include user-authored spec/issue changes in the protected baseline. Write these with redirected commands; the controller reads only digests and counts. Never put credentials or file contents in agent prompts. Preserve baselines for the entire run, with enough content evidence to compare user changes after a commit, not just their filenames. Capture the same evidence in `attempts/<id>/baseline/` before each attempt.

Before each attempt, append the following to an orchestrator-owned section of its issue, preserving previous attempt records:

```text
## Orchestration
Dependencies: issues/01-foundation.md (or none; include inferred-edge evidence)
Selection: prerequisite for 03; acceptance evidence at <path>
Attempt: <unique id>
Kind: initial | correction
Corrections used: 0
Baseline HEAD: <sha>
Baseline evidence: .orchestration/attempts/<attempt>/baseline/
Worker: dispatch_pending
Result: .orchestration/attempts/<attempt>/result.json
Blocker: none
```

These are metadata examples; substitute actual values. Persist all fields together with `Status: in_progress` through the transition protocol, with `active_attempt.phase` set to `dispatch_pending`. Never overwrite a concurrently changed issue. Immediately after spawning, record the runtime handle in `state.json` (`active_attempt.worker`, phase `worker_running`), then in the issue's `Worker:` field through the transition protocol. A crash in that gap requires runtime inspection; it never authorizes a second worker by default.

## Transition protocol

Issue Markdown and `state.json` cannot be replaced together atomically, so every controller write to an issue file (normalization, attempt record, worker handle, status, completion record) is a write-ahead transition. Edit only the `Status:` line, acceptance checkbox markers (see completion records below), and orchestrator-owned sections.

1. Hash the issue and require it to equal the recorded fingerprint. A mismatch is an external change: run invalidation first. Write the new content to a same-directory temporary file and hash it.
2. Write `pending_transition` with `fingerprint_before`, `fingerprint_after`, and the intended state change to `state.json`.
3. Replace the issue with the temporary file atomically, and confirm its hash equals `fingerprint_after`. Confirmation needs no reread of the body.
4. In one `state.json` write: record `fingerprint_after` as the issue fingerprint and apply the status/graph/frontier change. When the attempt is finished (`completed`, `blocked`, or back to `pending`), set `last_attempt` and clear `active_attempt`. Clear `pending_transition`.

A recorded fingerprint is always the fingerprint of the file as the controller last wrote it, so any other mismatch is an external change. On recovery with a `pending_transition`, hash the issue: equal to `fingerprint_after`, finish step 4; equal to `fingerprint_before`, repeat from step 1; anything else is an external change, so clear the transition and run invalidation. Never record an external edit as the controller's own.

## Epoch recovery

Resolve the recorded phase before selecting new work:

| Phase | Recovery |
| --- | --- |
| `dispatch_pending` | Inspect the runtime for a spawned worker. Adopt it if found. Dispatch the same attempt only when the runtime proves no worker was spawned; otherwise request confirmation. |
| `worker_running` | Observe/wait for the recorded worker. Once it is confirmed stopped, advance to `worker_returned` whether or not `result.json` exists. |
| `worker_returned` | Dispatch a fresh verifier. A missing result goes to the verifier in recovery mode; never re-implement. |
| `verifying` | A `verification.json` at the recorded verifier's path that passes the acceptance checks in [verification.md](verification.md) is accepted without re-verifying. Otherwise resolve the recorded verifier's liveness (a `dispatch_pending` handle as for a worker), then apply the `incomplete` rule in verification.md, including its retry limit. |
| `verdict_accepted` | Apply the transition implied by the accepted `verification.json`. |

Integration recovers the same way, using `integration.run_id`, `integration.verifier`, `integration.verifier_retries`, and `integration.phase`.

Verifiers run checks in the shared checkout, so a replacement verifier waits until the previous one is confirmed stopped, under the same liveness rules as workers. If HEAD/branch diverged unexpectedly, baseline evidence is missing, or changes cannot be attributed, mark an evidence blocker and stop unsafe writes. Never reset a stale `in_progress` blindly.

## State transitions

| From | Evidence/event | To/action |
| --- | --- | --- |
| `pending` | Ready, baseline saved, dispatch starting | `in_progress` |
| `in_progress` | `verification.json` accepted as `verified` | `completed` |
| `in_progress` | Genuine prerequisite/access blocker | `blocked`; record evidence and a concrete unblock predicate |
| `in_progress` | Failed attempt; another correction remains | `pending`; preserve diff and failure evidence |
| `in_progress` | Failed attempt; correction budget exhausted | `blocked`; record retry exhaustion |
| `blocked` | Recorded unblock predicate is now demonstrably true | `pending`; retain history/counter |
| `completed` | Verification demonstrates invalid completion or ticket regression | `pending` for a correction, or `blocked` if budget exhausted |

For inferred dependencies, a pending issue can remain `pending` but classify as blocked; persist the edge and reason. Do not relabel a completed ticket just because a prerequisite is later reopened; reverify its affected acceptance criteria and reopen only on evidence. When reopening a completed issue, in the same transition restore `- [ ]` on each criterion the regression evidence names; keep the other ticks.

Allow an initial attempt plus **at most two correction attempts per issue**. Increment and persist `Corrections used` in the issue and `state.json` BEFORE dispatching each correction, including corrections after final integration. Worker failure, insufficient verification, and an interrupted attempt with uncertain effects use this same budget; restarting the session or replacing the controller does not reset it. A confirmed genuine prerequisite blocker does not consume a correction by itself: after it clears, resume with a fresh initial-kind attempt unless a correction was already being attempted. Count any dispatched correction conservatively if its outcome was lost. A verifier `incomplete` or unusable result is not a worker failure and consumes no correction; `verifier_retries` bounds it per [verification.md](verification.md). Reset an exhausted budget only on explicit user authorization, recorded in the issue; merely completing a dependency cannot reset exhaustion.

A correction worker receives evidence by path: the previous `verification.json`, `result.json`, and attempt directory, plus the integration `result.json` when integration reopened the issue. Never pass or resume a previous conversation.

## Partial work and completion records

Preserve failed/blocked worker edits and report them. A correction worker may reconcile edits attributable to the same issue. Continue other ready issues only when partial edits cannot contaminate their touched files, prerequisites, tests, or commits. If isolation cannot be established in this shared checkout, block the affected frontier and request resolution; never hide the problem by reset/clean/stash or committing unfinished work.

On verified completion, write concise durable evidence in the issue itself, taken from `verification.json`:

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
Verification evidence: .orchestration/attempts/<attempt>/verifications/<verifier-id>/verification.json
```

In the same transition, tick the acceptance checklist: for each `acceptance` entry with `result: passed` and a non-null `index`, change `- [ ]` to `- [x]` on the issue's checkbox at that index, and change nothing else on the line. Tick only when the line's text equals the entry's `criterion`; otherwise leave it unticked and list the index as a mismatch in the completion record.

Store all implementation/correction SHAs when an issue spans multiple commits. Record test exit status and meaningful summaries; `skipped` without an accepted reason cannot satisfy a required check. Keep blocker history and attempt counters even after completion. Metadata may remain dirty during the run. At a safe boundary, the controller may commit only its own feature-tracking metadata when repository rules permit; never stage the entire feature directory or mix user-authored spec edits into that commit. Record the resulting HEAD in `state.json` so it is not mistaken for an external commit.

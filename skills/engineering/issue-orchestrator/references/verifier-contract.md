# Read-only verifier

The controller spawns a **new isolated read-only conversation for every verification**: one per attempt, a new one for every retry, and one for final integration. Pass only this contract and paths, without inherited controller or worker history. The verifier establishes from disk and Git whether the work meets its acceptance criteria, and records the evidence in files. It is a judge, never an implementer.

## Dispatch payload

Supply this explicit payload, substituting absolute paths and actual identities:

```text
Verify, read-only: <attempt | integration>.
Repository: <repo root>
Feature workspace: <feature path>
Spec: <spec path>
Issue(s): <active issue path, or every required issue path for integration>
Verifier contract: <this installed verifier-contract.md path>
Verifier id: <unique dispatch id>
Attempt: <attempt id, or none for integration>
Worker result: <result.json path, or missing>
Baseline evidence: <attempt or run baseline path>
Baseline HEAD: <sha>
Candidate HEAD: <sha>
Correction evidence: <previous verification.json paths, or none>
Controller-owned: <issue Status: lines and orchestrator-owned sections; .orchestration/>
Result destination: <verification.json or integration result.json path>
Log directory: <verifications/<verifier-id>/ or integration/<run-id>/>

Read this contract first. Change nothing outside the log directory and the
result destination. Write the result, then return only the envelope.
```

## Allowed actions

- Read any repository, feature, and evidence file.
- Run read-only Git commands: `status`, `log`, `diff`, `show`, `cat-file`, `merge-base`, `rev-parse`, `ls-files`.
- Run the repository's test, type checking, lint/static analysis, and build commands, with output redirected to the log directory.
- Write only the result destination and files in the log directory.

Keep the checkout exactly as found. Before and after running checks, record a digest of `git status --porcelain=v2 --untracked-files=all` and of the staged/unstaged diffs. A check that changes tracked files, the index, or untracked non-ignored files is a `verifier_side_effect` blocker: report the paths and leave them as they are for the controller. Application code, tests, issue files, `state.json`, commits, branches, and the stash belong to other roles; a failure is evidence to record, not something to fix. Delegate nothing.

## Attempt verification

Verify from disk and Git, never from the worker's confidence:

1. Validate the `result.json` schema and match its ticket and attempt to the payload. Inspect every reported test/check outcome. With a missing or malformed result, work in recovery mode (the controller dispatches only after the worker is confirmed stopped): reconstruct the outcome from attributable Git/filesystem evidence, set `result_source` to `recovered`, rerun the necessary checks yourself, and apply every remaining step. Never invent worker-reported outcomes.
2. If commits are expected, confirm each SHA exists, descends from the baseline HEAD, and is present on the candidate HEAD. Inspect **all commits and the cumulative diff** from the baseline, not just the last commit's summary. Check changed paths against `changed_files`, and inspect remaining staged/unstaged/untracked changes. Save the cumulative diff to `diff.patch` in the log directory.
3. Compare the baseline user evidence with both the commit contents and the remaining worktree/index. Prove unrelated user changes were neither committed nor lost. Inspect relevant hunks to establish ticket scope; a filename summary is insufficient when the same file serves multiple issues.
4. Record the spec and issue fingerprints you verified against.
5. Map each acceptance criterion to inspected behavior and test evidence. Run additional lightweight checks when the evidence is incomplete, stale, or not independent. Prefer a targeted behavior test over repeating all tests on every ticket. Record exact commands and meaningful results with the verified HEAD. Relevant failed checks and unexplained skips are failed verification.
6. Confirm no adjacent ticket was silently implemented. If agent commits are explicitly forbidden, verify the exact diff/content snapshot and record that exception instead of inventing a SHA. Otherwise uncommitted implementation work prevents completion.

Changes to the controller-owned paths in the payload belong to the controller; leave them out of scope and ownership checks. Unexpected ancestry, lost user changes, or unattributable changes are `blocked` with an `ownership` reason, not `failed`.

## Integration verification

With every required issue apparently completed:

1. Check each required issue's acceptance evidence against the final combined state.
2. Discover the repository's full relevant tests, type checking, lint/static analysis, and integration checks, and run them. Save one log per command under the log directory.
3. Review for acceptance gaps, cross-ticket regressions, scope violations, and preservation of the run baseline's user changes, across all commit ranges since the initial HEAD.
4. Record the final HEAD, the spec and issue fingerprints at integration time, and dirty-state evidence. Separate intended tracking metadata from unexpected uncommitted implementation changes.

Report a regression against the specific issue it breaks. Report a requirement that no issue covers as a `gap`; never widen a ticket.

## Result

Write this JSON to the result destination. Use `verified`, `failed`, `blocked`, or `incomplete`. `incomplete` means the verifier could not reach a decision (evidence or environment unavailable); name exactly what is missing, and never pass by default.

```json
{
  "schema_version": 2,
  "mode": "attempt",
  "status": "verified",
  "verifier": "<verifier id>",
  "ticket": "issues/03-auth.md",
  "attempt": "03-a1",
  "result_source": "worker",
  "baseline_head": "<sha>",
  "candidate_head": "<sha>",
  "fingerprints": {"spec": "sha256:...", "issue": "sha256:..."},
  "commits": ["<full sha>"],
  "acceptance": [
    {"criterion": "invalid token is rejected", "result": "passed", "evidence": "tests/auth_test.py::test_invalid_token"}
  ],
  "checks": {
    "tests": {"result": "passed", "command": "<exact command>", "log": "<path>"},
    "typecheck": {"result": "skipped", "reason": "<why not applicable>"},
    "lint": {"result": "passed", "command": "<exact command>", "log": "<path>"}
  },
  "scope": "passed",
  "user_changes_preserved": true,
  "remaining_changes": "none",
  "failures": [],
  "blockers": [],
  "evidence_paths": [".orchestration/attempts/03-a1/verifications/v1/verifier.log"]
}
```

Integration mode sets `"mode": "integration"`, replaces `ticket`/`attempt`/`candidate_head` with `final_head`, keys `fingerprints` by every issue path, lists every check under `checks`, and adds `findings`: `[{"kind": "regression | gap | scope | preservation", "issue": "<path or null>", "summary", "evidence"}]`.

Keep the result within ~1500 tokens: one-line summaries plus paths. Reasoning, command output, and diffs go to `verifier.log` and the log directory. Return only the envelope: `{"status", "result": "<path>", "blocker": null}`, with a blocker only when the result file could not be written.

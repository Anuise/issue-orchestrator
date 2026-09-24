# Independent verification

## Per-attempt gate

Confirm the worker has terminated, persist its result, and verify from disk/Git rather than its confidence:

1. Match the result to the active ticket and attempt. Validate the result schema and inspect every reported test/check outcome. A missing/malformed result cannot establish completion. Recovery exception: after confirming the old worker has stopped, the controller may reconstruct and persist a result from attributable Git/filesystem evidence and independently rerun the necessary checks. Label it controller-recovered, never invent worker-reported outcomes, and require every remaining gate below. Otherwise treat the missing result as a recoverable failure.
2. If commits are expected, confirm the SHA exists, descends from the recorded baseline, and is present on current HEAD. Inspect **all commits and the cumulative diff** from the attempt baseline, not just the last commit's summary. Check changed paths against `changed_files` and inspect remaining staged/unstaged/untracked changes.
3. Compare the original user baseline with both the commit contents and remaining worktree/index. Prove unrelated user changes were neither committed nor lost. Inspect relevant hunks to establish ticket scope; a filename summary is insufficient when the same file serves multiple issues.
4. Map each acceptance criterion to inspected behavior and test evidence. Run or delegate additional lightweight checks when the evidence is incomplete, stale, or not independent. Prefer a targeted behavior test over repeating all tests on every ticket. Record exact commands and meaningful results with the verified HEAD. Treat relevant failed checks and unexplained skips as failed verification.
5. Confirm no adjacent ticket was silently implemented. If agent commits are explicitly forbidden, verify the exact diff/content snapshot and record that exception instead of inventing a SHA. Otherwise uncommitted implementation work prevents completion.

Keep this a verification task, not implementation debugging. For failure, record the failed criterion and evidence; follow the persisted correction budget in [state-machine.md](state-machine.md). Spawn a **new isolated correction worker for exactly the same issue** with evidence paths. Never fix code or tests in the controller. At two unsuccessful correction attempts, block the issue and recompute the safe ready frontier. Unexpected Git ancestry or unattributable changes require ownership resolution before further writes.

## Final integration

After all required issues appear completed:

1. Reread the spec and complete issue inventory; ensure no required issue disappeared, remains `pending`/`in_progress`, or has an unresolved blocker. Check acceptance evidence for every required issue.
2. Discover the repository's full relevant tests, type checking, lint/static analysis, and integration checks. Run them on the final combined state. Record commands, exit results, applicability of skips, HEAD, issue/spec content fingerprints, and dirty-state evidence in `<feature>/.orchestration/integration.md`. Persist a concise summary in issue metadata if local logs will not travel with the checkout.
3. Once there are no active implementation workers, spawn a fresh isolated **read-only** review agent if available. Pass the spec, all issue paths, baseline/final HEAD and evidence paths. Ask for acceptance gaps, cross-ticket regressions, scope violations, and preservation of user changes. Supply no implementation history from the conversation. The reviewer must not edit code, tests, or status. If no independent reviewer capability exists, record that limitation and perform the same read-only review; isolated implementation delegation remains mandatory.
4. Reopen the appropriate issue for a demonstrated ticket-scoped regression, retaining its correction counter; dispatch a fresh same-issue correction worker within budget. Rerun affected checks and final integration after corrections. Report previously unidentified requirement gaps for a decision; do not silently create or broaden tickets.
5. Inspect final Git status and commit ranges. Require all intended implementation work committed (except the explicit repository prohibition), preserve original unrelated changes, and account for remaining tracking metadata separately. Completion is invalidated by subsequent implementation/spec/issue changes; revalidate affected evidence before claiming success again.

If integration cannot run because of missing access/environment, persist the blocker and report unfinished verification. Passing isolated ticket tests is not a substitute for a passing integration gate.

# Independent verification

Deep verification runs in a fresh read-only verifier per [verifier-contract.md](verifier-contract.md). The controller decides from the compact result plus cheap identity checks, and keeps diffs, logs, and code out of its own context.

## Per-attempt gate

1. Confirm the worker has terminated. Persist its envelope in `active_attempt.worker_result` and set `phase: worker_returned`.
2. Create `verifications/<verifier-id>/` in the attempt directory with the payload as `request.json`. Persist the verifier id with handle `dispatch_pending` and `phase: verifying` before spawning a fresh verifier; record its handle immediately after. A missing or malformed `result.json` still goes to the verifier, in recovery mode.
3. Accept `verification.json` only when it parses, its `schema_version` is known, and its canonical ticket, attempt, and verifier id match the active attempt. Its `candidate_head` must equal the current HEAD on the recorded branch, and its spec and issue `fingerprints` must equal those recorded in `state.json`. Confirm with digest-sized commands (`git rev-parse`, `git cat-file -e`, `git merge-base --is-ancestor`) that each reported commit exists and descends from the baseline HEAD. `verified` additionally requires every acceptance criterion `passed`, no blocker, and no required check failed or skipped without a reason.
4. A result that passes step 3 with status `verified`, `failed`, or `blocked` is accepted: persist `phase: verdict_accepted`, then apply the outcome through the transition protocol in [state-machine.md](state-machine.md):
   - `verified`: `completed`, with the completion record written from `verification.json`.
   - `failed`: record the failed criteria and evidence path, then follow the persisted correction budget.
   - `blocked`: persist the blocker and its unblock predicate. An `ownership` blocker or a `verifier_side_effect` stops further writes until resolved.
   - `incomplete`, or a result that fails step 3, is not accepted, and the phase stays `verifying`. When `verifier_retries` is 0 and the previous verifier is confirmed stopped, persist `verifier_retries: 1` with a new verifier id, then dispatch one more fresh verifier as in step 2, with the previous result path as correction evidence. This consumes no correction.

Exceptional recovery is the only case where the controller dereferences detailed evidence: an unusable verification with `verifier_retries` already 1, or a result that contradicts the identity checks. Read only the specific evidence files it names, label any conclusion `controller-reviewed`, and record that the normal gate did not complete. If that targeted review cannot establish the outcome, persist an evidence blocker. Never fix code or tests in the controller.

For a failed attempt within budget, the next epoch spawns a **new isolated correction worker for exactly the same issue** with the evidence paths. At two unsuccessful correction attempts, block the issue and reclassify the safe ready frontier.

## Final integration

An epoch that finds every required issue apparently completed runs integration instead of selecting an issue:

1. From validated `state.json` and fingerprints, ensure no required issue disappeared, remains `pending`/`in_progress`, or has an unresolved blocker, and each has an accepted `verification.json`. Any fingerprint mismatch goes through invalidation first.
2. Once there are no active implementation workers, create `integration/<run-id>/`, and persist `integration` as `running` with the run id, verifier id, handle `dispatch_pending`, and `phase: verifying` before dispatch. Every integration verifier, including a retry under the same `verifier_retries` rule as step 4, gets a new run id and directory. Its `request.json` holds the spec path, all required issue paths, initial and final HEAD, the fingerprints, the run baseline, and the verification paths. Dispatch a fresh read-only verifier in integration mode. Supply no implementation history from the conversation.
3. Accept its `result.json` under the same identity rules as step 3 above, with `final_head` in place of `candidate_head`, then persist `phase: verdict_accepted` and the outcome status. Persist the compact summary in `state.json` `integration`, and a concise one in issue metadata if local logs will not travel with the checkout.
4. Route the accepted outcome. `passed`: go to step 5. For a `regression` finding, reopen that issue with its correction counter retained and `integration` reset to `pending`; pass the integration `result.json` as correction evidence; a fresh same-issue correction worker runs in a later epoch within budget, and integration reruns afterward. Findings that reopen no issue (`gap`, `scope`, `preservation`, or a check failure no single issue owns) leave `integration` `failed` or `blocked`: stop and report them for a decision; do not silently create or broaden tickets. An epoch does not rerun integration for a `failed` or `blocked` result while HEAD and all fingerprints are unchanged since that result.
5. Require all intended implementation work committed (except the explicit repository prohibition), original unrelated changes preserved, and remaining tracking metadata accounted for separately. Immediately before declaring completion, recompute the spec, instruction, inventory, and issue fingerprints and HEAD. When they equal those in the `passed` result, declare completion; any change runs invalidation and integration again. Completion is invalidated by subsequent implementation/spec/issue changes; revalidate affected evidence before claiming success again.

If integration cannot run because of missing access/environment, persist the blocker and report unfinished verification. Passing isolated ticket tests is not a substitute for a passing integration gate.

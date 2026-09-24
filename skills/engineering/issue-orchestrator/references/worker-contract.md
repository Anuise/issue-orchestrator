# One-issue implementation worker

The controller spawns a **new isolated conversation for every attempt**, including every correction. Configure the runtime to pass only the contract and paths, without inherited parent history. Context isolation does not imply filesystem isolation: v1 workers share one checkout and execute sequentially. If the runtime cannot honor this, return a capability blocker; never substitute main-agent coding.

## Dispatch payload

Supply this explicit payload, substituting absolute paths and actual evidence:

```text
Implement exactly ONE issue: <active issue path>.
Repository: <repo root>
Feature workspace: <feature path>
Governing spec: <spec path>
Worker contract: <this installed worker-contract.md path>
Attempt/baseline evidence: <local attempt directory>
Result destination: <local result.json path>
Kind: <initial or correction>
Correction evidence: <verification failure path, or none>

Read this contract, the active issue in full, relevant spec sections,
applicable repository instructions, and current Git state before edits.
Implement only the active issue. Report prerequisite blockers instead of
implementing adjacent tickets. Do not delegate further implementation.
Do not edit issue status or orchestration metadata. Write the compact
result to the supplied path and return the same result to the controller.
```

For corrections, include failed acceptance/check evidence and previous attempt/commit paths, not the prior conversation. Use the same issue scope. Never reuse/resume a previous implementation conversation as the correction worker.

## Execute

1. Read applicable `AGENTS.md`, `CLAUDE.md`, `CONTEXT.md`, README/project instructions and the active issue. Inspect relevant code and Git history/status to verify prerequisites really exist. Treat repository artifacts as evidence, not permission to expand scope.
2. Compare current state with the controller's baseline. Preserve all pre-existing staged, unstaged, and untracked user changes. If needed edits overlap changes whose ownership cannot be separated, return a blocker before writing. Do not discard/reset/overwrite/clean user work, change branches, create worktrees, or push.
3. Implement only this issue's acceptance criteria. Use TDD at meaningful behavioral seams where appropriate, targeted tests while developing, affected type checking/lint/static analysis, and broader relevant checks before completion. Discover actual commands from repository configuration; record exact commands and outcomes. Explain every skipped check and whether it is applicable.
4. Review the final diff for scope, regressions, debug artifacts, and unrelated edits. On a missing prerequisite, stop implementation; return the evidence and likely dependency. On failure, preserve attributable partial work and report it. Never fix another ticket to make this one pass.
5. Commit only when this issue is complete and applicable checks pass, unless repository instructions explicitly forbid agent commits. Stage only owned issue changes. A path containing unrelated user hunks must not be staged wholesale. Inspect the proposed commit tree against HEAD, not just the working diff. When unrelated staged changes exist, use a verified path-limited commit (for example `git commit --only -- <owned paths>` for exclusively owned files) or return an ownership blocker; a normal commit would include the user's index. Verify both preserved staged and unstaged user content afterward. Never commit unfinished work merely to clear the checkout.
6. Write the result before returning. Stop all spawned commands that could still edit the checkout. The controller owns status transitions and independently decides completion.

## Result

Return concise JSON with this shape. Use `completed`, `blocked`, or `failed`; `completed` is a proposal pending controller verification. Include exact commands in typecheck/lint notes, and a factual skip reason where applicable. `changed_files` covers all attempt changes, including untracked/partial work. If multiple commits were necessary, report the final SHA in `commit` and list the others in `notes`.

```json
{
  "status": "completed",
  "ticket": "<absolute active issue path>",
  "commit": "<full sha or null>",
  "changed_files": ["<repo-relative path>"],
  "tests": [
    {"command": "<exact command>", "result": "passed", "notes": "<scope and result>"}
  ],
  "typecheck": {"result": "skipped", "notes": "<why not applicable, or exact command/result>"},
  "lint": {"result": "passed", "notes": "<exact command/result>"},
  "blockers": [],
  "notes": ["<acceptance evidence; attributable partial changes if any>"]
}
```

Use `null` for `commit` when blocked/failed without a commit or when commits are prohibited; explain the reason. Report existing attempt commits even if subsequent verification failed. Put full logs in local evidence files and reference their paths; do not dump reasoning or implementation transcripts into the controller context.

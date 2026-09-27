# Dynamic scheduling

The dependency graph and ranking persist in `state.json`. A normal epoch reuses them after fingerprint validation and reclassifies from persisted statuses; a fresh scheduler rebuilds them only when fingerprints show they are stale. Never silently trust a cached graph whose inputs changed.

## Fingerprints

A file fingerprint is `sha256:` of its raw bytes, computed by a hashing command that prints only the digest (`sha256sum`, `Get-FileHash`), so file bodies stay out of the controller's context. Track:

- `spec.md`.
- Repository instruction files that govern the run (`AGENTS.md`, `CLAUDE.md`, and similar), combined as one fingerprint.
- The issue inventory: the SHA-256 of the sorted canonical issue paths, one per line. It changes on add, remove, or rename.
- Each issue file.
- Branch and HEAD, compared with `current_head`.

Record each issue fingerprint as the controller last wrote it (see the transition protocol in [state-machine.md](state-machine.md)), so every mismatch is an external change.

## Invalidation

Each epoch compares the current values with `state.json` and rebuilds only what changed:

| Signal | Rebuild scope |
| --- | --- |
| `state.json` missing, unparsable, or an older schema | Full |
| Spec or instruction fingerprint changed | Full |
| Inventory changed | Added/removed issues and every edge touching them; reconcile renames |
| Issue fingerprint changed externally | That issue, its dependents, and its dependencies; reconcile any externally changed status metadata |
| HEAD moved with no active attempt to explain it | Revalidate completed issues whose evidence the new commits touch; a `current_head` that is no longer an ancestor of HEAD is an evidence blocker |
| Recovery finds `state.json` contradicting Git or attempt evidence | Full |

Detection uses fingerprints and Git identity only; an epoch never compares issue bodies with `state.json`. A worker's commits during an active attempt descend from its baseline and are explained by that attempt; the verifier checks them. After each applied transition, set `current_head` to HEAD.

When nothing is invalidated, the epoch reads no issue bodies. It reclassifies from the cached graph, persisted statuses, the active attempt, and blocker predicates: an issue whose dependencies all became verified joins the ready frontier without a rebuild.

## Fresh scheduler

Run every rebuild in a fresh isolated read-only agent, including the first one. Write `scheduler/<rebuild-id>/request.json` with the repository, feature, spec, and `state.json` paths, the rebuild scope, the changed/added/removed issue paths, the fingerprints of the spec, instruction, and issue files, and `graph.json` as the destination. The scheduler reads the spec, the instructions, the issues in scope in full, other issues only as needed to infer edges against them, and relevant Git evidence. It applies the rest of this document and writes `graph.json` with:

- The normalized issue inventory, with normalization needs (for example, a missing `Status:`) for the controller to apply.
- Dependencies with evidence, classifications, and invalid/blocked reasons with unblock predicates.
- Ranking inputs and the ready frontier.
- Repository facts the controller needs (`commits_allowed` and other governing rules).
- Rename reconciliations and removed required issues.
- The fingerprint of every spec, instruction, and issue file it read.

The scheduler edits nothing else and returns only an envelope of at most ~1000 tokens: status, `graph.json` path, and any stop-level finding (spec contradiction, missing input). A cycle is not stop-level; it appears as blocked members with the cycle path. The controller rejects the graph and reruns the scheduler when those fingerprints differ from the current files; after two reruns, files are changing during the run, so stop and report. Otherwise it validates the graph against the inventory and persists the merged graph with the read fingerprints (on the first run, this creates `state.json`), then applies normalization edits through the transition protocol.

## Graph and classification

Use canonical issue paths (feature-relative, such as `issues/03-auth.md`) as identities; numeric filename prefixes are only tie-breakers. Extract explicit `Depends on:`/dependency statements from prose or metadata. Infer edges from required APIs, schemas/migrations, interfaces, services, prerequisite behavior, acceptance criteria, and shared components whose change order matters. Shared files alone do not prove dependency. Record the concrete evidence for inferred edges; do not invent foundational work beyond the spec. Newly discovered issues enter the graph; removed/renamed required issues need reconciliation against the spec and history, not silent omission.

An edge `A -> B` means A requires B. Satisfy it only with verified prerequisite acceptance evidence present in this checkout, using Git history as supporting evidence. A claimed SHA or `completed` label without verification is not enough. A missing referenced issue, ambiguous identity, untestable acceptance requirement, or material spec contradiction is invalid; a dependency cycle blocks its members with the cycle path as evidence. Never break a cycle by pretending an edge is optional.

| Classification | Decision |
| --- | --- |
| completed | Valid `completed` record, acceptance evidence still applicable |
| in_progress | Active/recovering attempt; resolve liveness before any new dispatch |
| invalid | Requirements, status fields, identity, or referenced dependency cannot be resolved safely |
| blocked | Unsatisfied edge, external prerequisite, unsafe partial changes, or exhausted correction budget |
| ready | Pending, all dependencies verified, acceptance clear, edits safe, runtime available |

Propagate blocked/invalid prerequisites to dependents. Preserve the original issue requirements and persist each blocker with an observable unblock predicate. Reevaluate those predicates every epoch: dependencies completed, credential/access restored, or a human decision recorded (which changes the issue fingerprint). Retry exhaustion requires explicit authorization to release. If a spec contradiction affects the feature's intended behavior, stop for that decision; independent local blockers can coexist with ready work.

## Choose exactly one ready issue

Prefer prerequisites that unblock multiple issues; then work that tests high-risk assumptions or reduces downstream uncertainty. Use numeric order only when dependency/risk considerations are equivalent, then canonical path for a stable tie-break. The scheduler persists these as `rank`; the controller applies them to the current frontier. Record a short selection reason rather than a speculative full sequence, and reclassify every epoch, even when the earlier order still looks plausible.

If the ready frontier is empty, distinguish:

- All required issues verified completed: enter final integration.
- A live/recovering attempt: wait/recover it; do not claim everything is blocked.
- Unfinished issues: report their blocker/invalid reasons, dependency/cycle evidence, and the smallest external decision or action needed. Do not claim completion or spin on unchanged state.

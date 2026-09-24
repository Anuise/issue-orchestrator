# Dynamic scheduling

Rebuild from the spec, every issue, the persisted attempt records, and current Git evidence after every completed, blocked, failed, or recovered worker. Newly discovered issues enter the graph; removed/renamed required issues need reconciliation against the spec and history, not silent omission.

## Graph and classification

Use canonical issue paths as identities; numeric filename prefixes are only tie-breakers. Extract explicit `Depends on:`/dependency statements from prose or metadata. Infer edges from required APIs, schemas/migrations, interfaces, services, prerequisite behavior, acceptance criteria, and shared components whose change order matters. Shared files alone do not prove dependency. Record the concrete evidence for inferred edges; do not invent foundational work beyond the spec.

An edge `A -> B` means A requires B. Satisfy it only with verified prerequisite acceptance evidence present in this checkout, using Git history as supporting evidence. A claimed SHA or `completed` label without verification is not enough. A missing referenced issue, ambiguous identity, untestable acceptance requirement, or material spec contradiction is invalid; a dependency cycle blocks its members with the cycle path as evidence. Never break a cycle by pretending an edge is optional.

| Classification | Decision |
| --- | --- |
| completed | Valid `completed` record, acceptance evidence still applicable |
| in_progress | Active/recovering attempt; resolve liveness before any new dispatch |
| invalid | Requirements, status fields, identity, or referenced dependency cannot be resolved safely |
| blocked | Unsatisfied edge, external prerequisite, unsafe partial changes, or exhausted correction budget |
| ready | Pending, all dependencies verified, acceptance clear, edits safe, runtime available |

Propagate blocked/invalid prerequisites to dependents. Preserve the original issue requirements and persist each blocker with an observable release condition. Reconsider those conditions each cycle: dependencies completed, credential/access restored, or a human decision recorded. Retry exhaustion requires explicit authorization to release. If a spec contradiction affects the feature's intended behavior, stop for that decision; independent local blockers can coexist with ready work.

## Choose exactly one ready issue

Prefer prerequisites that unblock multiple issues; then work that tests high-risk assumptions or reduces downstream uncertainty. Use numeric order only when dependency/risk considerations are equivalent, then canonical path for a stable tie-break. Record a short selection reason rather than a speculative full sequence. Recompute after the worker result, even when the earlier order still looks plausible.

If the ready frontier is empty, distinguish:

- All required issues verified completed: enter final integration.
- A live/recovering attempt: wait/recover it; do not claim everything is blocked.
- Unfinished issues: report their blocker/invalid reasons, dependency/cycle evidence, and the smallest external decision or action needed. Do not claim completion or spin on unchanged state.

# CMS Rebuild — Spec (to-spec output)

Parent PRD: [docs/prd.md](prd.md). Published to the tracker as the spec
issue; tickets are its sub-issues. Seams check: resolved by locked
decision (single capture-to-core API seam); no user check performed
(YOLO mode — recorded here instead).

## Problem Statement

Commissioning teams run completions on a Salesforce-based CMS whose
per-seat pricing breaks at field scale, whose bulk workloads (imports,
generation, batch printing) fight the platform, and whose field use
assumes connectivity the site does not have. Punch walks produce hours
of write-up per walk: photos sorted by hand, defects retyped, one
spreadsheet per trade, while final payment waits on the paperwork.

## Solution

An off-platform commissioning system: a punchwalk fork turns a narrated
walk into verified punch items with proof, and a new CMS core holds the
project tree, templates, checksheets, approvals, and handover documents.
Captures post as punch items onto tree nodes over one seam. Field crews
work offline; the office sees live completion, outstanding work, and
handover readiness.

## User Stories

Field capture:

1. As a field inspector, I want to record a narrated walk on my phone, so
   that the walk itself becomes the punch-list input.
2. As a field inspector, I want each defect to carry its photo with the
   defect outlined, so that nobody disputes what or where it is.
3. As a field inspector, I want my spoken words attached to each item, so
   that the write-up matches what I called out.
4. As a field inspector, I want items to post onto the right project-tree
   node, so that punch items land where the work is tracked.
5. As a field inspector, I want capture to queue offline and sync later,
   so that dead zones never lose a walk.
6. As a field inspector, I want a re-walk to sign off fixed items with
   before/after photos, so that false "fixed" claims get caught.

Project tree and completion:

7. As a commissioning engineer, I want the full hierarchy
   (project/system/sub-system/tag group/tag), so that completions mirror
   the plant structure.
8. As a commissioning engineer, I want parents to complete only when all
   children complete, so that readiness is never claimed early.
9. As a commissioning engineer, I want bulk tree import from CSV, so that
   project load is not manual entry.
10. As a project manager, I want planning dates and manhours per node with
    rollup, so that progress and forecast stay visible.

Templates and checksheets:

11. As a commissioning engineer, I want a standard ITI pool (check, test,
    approval, report field), so that templates reuse proven items.
12. As a commissioning engineer, I want bulk template creation from CSV,
    so that hundreds of templates load in one pass.
13. As a commissioning engineer, I want an assignment matrix over tag
    types/items/groups/systems, so that one template fans out correctly.
14. As a commissioning engineer, I want one-click checksheet generation
    from the matrix, so that every object gets its checks.
15. As a field lead, I want barcoded checksheets with batch printing, so
    that paper rounds stay organized.
16. As a document controller, I want scanned completions to re-enter by
    barcode, so that paper evidence links to the node.

Field execution:

17. As a technician, I want to complete checks and record results offline,
    so that the job moves without connectivity.
18. As a technician, I want one-tap PLI raising from a failed check,
    prefilled with the issue, so that defects are never lost.
19. As a contractor, I want my company's outstanding work view, so that my
    crews know exactly what is open.

Approvals and handover:

20. As an approver, I want approval chains with signatures on checksheets
    and handover docs, so that acceptance is binding.
21. As a commissioning manager, I want DAC/MCC/RFU flow driven by the
    rollup plus open PLIs, so that handover reflects reality.
22. As a QA lead, I want certificates and notices attached to nodes, so
    that handover packs are complete.

Recurrence, dispatch, reporting, access:

23. As a preservation coordinator, I want preservation templates with
    recurrence and auto dates, so that idle equipment stays protected.
24. As a maintenance planner, I want maintenance checklists with
    recurrence, so that upkeep is scheduled not remembered.
25. As a trade lead, I want items routed to my discipline with a review
    tap, so that my team gets clean work packs.
26. As a project manager, I want live outstanding-work views and a
    completions scorecard, so that meetings run from facts.
27. As a project controls engineer, I want manhour reports
    (planned/forecast/earned/actual), so that EVM stays honest.
28. As a system admin, I want role-based access with view/edit separation,
    so that crews see only what they may touch.
29. As an auditor, I want an append-only AI ledger of every AI action, so
    that machine decisions are inspectable.
30. As a migration lead, I want legacy CSV import with expand-contract
    where wide, so that history survives the move.

## Implementation Decisions

- Modules: `apps/core` (new FastAPI service + React/Vite PWA: tree,
  templates, checksheets, punchlist, approvals, handover, preservations,
  reporting, access, audit) and `apps/capture` (punchwalk fork: pipeline,
  PWA record flow, dispatch). One monorepo, one integration branch.
- Stack (new core mirrors the leaf to keep one toolchain): Python FastAPI
  + SQLModel, SQLite for dev/test with Postgres-compatible migrations for
  prod; React/Vite PWA with offline queue; pytest + httpx ASGI tests.
- The seam is the only cross-service interface: item submission with
  idempotency key plus re-walk verification (see PRD section 6). The leaf
  never writes core tables; the core never reads leaf tables.
- Testing seams (seams check outcome): core HTTP API is the highest seam
  for all new core behavior; the leaf keeps its existing API-level suite;
  one new seam-contract suite posts items and verifications end to end.
  No other new seams.
- Completion rollup is a synchronous invariant on write, never a batch
  recompute: a parent flips to complete only via its children.
- Checksheet generation is deterministic from (template version, matrix):
  same inputs regenerate identical sheets, so re-generation is safe.
- AI ledger entries are written in the same transaction as the state
  change they describe; ledger rows are immutable once written.
- Handover signatures are immutable once complete; docs render from
  frozen snapshots, not live queries.
- Migration uses expand-contract only where a form is wide; narrow
  imports land directly.

## Testing Decisions

- Test external behavior only: HTTP responses, generated sheets, posted
  items, rollup states, rendered docs. No implementation-detail tests.
- Core: API-level tests per slice over the HTTP seam, seeded through
  public endpoints (no direct DB seeding except auth bootstrap).
- Leaf: keep the forked pytest suite green; keep the eval harness and
  holdout walk passing with existing targets.
- Seam: contract tests for submit (link + idempotent re-post),
  verification (fixed/still-open/needs-a-look), offline queue replay,
  and auth scoping.
- Prior art: punchwalk `tests/test_api_*.py` (API level),
  `punchwalk/eval/` (grounding/verification quality gates).

## Out of Scope

Per PRD section 8: punchwalk roadmap items (live glasses streaming,
Procore sync, spec citations), physical robots, WinPCS beyond the domain
checklist, Salesforce export/cutover tooling (wizard, manual).

## Further Notes

- Slice 1 (factory floor) is complete on this repo: scaffold, CI green on
  `main` + `integration`, tracker labels, vendored skills.
- to-tickets quiz resolved in YOLO mode: 13 slices stand as 13 tickets
  (12 open, slice 1 done); granularity kept coarse-by-design since each
  ticket gets a fresh implementer context; split only if an implementer
  stalls. Blocking edges per the slice graph; long pole 1 > 3 > 5 > 6 > 7.
- gh 2.83 lacks native sub-issue/dependency flags, so ticket edges use
  `Part of` / `Blocked by` lines per the tracker doc fallback.
- Frontier order: slices 2+3 first in parallel, then 4+5, then the rest
  by unblocked edges.

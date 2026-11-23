# CMS Rebuild — Product Requirements Document

Status: draft for factory input (to-spec). Date: 2026-11-11.
Repo: `fraserbrownirl/commissioning-cms` (private monorepo).
Sources: `~/Downloads/Product.zip` (2020, ~165 files, no code),
`~/Downloads/WinPCS Present.ppt` (2007 domain deck, checklist-only),
`github.com/lucianobogo/punchwalk` (MIT), prior-session handoff
`cms-rebuild-handoff.md`.

## North star

We transform project delivery — by automating, simplifying, and
accelerating the final stages of major infrastructure projects.

## 1. Background and goal

Rebuild the force.com commissioning CMS off-platform as an AI-powered system.
A punchwalk fork owns field capture; a new CMS core owns the record. The two
meet at one API seam: captures post as punch items onto project-tree nodes.

Drivers for leaving Salesforce: per-seat economics at field scale, bulk
workloads (bulk template creation, assignment import, checksheet generation,
batch printing), and offline field use.

## 2. Locked decisions

1. Off-platform rebuild. Staying on Salesforce and hybrid both rejected.
2. Fork punchwalk (MIT) as the capture leaf; build the CMS core behind it.
   Capture is a separate service beside the core, over one API seam.
   Monorepo, so the factory gets one integration branch and shared CI.
3. Execution runs through the mattpocock skills factory to completion:
   PRD in, then to-spec (with seams check), to-tickets
   (granularity/edges quiz), implement-spec (task graph, frontier, fresh
   context per ticket, TDD at seams, merger subagents, final code-review).
   Wizard covers manual steps (API keys, infra, Salesforce export/cutover).
4. Tracker: GitHub repo + issues (this repo).
5. Domain invariants carried from WinPCS/Product.zip: hierarchy completion
   rollup (children complete, parent completes), DAC/MCC/RFU handover flow,
   per-contractor outstanding work views, role-based access with view/edit
   separation.

## 3. System overview

```
field device (PWA, offline-capable)
  |- capture leaf (punchwalk fork): walk video -> verified items + proof
  |     posts items over the seam
  v
CMS core: project tree, templates, checksheets, punchlist, approvals,
  handover docs, preservations, reporting, access + audit
```

The capture leaf never owns domain state beyond its own pipeline tables. The
core never ingests raw video. The seam carries verified items with evidence
(photo, quote, timestamp, severity, trade/discipline suggestion) and returns
node linkage plus lifecycle state (open, fixed-claimed, verified, closed).

## 4. CMS core — domain model

### 4.1 Project tree

Hierarchy levels, top to bottom: Project, System Group, System, Sub-System,
Sub-Sub System, Tagged Item Group (e.g. Piping Tests), Tagged Item. Areas
cross-cut the tree. Individual components attach to tagged items.
Every node type and every checksheet has an owner.

Invariants:

- Completion rollup: a parent completes only when all children complete.
- Node levels carry planning fields: planned/actual start, planned/actual
  completion, planned manhours, forecast, earned, actual, non-productive.

### 4.2 Tag types and setup order

Setup follows the legacy six steps: (1) tag-type hierarchy, (2) project-tree
load, (3) individual template items, (4) templates, (5) assignment matrix,
(6) checksheet generation. Bulk CSV import is required for steps 2-5 at
project scale, not manual entry.

### 4.3 Individual template items (ITI)

Standard pool of reusable items. Four types:

- check item: task requiring pass/fail;
- test item: task requiring a recorded test result;
- approval item: marks that generated checksheets need a specific approval;
- report field: custom field for extra recorded information.

### 4.4 Templates

Groups of ITIs used to generate checksheets. Template types include AITR,
BITR, Preservation, and others; type restricts which tree parts a template
may attach to. Bulk creation via CSV (`Template Name | Template Type |
Assigned ITI`, ITI referenced by unique ID, template names unique). The
responsible (contractor/completer) company is set at template setup and
shows at the top of generated ITRs.

### 4.5 Assignment matrix

Rules mapping templates to tree scope: tag type (T), tagged item (I),
tagged item group (G), system (S), etc. Bulk creation via CSV
(`Item ID | Assigned Templates`, comma-separated template names).
Generation expands the matrix into one checksheet per (template, object).

### 4.6 Checksheets

Generated per assignment with a unique code usable as a barcode. The
checksheet tab exists for organizing and printing (filter by type, system,
tag type/group, then batch-print or select-and-print), not as a report.
Completed paper scans re-enter via a barcode-driven import spreadsheet
(checksheet ID, URL, status). Visual checksheet layout must match legacy
expectations closely enough that field crews accept it.

### 4.7 Punchlist (PLI)

Punchlist items attach to tree nodes. Activity types: Construction Ad Hoc,
Construction Walkdown, Punchlist Squad, Mechanical Completion Ad Hoc,
Mechanical Completion Walkdown, Pre-Commissioning, Ready for Commissioning,
Pre Start-up Safety Review, Ready for Startup, Maintenance. Failed checks
offer one-tap PLI raising prefilled with the issue. Workflow rules escalate
(e.g. PLI upgraded to critical notifies followers). Per-contractor
outstanding work views are required.

### 4.8 Preservations and maintenance

Preservation templates carry recurrence like tasks, with auto start/end
dates (set on start or by import). Maintenance module mirrors this: activity
plus checklist template with recurrence.

### 4.9 Handover flow

DAC/MCC/RFU document flow with approval chains and signatures. Certificates
and notices attach to nodes. Handover readiness derives from the completion
rollup plus outstanding PLIs.

### 4.10 Manhours and reporting

Node-level planned/forecast/earned/actual/non-productive manhours roll up
the tree. Reports include engineer-centric views, live outstanding-work
views, and a completions scorecard. Graphical reports plus links to reports.

### 4.11 Isolations, inhibits, faults, disciplines

Isolations/inhibits exist only on tagged items, with in/out dates, permits,
refs, and a task area. Faults are a managed list with discipline lookup and
role-gated additions; check/test creation can promote reworded issues into
it. Discipline list seeds from legacy (piping, electrical, instrumentation,
mechanical, HVAC, structural, telecoms, QC, safety, etc.).

### 4.12 Access and audit

Role-based access with view/edit separation per object, seeded from the
legacy roles model. Every AI action core-wide writes to an append-only AI
ledger (what was proposed, decided, and by which model call).

## 5. Capture leaf — punchwalk fork

Upstream: `github.com/lucianobogo/punchwalk`, MIT, hackathon build.
FastAPI + SQLite + in-process worker, React/Vite PWA. Eight tables:
Project, Walk, Turn, Point, Item, Dispatch, ModelCall, Job.
Measured cost ~= $0.54 per 105s 12-item walk, ~$0.03-0.05 per item.
Idempotent/resumable stages, eval harness with targets plus holdout walk,
cost ledger per model call. Human dispatch-approval tap exists on flagged
items. No approval chains, users/roles, tag/system hierarchy, or
multi-tenancy — the core provides those.

Keep: the pipeline (ingest, transcribe, ground, verify, items, dispatch),
idempotent/resumable stages, eval harness + holdout, cost ledger,
re-walk fixed-claim verification, PWA offline posture.
Strip: demo machinery (nightly reseed, demo project/snapshot, live-app
branding), hackathon extras outside the capture loop.
Change: items post to the core over the seam instead of local-only punch
lists; auth/tenancy/project context come from the core; vocabulary and
trade routing generalize from fit-out trades to commissioning disciplines;
dispatch generalizes into trade/discipline routing against the core.

## 6. The seam (API contract sketch for to-spec)

- `POST /seam/v1/items`: capture leaf submits a verified item with
  evidence (photo with defect outline, inspector quote, video timestamp,
  severity, suggested discipline, cost/ledger refs). Idempotency key
  required; duplicates return the existing PLI.
- Core links the item to a tree node (suggested by capture, confirmed by
  rule or user), assigns an owner, and returns PLI id + state.
- `POST /seam/v1/items/{id}/verification`: re-walk results close the loop
  (fixed + verified, still open, needs a look with before/after photos).
- Auth: service credentials issued by the core; per-project scoping.
- Failure posture: leaf queues offline and retries; core never partially
  applies a submission.

## 7. Non-functional requirements

- Offline field use: capture queues walks/items without connectivity and
  syncs later; field execution views work from a synced offline queue.
- Bulk scale: tree load, template/matrix CSV import, and checksheet
  generation must handle project-scale volumes without per-row UI work.
- Cost discipline: every model call lands in the cost ledger; per-item
  capture cost stays in the low single-digit cents.
- Auditability: AI ledger append-only; state transitions carry actor +
  timestamp; handover signatures immutable once complete.

## 8. Out of scope

- Punchwalk roadmap items: live glasses streaming, Procore sync,
  spec citations.
- Physical robots.
- WinPCS beyond the domain checklist (hierarchy, DAC/MCC/RFU, outstanding
  work, security levels).
- Salesforce export/cutover tooling details (wizard covers manually).

## 9. Build order — slice graph

1. Factory floor (none) — monorepo scaffold, CI green, tracker wired.
2. Strip the fork (1) — punchwalk local minus demo weight, tests green.
3. Project tree + completion invariant (1) — hierarchy CRUD with rollup.
4. Capture-to-core seam (2,3) — items post as PLIs onto nodes.
5. Templates engine (3) — ITIs, templates, matrix, generation.
6. Field execution (4,5) — complete checks, raise PLIs, offline queue.
7. Approvals + handover docs (6) — chains, signatures, DAC/MCC/RFU.
8. Recurrence (3) — preservations/maintenance schedules.
9. Dispatch (4) — trade routing generalized.
10. Live reporting (6) — outstanding views, manhours, scorecard.
11. Access + audit (3) — roles, view/edit split, AI ledger core-wide.
12. Migration (3,5) — legacy import, expand-contract where wide.
13. Re-walk closure generalized (4,6) — fixed-claim verification core-wide.

Long pole: 1 > 3 > 5 > 6 > 7. Run 2+3 parallel first. First shippable
slice defaults to tree-plus-capture-joint.

## 10. Factory input notes

- to-spec runs the seams check against section 6 plus the leaf/core
  boundary in section 5.
- to-tickets runs the granularity/edges quiz against the slice graph
  (quiz still owed).
- implement-spec: integration branch + frontier + per-ticket worktrees,
  TDD at seams, merger subagents, final code-review.
- Wizard covers: API keys, infra, Salesforce export/cutover.

## Appendix A. Sources

- `~/Downloads/Product.zip`: Design/ (Product Design, Child Nodes, Roles,
  Import Spreadsheets 1, Check Sheets, Data Model, Reports, Workorders,
  Flange Management, Maintenance module, ...), Zenator Stuff/ (Glossary,
  Instrument ITRs and Handover), Project Management/ (Setting up the CMS
  vfb1.docx, ERD.xlsx). Re-extract with
  `unzip -o ~/Downloads/Product.zip -x "*.mov" -d /tmp/product-scan`.
- `~/Downloads/WinPCS Present.ppt`: 2007 Complan deck, domain checklist.
- Punchwalk reference clone: `/tmp/punchwalk-ref` (delete and re-clone
  when stale).

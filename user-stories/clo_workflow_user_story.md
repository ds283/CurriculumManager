# User Story — Introducing a New Course-Level Outcome (CLO)

This document traces the full lifecycle of a new course-level outcome from
coordinator initiation through to TLC committee submission. It is written as
a narrative user story to support future design work and to surface
dependencies between workflow stages. Approval and publication are out of
scope.

The story uses a concrete example: a course coordinator introducing a new
CLO requiring AI literacy outcomes across modules in an engineering
programme.

---

## Actors

| Actor                                 | Role                                                                                                           |
|---------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Course coordinator                    | Initiates the CLO workflow; scopes and oversees downstream work; reviews convenor submissions before committee |
| Module convenor                       | Responds to initiation tasks; drafts new or revised MLOs and assessment changes                                |
| TLC (Teaching and Learning Committee) | Reviews and approves or rejects the full submission bundle                                                     |
| System                                | Generates tasks, milestones, LLM analyses, and scheduled deadlines automatically                               |

---

## Stage 1 — Initiation

**Module convenor — standalone unscheduled.**
A module convenor may initiate a `ModuleOutcomeContent` or assessment change
workflow on their own initiative, outside any orchestrated publication cycle.
No `ScheduledMilestone` is created at this point. The workflow runs as a
`WorkflowInstance` without scheduling metadata until the course coordinator
discovers it via `MilestoneOverlap` detection and decides whether to adopt it
into a `PublicationTarget`.

**Course coordinator — two valid entry points.**

**Option A — Scheduled, CLO as terminal target.** The coordinator creates a
`PublicationTarget` with `target_document_type = course_outcome` and a target
date. The backwards scheduler immediately generates a `ScheduledMilestone` for
the CLO document (initially with `document = NULL`) and computes an initiation
deadline working backwards from the target date through known committee meeting
constraints. Where a firm target date is not yet available, the coordinator may
create a `PublicationTarget` with `status = PROVISIONAL`; deadlines are
computed and flagged as provisional until the target date is confirmed.

**Option B — CLO as dependency of a course specification target.** A
`PublicationTarget` already exists for a forthcoming course specification
publication. The dependency graph declares that CLOs must be approved before
the course spec can proceed, so the scheduler automatically generates a
`ScheduledMilestone` for the new CLO under the existing target.

Standalone unscheduled initiation is not available to course coordinators for
`CourseOutcomeContent` workflows. All coordinator-initiated CLO changes operate
under a `PublicationTarget` from the point of initiation, ensuring dependency
tracking and the Stage 8 bundle gate are in place throughout.

In all coordinator-initiated cases, the coordinator authors the CLO text,
enters a `change_rationale` on the `Document` envelope explaining the
governance case, and submits the document to move it forward in its own
workflow.

---

## Stage 2 — Impact Analysis and Scope Confirmation

Once the CLO document is approved at the course coordinator level and before
downstream work is initiated, the system fires the LLM impact analysis.

The LLM reads all current module specifications and approved MLOs for the
course and produces a per-module coverage classification:

- **Full coverage** — one or more existing MLOs already substantially
  address the new CLO
- **Partial coverage** — existing MLOs partially address the CLO but a gap
  remains
- **No coverage** — no existing MLO addresses the new CLO; a new outcome is
  required

The coordinator reviews this structured table — one row per module — in the
UI. The LLM output is a starting point, not a decision. The coordinator can:

- Accept a row (module enters scope with LLM-recommended classification)
- Override a row to add a module the LLM did not flag, with a note
- Override a row to exclude a module the LLM flagged, with a note

All scoping decisions are recorded with provenance (LLM recommendation,
coordinator addition, or coordinator exclusion) for audit purposes.

Only after the coordinator confirms the scope does the backwards scheduler
generate `ScheduledMilestone` records for the in-scope modules and compute
initiation deadlines for each.

---

## Stage 3 — Task Decoration and Dispatch

Before initiation tasks are pushed to convenors, the coordinator has the
opportunity to decorate each task with module-specific guidance. For each
in-scope module the coordinator can review the LLM's per-module analysis —
which existing MLOs partially address the CLO, what the gap is, and where
a draft candidate MLO was suggested — and add or edit a `context_note` on
the `Task` record.

This note is the coordinator's direct communication to the convenor:
explaining the intent behind the CLO, noting relevant constraints (e.g.
"framing must be at Level 7 given the module's position in the programme
structure"), and providing any domain-specific guidance the LLM could not
infer.

Once the coordinator confirms, initiation tasks are pushed to the relevant
convenor pools. Each task carries:

- The CLO text and reference code
- The coordinator's `context_note`
- The LLM's coverage assessment for that module
- Any draft MLO text generated by the LLM (clearly labelled as AI-generated)
- The computed deadline from the `ScheduledMilestone`

---

## Stage 4 — Convenor Drafting (MLO)

The module convenor receives the task and opens a pre-populated
`ModuleOutcomeContent` workflow form. The LLM has pre-filled the outcome
text field based on the CLO, FHEQ level descriptors, and the module's
existing content. The convenor reviews each field — pre-filled content
submitted unchanged is flagged for additional scrutiny at review.

The convenor authors or revises the proposed MLO and submits the document.

---

## Stage 5 — Assessment Workflow Initiation

When the `ModuleOutcomeContent` document reaches `SUBMIT` state, the system
automatically initiates an assessment realignment workflow for the module.
This fires at submission rather than approval so that assessment revision
can proceed concurrently with MLO review, rather than being serialised behind
it.

The newly spawned assessment workflow is automatically attached to the
originating `PublicationTarget` via `MilestonePublicationTarget`. Its
deadline is computed by the backwards scheduler relative to the same target
date, and its at-risk status feeds into the same critical path view as all
other milestones under the target.

The coordinator can see the new milestone appear on their publication target
dashboard without any manual action.

---

## Stage 6 — Coordinator Review

When a downstream document (MLO or assessment) enters `COORDINATOR_REVIEW`
state, the system fires the LLM brief adequacy check asynchronously. By the
time the coordinator opens the review task, the check result is ready.

The check assesses whether the proposed content meets the brief: does the
MLO address the CLO it was spawned to support, is it pitched at the correct
FHEQ level, does it reflect the coordinator's guidance note, and is it
consistent with the stated change rationale?

The coordinator reviews the document alongside the LLM adequacy verdict. If
satisfied, they advance the document to `TLC_REVIEW`. If not, they push it
back to the convenor with a rejection reason recorded in
`WorkflowEvent.metadata`, which generates a new task in the convenor's pool.

This loop repeats until the coordinator is satisfied. Each iteration is fully
recorded in the `WorkflowEvent` audit trail.

---

## Stage 7 — Dashboard Visibility Throughout

At all times during stages 3–6, the coordinator's publication target
dashboard shows the complete picture of all in-flight workflows under the
originating `PublicationTarget`:

- Which `ScheduledMilestone` records exist (one per document per module)
- The current FSM state of each
- Which are on the critical path
- Which are at risk given current trajectory
- When any returned workflow spawns a new downstream workflow (e.g. a
  revised MLO triggers a new assessment workflow), the new milestone
  appears automatically on the dashboard

The backwards scheduler recomputes deadlines and risk flags whenever a
document is returned or a new milestone is added, ensuring the coordinator
always sees an accurate picture of schedule risk.

---

## Stage 8 — Bundle Advance to TLC

The coordinator cannot advance the CLO document to `TLC_REVIEW` until all
dependent workflows under the `PublicationTarget` have also reached
`TLC_REVIEW`. This is a hard FSM pre-transition guard, not an advisory
check.

Once all documents in the bundle are at `TLC_REVIEW`, the coordinator
advances the CLO, completing the bundle. The system attaches the bundle to
a `CommitteeMeeting` as an `AgendaItem`.

Before the meeting, the LLM committee screening check fires across all
submitted documents simultaneously, surfacing cross-document conflicts and
inconsistencies that no per-document check could catch.

---

## Stage 9 — TLC Review and Rejection Loop

The committee reviews the full bundle. Two outcomes are possible:

**Approval** — out of scope for this document.

**Rejection** — the committee records its required changes in the rejection
outcome (stored in `WorkflowEvent.metadata` on the TLC rejection transition).
The workflow returns to the coordinator.

On rejection, the backwards scheduler immediately recomputes deadlines across
all milestones under the `PublicationTarget`, since the iteration has consumed
calendar time. `at_risk` flags may change materially; the coordinator sees
the updated schedule on their dashboard when they receive the bundle back.

The coordinator reviews the committee's feedback and selectively returns
individual subworkflows to the relevant convenors. Each returned workflow
generates a new task in the convenor's pool, pre-decorated with the
committee's feedback.

As convenors rework their documents, any newly-spawned workflows (e.g. a
revised MLO triggering a new assessment change) are automatically captured
as `ScheduledMilestone` instances under the same `PublicationTarget` and
appear on the coordinator's dashboard.

When all revised documents have worked back through coordinator review to
`TLC_REVIEW`, the coordinator re-advances the CLO and the bundle returns to
committee for a further round.

---

## Key Constraints Summary

The following constraints are load-bearing for this workflow. Future design
decisions that touch these areas should explicitly consider their impact.

| Constraint                                                             | Rationale                                                                   |
|------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| Scope confirmation gates the scheduler                                 | Prevents milestones being generated for modules that don't need work        |
| Coordinator can override LLM scope in both directions                  | LLM is advisory; coordinator has domain knowledge the model cannot infer    |
| Assessment workflows fire at MLO `SUBMIT`, not `APPROVE`               | Enables concurrent processing; reduces end-to-end time                      |
| All downstream workflows attach to the originating `PublicationTarget` | Coordinator has a single dashboard view of the full programme of work       |
| No partial approval at committee                                       | Preserves context for rejected items on reconsideration                     |
| Hard FSM gate before advancing CLO to `TLC_REVIEW`                     | Ensures committee always sees a complete, coherent bundle                   |
| Scheduler recomputes on TLC rejection                                  | Coordinator sees accurate risk picture immediately on receiving bundle back |
| New milestones auto-attached in revision iterations                    | Coordinator dashboard remains complete without manual intervention          |

---

*Document status: draft — captured from CLO workflow walkthrough session.*
*Approval and publication stages are out of scope and not covered here.*

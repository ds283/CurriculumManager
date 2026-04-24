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
- The deadline inherited from the originating `PublicationTarget`

Each task also carries a FK to the originating `PublicationTarget`. At this
point no `ScheduledMilestone` exists for the module — the task represents
an unresolved obligation. The coordinator's publication target dashboard
surfaces open tasks without child milestones as a distinct risk category,
separate from milestones that are in-flight or at risk.

---

## Stage 4 — Convenor Response (MLO)

The module convenor receives the task and reviews the CLO text, the LLM
coverage analysis, the coordinator's guidance note, and any draft MLO text
provided. The convenor then decides how to respond:

- **Modify an existing `ModuleOutcomeContent`** — the convenor opens an
  existing approved MLO into a new revision workflow
- **Create a new `ModuleOutcomeContent`** — the convenor initiates a new
  outcome document
- **Both** — for example, creating a new MLO while simultaneously
  initiating a retirement workflow for an outcome that is being superseded

In all cases, the convenor nominates the document or documents they are
working on. Each nomination creates a `ScheduledMilestone` with the
`document` FK populated immediately, linked back to the originating `Task`
via `ScheduledMilestone.originating_task`. The milestone inherits its
`PublicationTarget` linkage from the task, and the backwards scheduler
computes `must_be_complete_by` and `must_be_initiated_by` at this point.

The coordinator's dashboard updates automatically when the first nomination
is made: the open task transitions from unresolved-obligation status to
in-progress, and the new concrete milestone or milestones appear under the
`PublicationTarget` view.

The LLM has pre-filled the outcome text field for any new `ModuleOutcomeContent`
document based on the CLO, FHEQ level descriptors, and the module's existing
content. The convenor reviews each field — pre-filled content submitted
unchanged is flagged for additional scrutiny at coordinator review.

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

## Stage 9 — TLC Review, Approval, and Rejection Loop

The committee reviews the full bundle presented as an `AgendaItem` on the
`CommitteeMeeting`. Two outcomes are possible.

---

**Approval**

The committee approves the bundle as a unit. No partial approval is permitted
(D10). The approval transition advances the CLO document and all dependent
documents — MLOs, assessment changes, and any other documents in the bundle —
from `TLC_REVIEW` to `APPROVED`. This is recorded as a `WorkflowEvent` on each
`WorkflowInstance`, with `actor_role = chair_of_tlc` (or whichever committee
role performs the approval transition).

The bundle does not publish immediately on approval. It moves to a holding
state pending the setting of an effective date and scheduled publication (see
Stage 10).

---

**Rejection**

The committee records its required changes in `WorkflowEvent.metadata` on the
bundle-level CLO rejection transition (`TLC_REVIEW → COORDINATOR_REVIEW`). The
entire bundle is returned to the coordinator as a unit; individual documents are
not returned directly to convenors by the committee.

On rejection, the backwards scheduler immediately recomputes deadlines across
all milestones under the `PublicationTarget`. `at_risk` flags may change
materially since the iteration has consumed calendar time. The coordinator sees
the updated schedule on their dashboard when they receive the bundle back.

The coordinator reviews the committee's feedback and selectively returns
individual sub-workflows to the relevant convenors. For each sub-workflow being
returned, the coordinator performs an explicit `TLC_REVIEW → CONVENOR_DRAFT`
back-transition and authors a `Task.context_note` on the newly generated
convenor task. This note is the coordinator's own communication: it may
incorporate, reframe, or expand on the committee feedback. Committee feedback is
not forwarded verbatim. The immutable committee record remains in
`WorkflowEvent.metadata` on the bundle rejection transition.

As convenors rework their documents, any newly spawned workflows — for example
a revised MLO triggering a new assessment realignment — are automatically
captured as `ScheduledMilestone` instances under the same `PublicationTarget`
and appear on the coordinator's dashboard without manual intervention (D11).

The revision cycle proceeds through coordinator review (Stage 6) and back to
`TLC_REVIEW` for each affected document. When all revised documents have
returned to `TLC_REVIEW`, the coordinator re-advances the CLO through the Stage
8 bundle gate, and the bundle returns to committee for a further round. There is
no limit on the number of rejection iterations; each round is fully captured in
the `WorkflowEvent` audit trail.

---

## Stage 10 — Setting the Effective Date

Once the bundle has been approved by TLC (all documents at `APPROVED`), a user
holding the `publication_manager` role sets `PublicationTarget.effective_from`
— the date and time from which the published curriculum comes into force. This
is typically aligned to an academic year boundary (e.g. 1 August for the
following intake year).

Setting `effective_from` is an audited action. It produces a `WorkflowEvent`
record on the CLO `WorkflowInstance` with `actor_role = publication_manager`,
capturing who set the date and when. It is not a silent field update.

Individuals eligible to hold the `publication_manager` role include the chair
of TLC for the relevant department, course coordinators, and designated faculty
administration staff. The `publication_manager` role is granted independently
of any other role the individual holds. The publication transition checks solely
for this role.

On saving `effective_from`, the system schedules a Celery publication task
(using `django-celery-beat`, backed by the application database) to fire at
the specified time. The Celery task ID is stored in
`PublicationTarget.scheduled_task_id`.

**Revising the effective date** — if the effective date needs to change before
it fires, the `publication_manager` revises `effective_from`. The system revokes
the previously scheduled Celery task (using the stored `scheduled_task_id`) and
schedules a replacement. The revision is recorded as a further `WorkflowEvent`.

---

## Stage 11 — Publication

At the scheduled time, the Celery publication task fires and executes the
following steps in a single atomic database transaction:

1. All documents in the bundle transition from `APPROVED` to `PUBLISHED`.
2. An `AssetVersion` is appended for each document, with `valid_from` set to
   `PublicationTarget.effective_from` and `status = published`.
3. The `valid_to` window on any prior `AssetVersion` for the same `Asset` is
   closed by setting `valid_to = effective_from`, making it clear that the
   prior version ceased to be in force at the exact moment the new one took
   effect.
4. `on_document_published(document)` fires for each document, updating the
   denormalised `last_published_at` and `last_published_document_type` fields
   on `ModuleIdentifier` and the relevant `CourseIdentifier`.
5. `has_in_flight_workflow` is cleared on `ModuleIdentifier` and
   `CourseIdentifier` for all modules whose workflows are now closed (D19
   retention — `WorkflowInstance` records are archived, not deleted).

No intermediate state is observable in which a subset of bundle documents are
published and others are not.

Before executing the transaction, the task checks that
`PublicationTarget.effective_from` still matches the timestamp it was scheduled
with. If they differ — indicating the effective date was revised after the task
was enqueued and revocation did not reach the worker in time — the task aborts
without making any changes and logs the discrepancy. No user intervention is
needed; the replacement task scheduled by the revision will fire at the correct
time.

On successful publication, `PublicationTarget.scheduled_task_id` is cleared.
The `PublicationTarget`, `ScheduledMilestone`, and all associated records are
retained permanently as the historical record of what was planned and delivered,
supporting transition duration estimation (Phase 6) and regulatory audit
responses.

---

## Key Constraints Summary (updated)

The following constraints are load-bearing for this workflow. Future design
decisions that touch these areas should explicitly consider their impact.

| Constraint                                                             | Rationale                                                                        |
|------------------------------------------------------------------------|----------------------------------------------------------------------------------|
| Scope confirmation gates the scheduler                                 | Prevents milestones being generated for modules that don't need work             |
| Coordinator can override LLM scope in both directions                  | LLM is advisory; coordinator has domain knowledge the model cannot infer         |
| Assessment workflows fire at MLO `SUBMIT`, not `APPROVE`               | Enables concurrent processing; reduces end-to-end time                           |
| All downstream workflows attach to the originating `PublicationTarget` | Coordinator has a single dashboard view of the full programme of work            |
| No partial approval at committee                                       | Preserves context for rejected items on reconsideration                          |
| Hard FSM gate before advancing CLO to `TLC_REVIEW`                     | Ensures committee always sees a complete, coherent bundle                        |
| Scheduler recomputes on TLC rejection                                  | Coordinator sees accurate risk picture immediately on receiving bundle back      |
| New milestones auto-attached in revision iterations                    | Coordinator dashboard remains complete without manual intervention               |
| `APPROVED` and `PUBLISHED` are distinct FSM states                     | Separates governance sign-off from operational publication; different role gates |
| Bundle publication is a single atomic transaction                      | No intermediate state where the published curriculum is internally inconsistent  |
| Setting `effective_from` is an audited action                          | Provides a full audit trail for the date on which the curriculum came into force |
| Publication gated on `publication_manager` role only                   | Separates publication authority from approval authority                          |
| Scheduler data retained permanently                                    | Supports duration estimation, capacity planning, and regulatory audit            |

---

*Document status: draft — captured from CLO workflow walkthrough session.*
*Approval and publication stages are out of scope and not covered here.*

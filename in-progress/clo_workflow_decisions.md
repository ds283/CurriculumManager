# CLO Workflow — Design Decisions and Open Questions

Captured during walkthrough of the course-level outcome (CLO) introduction
workflow. Covers the full lifecycle from coordinator initiation through to
TLC committee submission. Approval and publication are out of scope for this
document.

---

## Design Decisions

These are resolved positions. Future design work should not contradict them
without explicitly reconsidering the rationale recorded here.

---

### D1 — Entry points for CLO workflow initiation

**Module convenor (unscheduled, standalone):**
A module convenor may initiate a `CourseOutcomeContent`-adjacent workflow —
specifically a `ModuleOutcomeContent` or assessment change — on their own
initiative, outside any orchestrated publication cycle. The system creates the
relevant `Document` in `DRAFT` state and opens a `WorkflowInstance`. No
`ScheduledMilestone` is involved at initiation.

These orphan workflows are surfaced to the course coordinator via
`MilestoneOverlap` detection. The coordinator may adopt them into a
`PublicationTarget` by creating a `MilestonePublicationTarget` link, at which
point the backwards scheduler generates a `ScheduledMilestone` retroactively.
No scheduling metadata is required before that adoption decision.

**Course coordinator — two valid entry points only:**

1. **CLO as terminal target (Path 2)** — the coordinator creates a
   `PublicationTarget` with `target_document_type = course_outcome` and a
   target date. The backwards scheduler generates a `ScheduledMilestone` with
   `document = NULL` until the workflow is initiated, then populates the FK.
   If a firm target date is not yet known, a `PublicationTarget` with
   `status = PROVISIONAL` may be created; deadlines are computed but flagged
   as provisional and revised when the target date is confirmed.

2. **CLO as dependency of a course specification target (Path 3)** — a
   `PublicationTarget` already exists for a forthcoming course specification
   publication. The dependency graph declares that CLOs must be approved before
   the course spec can proceed, so the scheduler automatically generates a
   `ScheduledMilestone` for the CLO under the existing target.

Standalone unscheduled initiation is **not available to course coordinators**
for `CourseOutcomeContent` workflows. A coordinator-initiated CLO change always
operates under a `PublicationTarget`, ensuring dependency tracking, backwards
scheduling, and the Stage 8 bundle gate are in place from the point of
initiation.

**Rationale for restricting standalone to convenors:** the Stage 8 FSM
pre-condition (all dependent workflows must reach `TLC_REVIEW` before the CLO
can advance) is defined relative to a `PublicationTarget`. A CLO workflow
without a `PublicationTarget` has no bundle boundary, no enforced dependency
tracking, and no mechanism to bind downstream MLO and assessment milestones.
Exploratory coordinator drafting before a target date is known is accommodated
by the `PROVISIONAL` publication target status rather than a scheduling-free
path that bypasses the dependency graph.

Path 3 is the most common governance scenario. Path 2 applies when a CLO is
being introduced outside a full periodic review cycle.

---

### D2 — `change_rationale` lives on the `Document` envelope

The formal governance justification for a change is a universal concern
across document types — not specific to CLOs. It is stored as a nullable
`TextField` on the `Document` envelope, populated at workflow initiation,
and visible to reviewers across all document types.

It is nullable because not all document workflows are change-driven (e.g.
initial CLO introduction at programme inception).

---

### D3 — Coordinator-to-convenor guidance travels via `Task.context_note`

The coordinator's module-specific guidance note — surfaced in the mockup
as "Coordinator note" — is not stored on the `CourseOutcomeContent` document.
It belongs to the `Task` record pushed to the convenor, since it is addressed
to a specific actor about a specific piece of downstream work.

The `Task` model requires a new `context_note` nullable `TextField` to
support this. This field is populated as a side effect of the coordinator
approving the LLM impact analysis output, not authored directly on task
creation.

---

### D3a — `Task` carries a direct FK to `PublicationTarget`

A `Task` dispatched to a module convenor as part of a CLO-driven workflow
carries a direct FK to the originating `PublicationTarget`. This is
distinct from the `ScheduledMilestone → PublicationTarget` relationship and
serves a different purpose: it represents an unresolved obligation on the
target before any concrete document workflow has been initiated.

The scheduler treats open `Task` records linked to a `PublicationTarget` as
unresolved dependencies and surfaces them as risk flags in the coordinator
dashboard. This is distinct from at-risk or critical-path computation on
`ScheduledMilestone` records — a `Task` without a child milestone cannot
have `must_be_complete_by` or `must_be_initiated_by` values, so it cannot
participate in deadline arithmetic. Its role is solely to make the
dependency gap visible.

When a convenor nominates a document under a `Task` — whether by creating a
new `ModuleOutcomeContent` or opening a revision workflow on an existing one
— the resulting `WorkflowInstance` and `ScheduledMilestone` carry a FK back
to the originating `Task`. This closes the dependency gap: the scheduler
can now compute against a concrete milestone, and the coordinator dashboard
can collapse the task and its child milestones into a coherent module-level
view.

A single `Task` may spawn multiple child `ScheduledMilestone` records — for
example where a convenor creates a new MLO and simultaneously retires an
existing one, or where assessment workflows are triggered downstream. All
child milestones inherit the `PublicationTarget` linkage from the `Task`.

**Model implications:**
- `Task` requires a new nullable FK `publication_target → PublicationTarget`
- `Task` requires a new nullable FK `scheduled_milestone → ScheduledMilestone`
  for cases where the coordinator dispatches a task against a pre-existing
  milestone (e.g. Path 2 or Path 3 initiation where a `ScheduledMilestone`
  already exists with `document = NULL`)
- `WorkflowInstance` and `ScheduledMilestone` require a new nullable FK
  `originating_task → Task` to preserve the audit trail from document back
  to the task that drove it

**Rejected alternative — sentinel `ScheduledMilestone`:** a `ScheduledMilestone`
with `document = NULL` was considered as the parent object representing the
module obligation. This was rejected because such a milestone cannot carry
`must_be_complete_by` or `must_be_initiated_by` values and is therefore
opaque to the scheduler. The apparent advantage of keeping the dashboard
homogeneous does not outweigh the semantic cost of a milestone that does not
represent a scheduled document workflow. `Task` is the semantically correct
object to represent an outstanding obligation dispatched to an actor.

---

### D4 — Downstream work targets `ModuleOutcomeContent`, not `ModuleSpecificationContent`

The unit of work pushed to module convenors in response to a CLO change is
a new or revised `ModuleOutcomeContent` document, not a full
`ModuleSpecificationContent` revision.

Rationale: requesting a full module spec revision is the wrong granularity
when the change is targeted at a single outcome. It obscures the audit trail
linking the MLO to the CLO that drove it, and imposes unnecessary authoring
burden on the convenor.

A `ModuleSpecificationContent` revision is triggered later, automatically,
once all MLO and assessment workflows for the module have reached `Approved`.
See D7.

---

### D5 — Assessment workflow trigger point is `ModuleOutcomeContent` reaching `SUBMIT`

Assessment realignment workflows are initiated when `ModuleOutcomeContent`
reaches `SUBMIT`, not `APPROVE`. This allows MLO review and assessment
revision to proceed concurrently rather than serially.

Reviewers of the MLO submission can see that assessment realignment is
already in train, which is relevant committee context.

---

### D6 — Downstream workflows attach to the originating `PublicationTarget`

Assessment workflows spawned from MLO submission are attached to the
originating `PublicationTarget` via `MilestonePublicationTarget`. Their
deadlines are computed by the backwards scheduler relative to the same
target date, and their at-risk status feeds into the same critical path view.

This attachment is a FSM side effect triggered when the downstream workflow
is spawned — it does not require coordinator intervention.

---

### D7 — `ModuleSpecificationContent` revision is auto-initiated, late-stage

A module specification revision is required to incorporate approved MLO and
assessment changes into the authoritative published record. It is
auto-initiated by the system only once all MLO and assessment workflows for
that module have reached `APPROVED`. It is an incorporation and sign-off
step rather than substantive authoring, and must not be manually triggered
by the convenor.

---

### D8 — Scoping step is required before the scheduler generates milestones

The backwards scheduler must not generate `ScheduledMilestone` records for
downstream MLO workflows until the coordinator has confirmed scope at the
impact analysis review step. The sequencing is:

1. CLO approved
2. LLM impact analysis runs
3. Coordinator reviews and confirms scope (with override capability in both
   directions — see D9)
4. Scheduler generates milestones only for confirmed in-scope modules
5. Initiation tasks pushed to convenors

This sequencing must be enforced as an explicit workflow state or gate, not
left implicit.

---

### D9 — Coordinator can override LLM scope recommendations in both directions

The LLM impact analysis output is a starting point for scope confirmation,
not a gate. The coordinator must be able to:

- **Add** a module the LLM classified as fully covered or did not flag
- **Remove** a module the LLM flagged as needing changes

Both override directions must be supported in the UI and recorded with
provenance in the data model (see G2).

---

### D10 — No partial approval at committee

The committee approves or rejects the full submission bundle as a unit.
Partial approval is not permitted.

Rationale: partial approval removes context from rejected items when they
are returned for reconsideration. The committee's feedback applies to the
bundle as a coherent whole.

This constraint must be enforced in the FSM: the approve/reject transition
on the CLO workflow must be conditional on all dependent workflows in the
bundle being at `TLC_REVIEW` state. The reject transition propagates back
to the coordinator rather than to individual document authors.

---

### D11 — Automatic milestone capture for newly-spawned workflows in revision iterations

Any workflow spawned during a revision iteration — for example a new
assessment workflow triggered by a revised MLO — must be automatically
attached to the originating `PublicationTarget` as a new `ScheduledMilestone`
via `MilestonePublicationTarget`. This happens as a FSM side effect without
coordinator intervention, regardless of which iteration of the bundle the
spawning occurs in.

---

### D12 — Scheduler recomputes on TLC rejection

When a bundle is rejected by TLC and returned to the coordinator, the
backwards scheduler must recompute deadlines across all milestones for the
originating `PublicationTarget`. This recomputation is triggered
automatically by the rejection transition, not deferred to the next
dashboard view.

---

### D13 — `COORDINATOR_REVIEW` is a distinct FSM state

The coordinator's pre-committee quality review is a named FSM state
(`COORDINATOR_REVIEW`), distinct from `SUBMITTED` and `TLC_REVIEW`. This
matters for pipeline visualisation, task pool assignment, and LLM trigger
points (see D14). The coordinator and TLC members are different role pools.

---

### D14 — Brief adequacy LLM check fires on entry to `COORDINATOR_REVIEW`

The LLM brief adequacy check (workflow 8 in `llm_workflows.md`) fires when
a downstream document (e.g. `ModuleOutcomeContent`) enters
`COORDINATOR_REVIEW` state, not at submission. This ensures the coordinator
has a ready analysis when they open the task, without running the check on
documents that are pushed back before the coordinator sees them.

---

### D15 — Committee scheduling eligibility depends on `PublicationTarget` scope

A document reaching `TLC_REVIEW` state is necessary but not sufficient for
eligibility to be attached to a `CommitteeMeeting` as an `AgendaItem`. The
committee scheduling logic must apply the following eligibility rule:

A document at `TLC_REVIEW` is eligible for committee scheduling if and only
if **one** of the following conditions holds:

1. **Standalone** — the document's `WorkflowInstance` has no
   `PublicationTarget` linkage (directly or via `ScheduledMilestone`). It
   should be surfaced in the UI for inclusion in the next available
   `CommitteeMeeting` for the relevant committee. Standalone documents are
   prioritised for prompt scheduling so that they move efficiently through
   the workflow without being held by orchestration concerns that do not
   apply to them.

2. **Orchestrated bundle complete** — the document's `WorkflowInstance` is
   linked to a `PublicationTarget` (via `ScheduledMilestone` or `Task`),
   and all sibling documents under that `PublicationTarget` have also
   reached `TLC_REVIEW`. Only at this point is the bundle as a whole
   eligible for committee scheduling. The parent CLO document's
   advance-to-`TLC_REVIEW` transition (the Stage 8 bundle gate, D10) is
   the moment this condition becomes satisfiable.

Documents in the orchestrated path that have reached `TLC_REVIEW` but whose
siblings have not are ineligible for committee scheduling. They must not be
attached to a `CommitteeMeeting` until the bundle is complete.

**UI implications:** the coordinator dashboard must distinguish two distinct
sub-states for documents at `TLC_REVIEW`:

- **Eligible — awaiting scheduling** — standalone documents, or orchestrated
  bundles where all siblings are also at `TLC_REVIEW`. Surfaced as
  actionable: the coordinator can attach to an upcoming `CommitteeMeeting`.
- **Held — awaiting siblings** — orchestrated documents at `TLC_REVIEW`
  where one or more siblings have not yet reached `TLC_REVIEW`. Displayed
  with a summary of what remains outstanding, so the coordinator can see
  exactly what is blocking committee scheduling.

These are UI presentation states derived from the eligibility rule above,
not additional FSM states. The FSM state remains `TLC_REVIEW` in both cases.

**Implementation note:** the eligibility query runs against
`PublicationTarget` scope at scheduling time, not at the point of FSM
transition. Nothing in the FSM itself enforces or records eligibility —
it is computed on demand by the scheduling UI and the committee attachment
logic.

**Depends on:** D10 (bundle gate), G4 (`AgendaItem` not yet modelled),
G5 (bundle object not yet modelled).

---

### D16 — FSM states distinguish `APPROVED` from `PUBLISHED`

All document workflows in the bundle carry a distinct `APPROVED` state and a
distinct `PUBLISHED` state. Committee approval and publication are separate FSM
transitions with separate `WorkflowEvent` audit records. A document at
`APPROVED` has passed governance; a document at `PUBLISHED` has had an
`AssetVersion` committed with a `valid_from` date set to
`PublicationTarget.effective_from`.

This separation allows the approval and publication steps to be performed by
different actors under different role permissions, and ensures the audit trail
distinguishes governance sign-off from the operational act of publication.

---

### D17 — Bundle publication is a single atomic Celery task

Publication of a bundle is executed by a Celery task on the `scheduling` queue.
The task runs a single `transaction.atomic()` block that:

1. Transitions all documents in the bundle from `APPROVED` to `PUBLISHED`
2. Appends an `AssetVersion` for each document with `valid_from` set to
   `PublicationTarget.effective_from` and `status = published`
3. Closes the `valid_to` window on any prior `AssetVersion` for the same
   `Asset`, setting `valid_to = effective_from`
4. Calls `on_document_published(document)` for each document, updating the
   denormalised fields on `ModuleIdentifier` and `CourseIdentifier`
5. Records a `WorkflowVersionEvent` for each version action

No intermediate state is observable in which a subset of bundle documents are
published and others are not. Staged publication is not supported for this
workflow.

**Rationale:** the bundle is approved as a coherent unit (D10). Publishing it
piecemeal would create a window in which the published curriculum is internally
inconsistent. Atomicity eliminates that risk.

---

### D18 — `PublicationTarget` carries `effective_from` and `scheduled_task_id`

Two new fields are added to `PublicationTarget`:

- `effective_from` — `DateTimeField (nullable)`. The date and time from which
  the published documents come into force. Set by a `publication_manager` after
  all final approvals are confirmed. This value is used as `valid_from` on all
  `AssetVersion` records created by the publication transaction.

- `scheduled_task_id` — `CharField (nullable, max_length=255)`. The Celery task
  ID of the pending publication task. Stored so the task can be revoked if
  `effective_from` is revised before it fires.

Setting or revising `effective_from` is an audited action. It must produce a
`WorkflowEvent` record on the bundle's CLO `WorkflowInstance`, with
`actor_role = publication_manager`. It is not a silent field update.

When `effective_from` is set, the system schedules the Celery publication task
using `apply_async(eta=effective_from)` and stores the returned task ID in
`scheduled_task_id`. Both the field write and the task scheduling must occur
within the same `transaction.atomic()` block using Celery's
`task_always_eager=False` and the `on_commit` hook, so that the task is only
enqueued if the database transaction commits successfully.

---

### D18a — Publication task validates `effective_from` on entry

The Celery publication task checks on entry that
`PublicationTarget.effective_from` matches the timestamp it was scheduled with.
If they differ — indicating the effective date was revised after the task was
enqueued — the task aborts without making any changes and logs the discrepancy.
A new task will have been scheduled by the revision action that changed
`effective_from`.

This guard protects against the edge case where Celery's `revoke()` did not
reach the worker before the task was picked up (revocation is best-effort when
a task is already in flight).

---

### D19 — Scheduler data and workflow records are retained on completion

`WorkflowInstance` records are archived on completion by setting an `archived`
boolean flag. Archived instances are excluded from active task queue queries and
coordinator dashboards but remain queryable for audit and reporting purposes.

`PublicationTarget`, `ScheduledMilestone`, `WorkflowEvent`, `AssetVersion`, and
`WorkflowVersionEvent` records are retained permanently. They form the
historical record used for transition duration estimation (Phase 6), PSRB
submission evidence, and responses to regulatory audit requests. They must never
be deleted by application logic.

---

### D20 — Post-TLC-rejection back-transitions are explicit FSM transitions

Downstream document workflows (`ModuleOutcomeContent`, `AssessmentContent`, and
any other type in the bundle) carry an explicit `TLC_REVIEW → CONVENOR_DRAFT`
back-transition in their FSM definitions. This transition is not implicit or
inferred.

The committee's required changes are recorded in `WorkflowEvent.metadata` on
the bundle-level CLO rejection transition (`TLC_REVIEW → COORDINATOR_REVIEW`).
When the coordinator returns an individual sub-workflow to a convenor, the
coordinator authors a `Task.context_note` on the newly generated convenor task.
This note is the coordinator's own communication: it may incorporate, reframe,
or expand on the committee feedback at the coordinator's discretion. Committee
feedback is not forwarded verbatim.

The committee record in `WorkflowEvent` is immutable. The coordinator's note is
their own authored communication and is distinct from it.

---

### D21 — `publication_manager` is a dedicated role in `TenantMembership`

Publication of an approved bundle is gated solely on the `publication_manager`
role. This is a new value in the `TenantMembership` role `TextChoices` and is
granted independently of all other roles.

Individuals holding `publication_manager` will typically also hold
`chair_of_tlc`, `course_coordinator`, or a faculty administration role, but
those are separate `TenantMembership` records. The publication FSM transition
checks solely for `publication_manager` and does not inspect the actor's other
roles.

When a publication transition is recorded in `WorkflowEvent`, `actor_role` must
be set to `publication_manager` explicitly by the FSM transition handler. The
handler must not infer the acting role from the user's full role set.

**Rationale for a dedicated role rather than object-level permissions:** the
system operates within a single faculty tenant. The granularity of
`django-guardian` object-level grants — restricting a specific user to a
specific `PublicationTarget` — is not required at this scope. A role-based
check is consistent with all other FSM permission gates in the system and keeps
the permission model in a single location.

---

### D22 — Celery Beat scheduler backed by the application database

The Celery Beat scheduler uses `django-celery-beat`, persisting scheduled task
entries in the application's PostgreSQL database rather than in Redis or the
default file-based Beat store.

**Rationale:** setting `effective_from` and enqueuing the corresponding Celery
task must be atomic — if the database write commits but the task is not
persisted, the publication will silently never fire. Using the same PostgreSQL
database for both the `PublicationTarget` record and the scheduled task entry
allows both writes to occur within a single `transaction.atomic()` block via
Django's `on_commit` hook. Redis with AOF persistence does not provide this
transactional guarantee.

The `scheduled_task_id` stored on `PublicationTarget` references the
`django-celery-beat` periodic task entry, allowing it to be deleted (revoked)
if `effective_from` is revised before the task fires.

---

## Open Questions

These are unresolved design questions that must be answered before the
affected components can be fully specified.

---

### G2 — Coordinator scoping decision provenance not modelled

There is no model currently capturing the per-module scoping decision made
at the impact analysis review step. The audit trail must record not just
which modules are in scope but why — LLM recommendation accepted, coordinator
override added, LLM recommendation rejected.

Candidate location: additional fields on `ScheduledMilestone`:

- `scope_origin` — `TextChoices`: `llm_recommended`, `coordinator_added`,
  `llm_recommended_excluded`
- `scope_note` — nullable `TextField`

Alternatively these fields could live on the `LLMResult` row if that model
is restructured per G3.

**Prerequisite for:** D8, D9.

---

### G3 — `LLMResult` structure insufficient for impact analysis

`LLMResult` is currently a generic advisory output type. The impact analysis
use case requires a structured, row-level format — one record per module —
where each row carries the LLM's coverage classification, explanatory text,
and a coordinator override state (accepted, rejected, overridden).

A generic `LLMResult` with a blob output JSON is likely insufficient. The
impact analysis may warrant its own result sub-type or structured sub-model.

**Prerequisite for:** D8, D9, G2.

---

### G4 — `AgendaItem` model not specified

The `CommitteeMeeting` model exists as a scheduling input but there is no
specified model for attaching documents to a meeting agenda. An `AgendaItem`
model is needed that links a `Document` (or `WorkflowInstance`) to a
`CommitteeMeeting`, carries agenda order, and records the meeting outcome
against that item.

**Prerequisite for:** G5, committee stage implementation.

---

### G5 — Explicit submission bundle object not modelled

The submission bundle presented to TLC is currently implicit — it is the
set of `ScheduledMilestone` instances under a `PublicationTarget` that have
reached `TLC_REVIEW`. This may be insufficient.

The committee rejection record, `AgendaItem` (G4), and the coordinator's
return workflow all need an explicit object to reference. A bundle model
would also give submissions a stable identity across multiple TLC cycles,
making it possible to query the full revision history of a submission.

**Depends on:** G4.
**Prerequisite for:** D10, D12, committee stage implementation.

---

### G6 — Structured vs narrative content in `ModuleSpecificationContent`

`ModuleSpecificationContent` bundles two separable concerns:

- **Structured fields** — credit points, level, delivery mode, period,
  component references. Largely derived from approved component documents.
- **Freeform narrative** — module description and outline, requiring genuine
  editorial judgement and potentially impacted by MLO and assessment changes
  independently of any structured field change.

These may warrant independent amendment pathways. A convenor should be able
to revise narrative text without initiating a full module spec workflow. The
auto-triggered incorporation revision (D7) should not conflate structured
field updates with narrative authoring.

**Requires:** input from curriculum team on what committees need to see and
approve at each stage.

---

*Document status: draft — captured from CLO workflow walkthrough session.*
*Approval and publication stages are out of scope and not covered here.*

---

### G7 — `PublicationTarget` provisional status not yet modelled

D1 introduces the concept of a `PublicationTarget` with `status = PROVISIONAL`
to accommodate coordinator-initiated CLO workflows where a firm target date is
not yet known. This status is not currently modelled on `PublicationTarget`.

The following questions need resolving before it can be implemented:

- Does `status` warrant a `TextChoices` field on `PublicationTarget`, or is
  provisional state better represented as a nullable `confirmed_date` alongside
  a required `provisional_date`, with confirmation being an explicit transition?
- Should the backwards scheduler compute deadlines against a provisional target
  date, and if so, how are provisional deadlines distinguished from confirmed
  ones in the UI and in `ScheduledMilestone` records?
- Is there a workflow action (coordinator confirms the target date) that
  transitions the `PublicationTarget` out of provisional state, or is it a
  simple field edit?

**Prerequisite for:** D1 (Path 2 provisional variant).

---

### G8 — `Task` model fields for `PublicationTarget` linkage not yet specified

D3a introduces two new FKs on `Task` and one on `WorkflowInstance` and
`ScheduledMilestone`. The `Task` model is not yet fully specified in
`data_models.md`. The following need resolving before implementation:

- What is the full field specification for `Task`, including status
  `TextChoices` sufficient to distinguish: dispatched-awaiting-response,
  in-progress (convenor has nominated at least one document), complete
  (all child milestones resolved), and withdrawn?
- The `scheduled_milestone` FK on `Task` applies when a task is dispatched
  against an already-existing `ScheduledMilestone` (Paths 2 and 3). In the
  task-first path (no pre-existing milestone), this FK is NULL at dispatch
  and remains NULL — the milestone is created by the convenor's action.
  This dual-mode behaviour needs to be clearly documented and enforced in
  `clean()`.
- When a convenor nominates multiple documents under a single `Task`, the
  coordinator dashboard must present these as a coherent module-level group.
  The grouping query relies on `ScheduledMilestone.originating_task`. The
  display logic for this view needs specifying in `UI_SPEC.md`.

**Prerequisite for:** D3a, Stage 3 and Stage 4 implementation.
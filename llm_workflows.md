# LLM-Enhanced Workflows

This document describes each workflow where a locally-hosted large language
model (LLM) is used to reduce workload or surface insights. In every case
the LLM is an assistant to human decision-making, not a decision-maker.
All outputs are stored, labelled with provenance, and reviewed before use.

**Model:** Qwen 2.5 32B at Q4 quantisation, running on-premises via Ollama.
High-complexity tasks (committee screening, regulatory alignment) may use a
larger model where hardware permits.

**Delivery mechanism:** All LLM tasks are asynchronous Celery tasks on the
dedicated `llm` queue. They never block the request/response cycle. Results
are written to `LLMResult` records and surfaced in the UI when ready.

**Governance principle:** No LLM output is used to record, trigger, or
constitute a governance decision. Every output is labelled
`generated_by / generated_at / reviewed_by / reviewed_at`.

---

## 1. Committee agenda conflict screening

### Purpose

Before each governance committee meeting (e.g. Faculty Teaching and
Learning Committee), multiple documents are submitted for approval. A
reviewer must currently read every document against every other to catch
inconsistencies. For a busy committee this may mean reading 15–20
documents looking for conflicts that may not be obvious at the level of
any individual submission.

The LLM reads across all submissions and produces a structured screening
report, allowing the committee administrator and chair to focus their
attention on flagged items rather than performing an exhaustive manual
cross-read.

### Trigger

The Celery task is triggered automatically when the submission deadline
for a `CommitteeMeeting` passes. It is also available on demand via an
administrator action.

### Inputs

- All `AssetVersion` records submitted for this meeting (via
  `WorkflowVersionEvent` records linking workflow instances to the
  meeting)
- The committee's standing terms of reference and policy constraints
  (stored as a tenant-level configuration document)
- The approved versions of any documents the submissions seek to amend

### What the LLM does

The prompt instructs the model to read all submissions and identify:

1. **Learning outcome overlaps** — module-level outcomes that duplicate
   or substantially overlap across different submissions at the same
   meeting
2. **Assessment inconsistencies** — proposed assessment weightings or
   formats that conflict with course-level policy stated in another
   submission or in an approved parent document
3. **Terminology inconsistencies** — the same concept described
   differently across related submissions (e.g. "AI literacy" vs
   "digital intelligence" across modules that are explicitly linked)
4. **Policy conflicts** — a proposed change that appears to contradict
   an existing approved policy, even if not directly referenced in the
   submission
5. **Missing cross-references** — submissions that affect shared
   resources (e.g. a shared module appearing in two courses) without
   acknowledging the dependency

The model is instructed to return structured JSON: a list of flagged
items, each with the affected submissions, a brief explanation, and a
severity classification (`advisory`, `significant`, `blocking`).

### Output

A `CommitteeScreeningReport` record is created, linked to the
`CommitteeMeeting`. The report is surfaced in the committee
administrator's view of the meeting, presented as a grouped list of
flags with links to the relevant submissions. The committee chair sees
a summary count on the meeting overview.

The report is explicitly labelled as AI-generated and advisory. The
committee chair or administrator marks each flag as acknowledged,
actioned, or dismissed before the meeting.

### What the LLM does not do

- It does not recommend approval or rejection of any submission
- It does not modify any document or workflow state
- Its output has no formal standing in the governance process unless
  a human acts on it

### Context window requirements

A busy committee may have 15–20 submissions totalling 30,000–60,000
words. This is the primary task driving the choice of a large context
window model. The 32B model at Q4 handles approximately 32k tokens
comfortably; for larger meetings a 70B model or multi-pass
summarisation approach may be needed.

---

## 2. Back-propagation impact analysis

### Purpose

When a course-level learning outcome (PLO) changes — for example, a
new outcome is added: "Students acquire familiarity with AI-driven data
analysis workflows" — the course coordinator must identify which
module-level outcomes need to change, which assessments are affected,
and which modules have a gap. Currently this requires manually reading
all module specifications for the course.

The LLM drafts this impact mapping, which the coordinator reviews,
adjusts, and approves before initiation tasks are pushed to module
convenors.

### Trigger

Fired when a course-level change request is approved and the
backwards scheduler is about to generate initiation tasks. The impact
analysis runs first; the coordinator reviews it before confirming that
initiation tasks should be pushed.

### Inputs

- The proposed new or amended PLO (from the course-level change
  request form)
- All current approved module specifications for the affected course
  (via `AssetVersion` records)
- All current approved module-level learning outcomes
- The existing PLO-to-MLO alignment matrix if one exists

### What the LLM does

The model reads the proposed PLO and all module specifications and
produces:

1. **Modules with existing coverage** — modules where one or more
   current MLOs already substantially address the new PLO. Includes
   a brief explanation of which MLO and how closely it maps.
2. **Modules with partial coverage** — modules where existing MLOs
   partially address the PLO but a gap remains. Identifies the gap.
3. **Modules with no coverage** — modules where no existing MLO
   addresses the new PLO. These require a new MLO to be drafted.
4. **Affected assessments** — assessments currently mapped to MLOs
   that will change, flagged for review.
5. **Draft MLO suggestions** — for modules in category 3, a draft
   candidate MLO derived from the proposed PLO and the module's
   existing content and level descriptor.

Output is structured JSON with one entry per module.

### Output

An `LLMResult` record of type `impact_analysis` is created, linked to
the course-level change request document. The coordinator sees a
structured table in the UI: one row per module, with coverage
classification, explanation, and (where applicable) draft MLO text.

The coordinator can accept, edit, or reject each row. Accepted rows
form the basis of the initiation tasks pushed to module convenors —
each convenor receives their task pre-populated with the impact
analysis for their module, including the draft MLO if one was
generated.

### What the LLM does not do

- It does not initiate any workflows or create any documents
- It does not determine which modules are in scope — that is derived
  from the course's document hierarchy
- Draft MLOs are clearly labelled as AI-generated suggestions requiring
  author review and approval through the normal workflow

---

## 3. Form field pre-population

### Purpose

When a module convenor receives an initiation task — asking them to
produce a new or revised module specification, learning outcome, or
assessment brief — they are typically given a blank or lightly
prefilled form. Much of what they need to write is either derivable
from context (the change being made, the existing document content) or
follows established patterns.

Pre-population reduces the time a convenor spends on routine content
and allows them to focus on the judgements that require their expertise.

### Trigger

Fired when an initiation task is created and assigned to a convenor.
The pre-population runs before the convenor opens the form for the
first time. If the LLM result is not yet ready when they open the
form, the form opens blank and the draft is inserted when it arrives.

### Inputs

- The current approved version of the document being revised
- The change context: what change has been requested, what PLO or
  course-level decision is driving it
- For learning outcomes: the relevant FHEQ level descriptor and any
  applicable Subject Benchmark Statement excerpts
- For assessments: the module's credit value, existing assessment
  pattern, and the MLOs the new assessment must address
- Institutional conventions (word limits, standard phrasing patterns)
  from a tenant-level configuration

### What the LLM does

Depending on the document type:

**Module specification:** Drafts the rationale field explaining why
the change is being made (derived from the change context), and
suggests revisions to the affected sections. Leaves unchanged sections
blank to avoid noise.

**Learning outcome:** Drafts a candidate MLO aligned to the proposed
PLO change and the module's existing level and content. Applies Bloom's
taxonomy verb appropriate to the FHEQ level.

**Assessment brief:** Drafts the assessment description, alignment
mapping (which MLOs the assessment addresses), and suggested marking
criteria structure. Proposes a submission format consistent with the
module's existing pattern.

Output is a JSON object keyed by form field name.

### Output

The form opens pre-populated with the LLM-generated content. Each
pre-populated field is visually marked with an indicator ("AI draft —
please review") and the convenor's subsequent edits are tracked. The
final submitted form carries a record of which fields were AI-drafted
and which were written or substantially edited by the author.

Pre-populated content that is submitted unchanged is flagged for
additional scrutiny at the review stage — reviewers can see which
fields the author engaged with and which were accepted without
modification.

### What the LLM does not do

- It does not submit the form or advance the workflow
- Pre-populated content has no different standing in the governance
  process than content written by the author — it must pass through
  the same review and approval stages

---

## 4. Pre-submission consistency check

### Purpose

Before a document is submitted for committee review, a brief automated
check identifies obvious problems that would typically result in the
document being referred back — saving committee time and reducing
revision cycles.

This is a soft gate: the check surfaces issues and asks the author to
confirm or revise, but does not block submission. The author can
override a flag with a recorded rationale.

### Trigger

Fired when the author initiates the `submit` transition. The check
runs synchronously (or near-synchronously on a fast path) because it
is part of the submission user journey.

### Inputs

- The document being submitted (full content)
- The parent document it relates to (e.g. the course specification
  for a module specification submission)
- Any policy documents referenced in the submission
- The committee's standing requirements for submissions of this type

### What the LLM does

1. **Internal consistency** — does the proposed content contradict
   itself? (e.g. credit points stated in the rationale differ from
   those in the structured fields)
2. **Parent document alignment** — does the proposed content align
   with the parent document it belongs to? (e.g. does the module
   specification's stated level match the course's requirements
   for modules at that point in the structure?)
3. **Rationale adequacy** — does the rationale field actually justify
   the change being made, or is it generic boilerplate? The model
   checks whether the rationale is specific to the proposed change.
4. **Assessment-LO alignment** — for module specifications, does each
   assessment have a plausible mapping to at least one LO?
5. **Completeness** — are required fields populated with substantive
   content rather than placeholder text?

Output is a list of flags, each with a severity (`advisory` or
`significant`) and a plain-English explanation.

### Output

Flags are shown to the author in the submission confirmation screen.
`Advisory` flags can be dismissed without comment. `Significant` flags
require the author to either revise the document or record a rationale
for proceeding. The flag list and any recorded rationales are attached
to the `WorkflowEvent` for the submission transition, making them
visible to reviewers.

---

## 5. Regulatory alignment pre-check

### Purpose

Course and module specifications must align with external regulatory
frameworks: FHEQ level descriptors, relevant Subject Benchmark
Statements, PSRB requirements (where applicable), and OfS conditions
of registration. Manual checking against these frameworks is time-
consuming and requires specialist knowledge.

The LLM pre-check is run before a document reaches committee, giving
the author and quality officers an early view of potential alignment
gaps.

### Trigger

Fired when a document transitions into a review state (e.g.
`FACULTY_REVIEW`). The result is available to the reviewer before
they begin their assessment.

### Inputs

- The submitted document version
- The relevant FHEQ level descriptor (selected by document type and
  level field)
- The relevant Subject Benchmark Statement(s) (selected by subject
  area field, retrieved from a stored reference library)
- Any applicable PSRB requirements (stored per course as tenant
  configuration)

### What the LLM does

1. **FHEQ level alignment** — for each learning outcome, does it
   clearly correspond to a descriptor at the stated FHEQ level? Flags
   outcomes that appear to be pitched at a different level.
2. **Subject Benchmark coverage** — does the course/module address
   the threshold standards described in the relevant Subject Benchmark
   Statement? Identifies benchmark areas with no corresponding
   coverage.
3. **PSRB requirements** — where applicable, checks that required
   elements (specific topics, placement hours, competency statements)
   are present and described.
4. **OfS conditions** — checks for obvious gaps relative to relevant
   OfS conditions of registration (primarily B conditions relating to
   quality and standards).

Output is a structured JSON report: one section per framework, with a
coverage summary and a list of specific gaps or concerns.

### Output

An `LLMResult` record of type `regulatory_check` is attached to the
document version. Reviewers see the report alongside the document in
their review view. The report is advisory — it does not constitute a
formal compliance assessment. A human quality officer is still required
to sign off regulatory alignment as part of the approval workflow.

Significant gaps flagged by the pre-check that are not addressed
before Senate approval are recorded in the audit trail, creating an
evidence base for future regulatory review.

---

## 6. Scheduling plain-English summary

### Purpose

The backwards scheduler produces a set of `ScheduledMilestone` records
with computed dates, risk flags, and critical path indicators. This
output is precise but technical. Course coordinators need to
understand not just the dates but the reasoning behind them —
particularly when committee meeting constraints are creating a tight
deadline or when a document is at risk.

The LLM translates the scheduler output into a plain-English narrative
that coordinators can read quickly and share with academic colleagues.

### Trigger

Fired whenever a `PublicationTarget` is created or updated, and
whenever the schedule is recomputed (e.g. when committee meeting dates
change or a new target is added that affects existing milestones).

### Inputs

- The full set of `ScheduledMilestone` records for the target
- The `CommitteeMeeting` records used in the schedule computation
- The document dependency graph (serialised as a structured list of
  dependencies and their resolution rules)
- Any at-risk or deferred milestones and their characterisation

### What the LLM does

Produces a short narrative (3–5 paragraphs) covering:

1. **The target and its deadline** — what is being aimed for and by
   when, in plain terms
2. **The critical path** — which documents are on the critical path,
   which committee meetings are the binding constraints, and what the
   key submission deadlines are
3. **Actions needed now** — which documents need to be initiated
   immediately, and by whom
4. **Risks** — any milestones that are at risk, with a plain-English
   explanation of why (e.g. "The March TLC meeting is the last one
   before the course specification submission deadline. If the
   COMP4012 assessment brief is not submitted by 14 February, it will
   miss that meeting and the course specification cannot proceed to
   Senate in May.")
5. **Decisions needed** — any deferred linking decisions that require
   the coordinator's attention

Output is plain text (Markdown).

### Output

The narrative is displayed at the top of the publication target
dashboard, replacing the raw milestone table as the primary view for
coordinators who are not working through the detail. The full milestone
table remains available below it. The narrative is regenerated each
time the schedule is recomputed, and the previous version is archived
in the `LLMResult` history.

---

## 7. Assessment congestion analysis (LLM-assisted component)

### Purpose

The assessment congestion check (which identifies weeks where student
submission load is excessive) is primarily a deterministic algorithm
over structured data. The LLM component provides a plain-English
summary of the congestion analysis and, where congestion is identified,
suggests which assessments might be candidates for rescheduling and why.

### Trigger

Fired as part of the assessment congestion Celery task, after the
deterministic analysis has completed.

### Inputs

- The output of the deterministic congestion analysis (week-by-week
  submission load, flagged weeks, affected cohorts)
- The proposed assessments that triggered the analysis
- The existing assessment schedule for the affected cohorts

### What the LLM does

1. Explains the congestion in plain English: which weeks are affected,
   which student cohorts, and by how much the threshold is exceeded
2. Identifies which proposed assessments are contributing most to the
   congestion
3. Suggests candidate assessments for rescheduling (from the proposed
   set or the existing schedule), with reasoning based on assessment
   type and module dependencies
4. Notes any constraints that limit rescheduling options (e.g. an
   assessment that must fall after a specific teaching block)

### Output

Appended to the `CongestionReport` record as a narrative field.
Surfaced to the timetabling coordinator alongside the raw congestion
data. Suggestions are advisory — the coordinator makes the final
decision on any rescheduling.

---

## 8. Coordinator brief adequacy check

### Purpose

When a module convenor submits a proposed MLO (or other downstream
document) in response to a coordinator-initiated change, the coordinator
must judge whether the proposed content actually meets the brief they set.
This check reads the original change context — the CLO, the coordinator's
rationale, and the task guidance note — and assesses whether the proposed
content adequately addresses them.

This is distinct from the pre-submission consistency check (workflow 4),
which is author-facing and checks internal document coherence. This check
is coordinator-facing and checks adequacy against the originating brief.

### Trigger

Fired when a `ModuleOutcomeContent` (or other coordinator-scoped
downstream document) enters `COORDINATOR_REVIEW` state. Runs
asynchronously in the background; the coordinator sees the result when
they open the review task.

### Inputs

- The proposed MLO text from `ModuleOutcomeContent`
- The CLO that drove the change, from `CourseOutcomeContent`
- The `change_rationale` field from the `Document` envelope of the
  originating CLO workflow
- The `context_note` from the `Task` record that was pushed to the
  convenor
- The FHEQ level from `ModuleIdentifier`, used to assess whether the
  proposed outcome is pitched at the right level

### What the LLM does

1. **Brief coverage** — does the proposed MLO address the CLO it was
   spawned to support? Does it cover the full scope of the CLO or only
   part of it?
2. **Level appropriateness** — is the proposed outcome pitched at the
   correct FHEQ level, consistent with the module's level and the
   coordinator's guidance?
3. **Guidance adherence** — does the proposed content reflect the
   specific framing or constraints set out in the coordinator's task
   note, where one was provided?
4. **Rationale alignment** — is the proposed change consistent with the
   stated rationale for the CLO introduction?

Output is a short structured report with a headline verdict
(`meets_brief`, `partially_meets_brief`, `does_not_meet_brief`) and a
plain-English explanation for each dimension above.

### Output

An `LLMResult` record of type `brief_adequacy_check` is attached to the
`ModuleOutcomeContent` document and surfaced in the coordinator's review
view alongside the document content. The headline verdict and explanation
are displayed prominently at the top of the review panel.

`partially_meets_brief` and `does_not_meet_brief` verdicts are treated as
significant flags. The coordinator can proceed and push the document to
`TLC_REVIEW` despite a flag, but must record a rationale for doing so.
That rationale is attached to the `WorkflowEvent` for the transition and
is visible to TLC reviewers.

### What the LLM does not do

- It does not block the coordinator from advancing the workflow
- It does not assess the MLO against Subject Benchmark Statements or
  FHEQ descriptors — that is the role of the regulatory alignment
  pre-check (workflow 5)
- It does not evaluate whether the MLO is well-written or internally
  consistent — that is the role of the pre-submission consistency
  check (workflow 4)

---

## Cross-cutting implementation notes

### Human oversight at every stage

Every LLM workflow follows the same pattern:

```
Trigger (FSM transition or schedule event)
    ↓
Celery task dispatched to llm queue
    ↓
LLMResult record created (status: pending)
    ↓
Inference runs (Ollama)
    ↓
LLMResult updated (status: complete, output stored)
    ↓
UI notification to relevant user
    ↓
Human reviews, accepts/edits/rejects
    ↓
LLMResult updated (reviewed_by, reviewed_at, accepted)
```

### Prompt management

Prompts are stored as versioned template strings in the codebase
(not in the database). Each prompt version is identified by a hash
stored on the `LLMResult` record, making it possible to audit which
prompt version produced any given output.

### Failure handling

LLM inference can fail or time out. All LLM tasks implement:
- Retry with exponential backoff (up to 3 attempts)
- Graceful degradation — if inference fails, the workflow continues
  without the LLM output and the user is notified that the AI
  assistance is unavailable
- The absence of an LLM result never blocks a workflow transition

### Data sovereignty

All inference runs on-premises via Ollama. No document content,
learning outcome text, or institutional data is sent to external APIs.
This is a hard architectural constraint, not a configuration option.

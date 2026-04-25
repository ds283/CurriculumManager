# Scheduling and Publication Target — Design Decisions

Captured during design iteration on `PublicationTarget` provisional status
(G7) and extended to cover the broader scheduling subsystem. Decisions here
apply across all document workflows, not only the CLO workflow.

---

## Design Decisions

These are resolved positions. Future design work should not contradict them
without explicitly reconsidering the rationale recorded here.

---

### S1 — `PublicationTarget.target_date` is coordinator-owned; the scheduler
never writes to it

`target_date` is set by the owner of the `PublicationTarget` and may only be
changed by a user action. The scheduler does not modify it. When the scheduler
detects that the current `target_date` is no longer achievable it emits a
`SchedulerAdvisory` of type `SUGGESTED_RESCHEDULE` carrying a
`suggested_target_date`, which is surfaced to the target owner as an advisory
notification. The owner confirms or rejects the suggestion; confirmation
creates a `PublicationTargetDateChange` log entry and triggers a scheduler
recompute against the new date.

**Rationale:** keeping `target_date` coordinator-owned preserves the clean
audit trail of deliberate governance decisions. Automatic adjustment by the
scheduler would make the date a computed artefact rather than a governance
commitment, eroding its meaning in regulatory and accreditation contexts.

---

### S2 — Target date changes are recorded in an explicit audit log

Every change to `PublicationTarget.target_date` is recorded in a
`PublicationTargetDateChange` record carrying the previous date, new date,
acting user, timestamp, and an optional rationale. This is not covered by
`django-simple-history` field tracking alone, which does not capture the
advisory context or the deliberate governance nature of the change.

---

### S3 — Date confidence is a four-level `TextChoices` on both
`PublicationTarget` and `CommitteeMeeting`

The same vocabulary is used on both models:

| Value         | Meaning                                                                 |
|---------------|-------------------------------------------------------------------------|
| `WINDOW`      | Only a date range is known; no point estimate yet                       |
| `PROVISIONAL` | A specific date has been nominated but is explicitly subject to change  |
| `INDICATIVE`  | Derived from a known recurring pattern; reliable enough for planning    |
| `CONFIRMED`   | Formally set and published                                              |

Confidence ordering from least to most reliable:
`WINDOW < PROVISIONAL < INDICATIVE < CONFIRMED`

Transitions on `PublicationTarget.date_confidence` are unidirectional toward
`CONFIRMED`, enforced in `clean()`. A target may not regress from CONFIRMED
to a lower state; a new target must be created if governance circumstances
change materially.

**Rationale for INDICATIVE above PROVISIONAL:** an indicative date is drawn
from a known institutional calendar pattern and is reliable in a systemic
sense. A provisional date is a nominated point estimate carrying an explicit
caveat of uncertainty. The pattern-derived date is more trustworthy than the
caveat-bearing one.

---

### S4 — `ScheduledMilestone` carries a derived date confidence and uncertainty
envelope

The scheduler propagates `date_confidence` to each `ScheduledMilestone` as
the minimum confidence across all linked `PublicationTarget` records
(`MIN` in the ordering `WINDOW < PROVISIONAL < INDICATIVE < CONFIRMED`).
The shared-milestone case is handled automatically: a milestone shared across
two targets with different confidence levels inherits the lower value.

When `date_confidence` is below `CONFIRMED`, the scheduler also computes an
uncertainty envelope:

- `earliest_must_complete` — computed from the earliest plausible target date
  (tightest deadline)
- `latest_must_complete` — computed from the latest plausible target date
  (most relaxed deadline)
- `earliest_must_initiate` — computed from the earliest plausible target date
- `latest_must_initiate` — computed from the latest plausible target date

These fields are NULL when `date_confidence` is `CONFIRMED`. The UI should
present the range rather than the point estimate when the envelope is
populated, particularly for `must_be_initiated_by`, since that is the field
convenors act on.

---

### S5 — `CommitteeMeeting` supports both standing and exceptional meetings

Standing committee meetings (TLC, Board of Studies, etc.) are entered against
a mandated cadence and typically carry `INDICATIVE` confidence before formal
confirmation. They may be populated several years in advance. The
`earliest_expected_date` / `latest_expected_date` window captures the
institutionally mandated scheduling range.

Exceptional meetings created ad hoc for outstanding items are flagged with
`is_exceptional = True`. These are unlikely to have a meaningful date window
and are expected to follow a `PROVISIONAL → CONFIRMED` lifecycle with a short
lead time.

`meeting_date` is nullable to support the `WINDOW` state, where only a range
is known. It becomes required when `date_confidence` is `INDICATIVE` or
`CONFIRMED`, enforced in `clean()`. `submission_deadline` is also nullable,
populated once `meeting_date` is known.

---

### S6 — `CommitteeMeeting` confirmation is audited

When a `CommitteeMeeting.date_confidence` transitions to `CONFIRMED`, the
acting user and timestamp are recorded in `confirmed_by` / `confirmed_at`.
This transition triggers a scheduler recompute for all `PublicationTarget`
records whose critical path includes this meeting, propagating revised
confidence and date envelopes to downstream milestones.

---

### S7 — `PublicationTarget` carries a computed schedule status

The backwards scheduler sets `schedule_status` on each `PublicationTarget`
on every recompute. The status reflects the overall achievability of the
target given the current state of its milestones.

| Value           | Terminal? | Set by    | Meaning                                                             |
|-----------------|-----------|-----------|---------------------------------------------------------------------|
| `ON_TRACK`      | No        | Scheduler | All milestones within tolerance                                     |
| `AT_RISK`       | No        | Scheduler | One or more milestones flagged `at_risk`                            |
| `CRITICAL`      | No        | Scheduler | Path technically still open but requires immediate action           |
| `UNACHIEVABLE`  | No        | Scheduler | No viable path to target date; recovery possible via reschedule     |
| `COMPLETED`     | Yes       | Scheduler | All milestones resolved; normal successful outcome                  |
| `ABANDONED`     | Yes       | Owner     | Closed without publication; deliberate management decision          |
| `SUPERSEDED`    | Yes       | Owner     | All workflows migrated to rescoped targets; nothing remains here    |

`UNACHIEVABLE` is **not** terminal. A coordinator may recover by adjusting the
target date (see S1), after which the scheduler may move the status back to
`AT_RISK` or `ON_TRACK`. `ABANDONED` requires a rationale (see S8).

`schedule_status` is the only field on `PublicationTarget` that the scheduler
writes to. `target_date` and all other user-owned fields are never touched by
the scheduler.

---

### S8 — Abandonment is a deliberate owner action requiring rationale

Marking a `PublicationTarget` as `ABANDONED` requires the owner to supply a
rationale, recorded on `PublicationTarget.abandoned_rationale`. This is a
governance action, not a scheduling outcome. It is distinct from
`UNACHIEVABLE`, which is a computed state indicating the schedule cannot be
met but recovery may still be possible (via reschedule or descope).

---

### S9 — Slippage detection runs on a daily Celery Beat schedule

The backwards scheduler is fired on a daily Celery Beat schedule in addition
to its event-driven triggers (meeting date changes, milestone state changes,
`MilestonePublicationTarget` additions/removals). The daily run ensures that
the passage of time alone — specifically, `must_be_initiated_by` or
`must_be_complete_by` dates crossing `date.today()` — is detected promptly
without waiting for an external event.

The scheduler is idempotent: if no inputs have changed since the previous run,
it writes nothing and fires no notifications. This makes daily runs cheap as
the number of active targets grows.

**Rationale:** without periodic runs, a milestone could become `is_overdue` and
a target could tip into `UNACHIEVABLE` silently, with the coordinator only
discovering the situation when an unrelated event (e.g. a committee meeting
date change) triggers the next recompute. Daily runs bound the detection lag
to at most one day.

---

### S10 — Scheduler advisories are deduplicated via `SchedulerAdvisory`

When the scheduler detects a condition warranting coordinator attention, it
creates a `SchedulerAdvisory` record rather than emitting a bare notification.
Before creating a new advisory, the scheduler checks for an open advisory of
the same `advisory_type` on the same `publication_target`; if one exists, no
duplicate is created. The coordinator dismisses the advisory (recording
`dismissed_by` / `dismissed_at`), after which a new advisory of the same
type may be raised if the condition recurs.

This prevents repeated daily runs from generating notification noise.

Advisory types:

| Value                   | Condition                                                           |
|-------------------------|---------------------------------------------------------------------|
| `SUGGESTED_RESCHEDULE`  | Scheduler has computed a suggested revised target date              |
| `MILESTONE_CRITICAL`    | One or more milestones have entered CRITICAL status                 |
| `TARGET_UNACHIEVABLE`   | Target has transitioned to UNACHIEVABLE                             |

---

### S11 — `COMPLETED` replaces `is_active`; active state is a computed property

`PublicationTarget.is_active` is removed as a stored field. Its role is
replaced by:

1. **`COMPLETED` terminal status** — the normal successful outcome, recorded
   by the scheduler when all milestones have reached their `required_state`
   and the target date has passed. Distinct from `ABANDONED` (closed without
   publication) and `SUPERSEDED` (all workflows migrated to rescoped targets).

2. **`is_active` computed property** — returns `True` when `schedule_status`
   is not in `{COMPLETED, ABANDONED, SUPERSEDED}`. Used in application code
   where a boolean predicate is convenient.

3. **Partial index on non-terminal statuses** — replaces the query role of the
   stored boolean:

   ```python
   # In Meta.indexes:
   Index(
       fields=['tenant', 'schedule_status'],
       condition=~Q(schedule_status__in=['COMPLETED', 'ABANDONED', 'SUPERSEDED']),
       name='publication_target_active_idx',
   )
   ```

   The `TenantScopedManager` default queryset filters on this condition rather
   than `is_active=True`.

**Rationale:** a stored boolean that must be kept in sync with `schedule_status`
is a maintenance liability. The partial index provides equivalent query
performance without the risk of the two fields diverging. `COMPLETED` fills the
semantic gap — previously there was no way to record that a target had finished
normally, only that it had been abandoned or superseded.

**Terminal status summary:**

| Value       | Set by      | Meaning                                               |
|-------------|-------------|-------------------------------------------------------|
| `COMPLETED` | Scheduler   | All milestones resolved; normal successful outcome    |
| `ABANDONED` | Owner       | Closed without publication; deliberate decision       |
| `SUPERSEDED`| Owner       | All workflows migrated to rescoped targets            |

`UNACHIEVABLE` is **not** terminal — a coordinator may recover by adjusting
the target date, after which the scheduler may move the status back to
`AT_RISK` or `ON_TRACK`.

# Document Models

This file specifies the content models for each `Document.DocumentType` value registered
in `data_models.md`. It is a companion to `data_models.md`, not a replacement — the
`Document` envelope, workflow layer, asset versioning, and all other cross-cutting tables
are documented there.

## Conventions

Each document type has exactly one content model following the pattern
`OneToOneField(Document, related_name='[type]')`. Content models carry only
type-specific fields; shared metadata lives on the `Document` envelope.

Models marked **stub** have a settled `DocumentType` registration and will appear in
migrations, but their fields are incomplete pending specification work with the
curriculum team. The `document` OneToOneField is the only field generated for stubs.

Models marked **deferred** have an unresolved governance question that must be answered
before their FSM and fields can be specified. They are registered in `DocumentType` to
satisfy the startup validation check and will be stubbed in migrations until resolved.

For the extension checklist that must be followed when promoting a stub or deferred model
to a fully specified one, see `DESIGN.md`.

---

## Content models

### `ModuleSpecificationContent`

| Field           | Type                     | Notes                                                                                                              |
|-----------------|--------------------------|--------------------------------------------------------------------------------------------------------------------|
| `id`            | PK                       |                                                                                                                    |
| `document`      | OneToOneField → Document | `related_name='module_spec'`                                                                                       |
| `credit_points` | PositiveIntegerField     |                                                                                                                    |
| `level`         | FK → FHEQLevel           | FHEQ level. FK to controlled vocabulary, not free CharField                                                        |
| `subject_area`  | CharField                |                                                                                                                    |
| `period`        | FK → PeriodIdentifier    | Delivery semester. If partial-period delivery is introduced later, add M2M to `PeriodWeekIdentifier` at that point |
| `outline`       | TextField                | Stored as HTML (WYSIWYG input). May be diffed between versions in the change request review UI                     |

**Accessing content:**

```python
doc = Document.objects.select_related('module_spec').get(pk=pk)
content = doc.module_spec  # single indexed join, no second query
```

**Note:** The associations between a module specification and its learning
outcomes, assessments, and teaching activities are navigated via
`ModuleIdentifier`, not via FKs on this model. The module specification
page is a dashboard view that assembles the current approved state of all
governed objects anchored to the `ModuleIdentifier` — it does not own them.

**Integrity note:** `period.tenant` must equal `document.tenant` — enforced
in `clean()`.

---

### `TeachingActivityContent`

A governed document describing one type of teaching activity delivered
as part of a module. Anchored to `ModuleIdentifier`. Has its own FSM
and `AssetVersion` history.

Multiple `TeachingActivityContent` documents can be in the `APPROVED`
state simultaneously for the same module. The current approved teaching
pattern for a module is therefore a queryset filtered by `module_identifier`
and current approved state, not a single object.

There is no slot registry for teaching activities (contrast with
`AssessmentSlotIdentifier`). Teaching patterns are ephemeral: no other
governed document needs to reference a specific teaching activity as a
stable anchor.

| Field            | Type                     | Notes                                                                      |
|------------------|--------------------------|----------------------------------------------------------------------------|
| `id`             | PK                       |                                                                            |
| `document`       | OneToOneField → Document | `related_name='teaching_activity'`                                         |
| `module`         | FK → ModuleIdentifier    | `related_name='teaching_activities'`                                       |
| `activity_type`  | CharField                | `TextChoices`: `lecture`, `workshop`, `seminar`, `lab`, `other`            |
| `duration_hours` | DecimalField             | Duration of a single occurrence in hours. `max_digits=4, decimal_places=1` |
| `period`         | FK → PeriodIdentifier    | The period this pattern applies to                                         |
| `ordering`       | PositiveIntegerField     | Display order within the module's activity list                            |

**Constraints:** No uniqueness constraint on `(module, activity_type)` — a
module may have multiple activities of the same type. Multiple
`TeachingActivityContent` documents may be in the `APPROVED` state
simultaneously for the same module.

**Integrity note:** `document.tenant` must equal `module.tenant` — enforced
in `clean()`.

**Implementation note:** The timetabling spreadsheet export reads
`week_pattern` as a derived property. Queries such as "all teaching
activities in Week 7" operate on `TeachingActivityWeek` directly, not
on the pattern string.

**Derived properties** (not persisted):

- `week_pattern` — digit string e.g. `'33333333333'`; position *i*
  (0-based) = occurrences in `PeriodWeekIdentifier` at sequence *i+1*.
  Suitable for display and timetabling export.
- `total_contact_hours` — sum of `occurrences × duration_hours` across all
  `TeachingActivityWeek` child records.

```python
@property
def week_pattern(self):
  """
  Returns the week pattern as a digit string, e.g. '33333333333'.
  Position i (0-based) = occurrences in PeriodWeekIdentifier at
  sequence i+1. Suitable for display and timetabling export.
  """
  weeks = self.weeks.select_related('period_week').order_by(
    'period_week__sequence'
  )
  return ''.join(str(w.occurrences) for w in weeks)


@property
def total_contact_hours(self):
  return sum(
    w.occurrences * self.duration_hours for w in self.weeks.all()
  )
```

---

### `TeachingActivityWeek`

Child table of `TeachingActivityContent`. One row per `PeriodWeekIdentifier`
in which the activity occurs. Provides an indexed query anchor for
week-level queries that cannot be answered efficiently by parsing the
`week_pattern` string.

| Field         | Type                         | Notes                                                                               |
|---------------|------------------------------|-------------------------------------------------------------------------------------|
| `id`          | PK                           |                                                                                     |
| `activity`    | FK → TeachingActivityContent | `related_name='weeks'`, `on_delete=CASCADE`                                         |
| `period_week` | FK → PeriodWeekIdentifier    | The specific week this row describes                                                |
| `occurrences` | PositiveSmallIntegerField    | Number of times the activity occurs in this week. The digit value in `week_pattern` |

**Constraints:** `unique_together = (activity, period_week)`. The week set
must cover all weeks in the activity's period exactly once — enforced in
`TeachingActivityContent.clean()`.

**Week-level query pattern:**

```python
# All teaching activities occurring in a specific week
TeachingActivityWeek.objects.filter(
  period_week=week,
  occurrences__gt=0,
).select_related('activity__module')
```

---

### `LearningOutcomeContent`

| Field          | Type                     | Notes                                                                                                                                                         |
|----------------|--------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `id`           | PK                       |                                                                                                                                                               |
| `document`     | OneToOneField → Document | `related_name='learning_outcome'`                                                                                                                             |
| `module`       | FK → ModuleIdentifier    | `related_name='learning_outcomes'`                                                                                                                            |
| `outcome_code` | CharField                | Stable human-readable identifier, e.g. `MLO-A`. Assigned at approval time as a workflow side effect, never at form initiation. Never reused after retirement. |
| `text`         | TextField                | The outcome statement. Plain text.                                                                                                                            |
| `ordering`     | PositiveIntegerField     | Display sequence within the module's outcome list                                                                                                             |

**Constraints:** `unique_together = (module, outcome_code)`

**Integrity note:** `document.tenant` must equal `module.tenant` — enforced
in `clean()`.

**Note on `outcome_code` assignment:** the code is not set by the author on
the introduction form. It is assigned by the FSM approval side effect and
recorded on the resulting `WorkflowEvent.metadata` and `AssetVersion`. This
means reviewers see "proposed new outcome" during the review period rather
than the eventual code — acceptable given the short review window.

**Note on FHEQ level:** no per-outcome FHEQ field is needed. The FHEQ level
is carried on `ModuleIdentifier` and applies implicitly to all outcomes
anchored to that module. The LLM regulatory pre-check uses the module-level
`FHEQLevel` when assessing outcome alignment.

---

### `CourseSpecificationContent`

No fields specified yet beyond the structural anchor. Stub pending
specification work with the curriculum team.

| Field      | Type                     | Notes                           |
|------------|--------------------------|---------------------------------|
| `id`       | PK                       |                                 |
| `document` | OneToOneField → Document | `related_name='course_spec'` |

---

### `AssessmentContent`

| Field             | Type                                 | Notes                                                                                                                           |
|-------------------|--------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| `id`              | PK                                   |                                                                                                                                 |
| `document`        | OneToOneField → Document             | `related_name='assessment'`                                                                                                     |
| `slot`            | FK → AssessmentSlotIdentifier        | Stable identity anchor. Assessment type is carried on the slot, not repeated here                                               |
| `module`          | FK → ModuleIdentifier                | Denormalised from `slot.module` for query efficiency. Must equal `slot.module` — enforced in `clean()`                          |
| `weight`          | PositiveSmallIntegerField            | Percentage weighting, 0–100                                                                                                     |
| `term`            | FK → PeriodIdentifier (nullable)     | Teaching term this assessment falls within. NULL for assessments not tied to a specific term                                    |
| `submission_week` | FK → PeriodWeekIdentifier (nullable) | Week-level anchor for congestion analysis and scheduling                                                                        |
| `submission_day`  | CharField (nullable)                 | `TextChoices`: `monday`, `tuesday`, `wednesday`, `thursday`, `friday`. Friday submissions flagged in future congestion analysis |
| `submission_time` | TimeField (nullable)                 | Time of day for submission. Nullable independently of `submission_day` — week may be known before day is confirmed              |
| `submit_to`       | CharField (blank)                    | Submission platform for governance record, e.g. `Canvas Turnitin`. No operational integration with this system                  |
| `conflation_rule` | CharField (blank)                    | Pending stakeholder discussion. May move to a pattern-level model                                                               |

**Constraints:** `unique_together = (module, slot)`

**Integrity note:** `document.tenant` must equal `module.tenant`, and
`slot.module` must equal `module` — both enforced in `clean()`.

**Note on pass mark:** not held per assessment. Pass mark is derived from
`FHEQLevel` via institutional policy. To be captured as a policy lookup
model when course regulations are in scope.

**Note on `submission_day`:** a Friday value will be flagged by the
assessment congestion analysis once that is implemented.

---

### `AssessmentLearningOutcome` (join table)

The many-to-many between `AssessmentContent` and `LearningOutcomeContent`,
held as an explicit join table rather than a Django implicit M2M so that
the relationship can carry provenance and be referenced by `WorkflowEvent`.

| Field        | Type                        | Notes                                |
|--------------|-----------------------------|--------------------------------------|
| `id`         | PK                          |                                      |
| `assessment` | FK → AssessmentContent      | `related_name='outcome_mappings'`    |
| `outcome`    | FK → LearningOutcomeContent | `related_name='assessment_mappings'` |

**Constraints:** `unique_together = (assessment, outcome)`

**Integrity note:** `assessment.module` must equal `outcome.module` —
enforced in `clean()`. An assessment may only be mapped to outcomes
anchored to the same module.

---

### `ReaccreditationContent`

Stub pending specification work. Follows the same structural pattern as
other content models.

| Field      | Type                     | Notes                            |
|------------|--------------------------|----------------------------------|
| `id`       | PK                       |                                  |
| `document` | OneToOneField → Document | `related_name='reaccreditation'` |

---

### `ModuleRequisiteContent`

**Deferred** — governance model TBD with curriculum team. Whether requisite
changes go through a standalone workflow or as part of a module specification
change request is unresolved. Stub only.

| Field      | Type                     | Notes                             |
|------------|--------------------------|-----------------------------------|
| `id`       | PK                       |                                   |
| `document` | OneToOneField → Document | `related_name='module_requisite'` |

---

### `CurriculumReviewContent`

**Deferred** — follows the same pattern as `ReaccreditationContent`. Stub
only.

| Field      | Type                     | Notes                              |
|------------|--------------------------|------------------------------------|
| `id`       | PK                       |                                    |
| `document` | OneToOneField → Document | `related_name='curriculum_review'` |

## Content models

### `ModuleSpecificationContent`

| Field           | Type                     | Notes                                                                                                              |
|-----------------|--------------------------|--------------------------------------------------------------------------------------------------------------------|
| `id`            | PK                       |                                                                                                                    |
| `document`      | OneToOneField → Document | `related_name='module_spec'`                                                                                       |
| `credit_points` | PositiveIntegerField     |                                                                                                                    |
| `level`         | FK → FHEQLevel           | FHEQ level. FK to controlled vocabulary, not free CharField                                                        |
| `subject_area`  | CharField                |                                                                                                                    |
| `period`        | FK → PeriodIdentifier    | Delivery semester. If partial-period delivery is introduced later, add M2M to `PeriodUnitIdentifier` at that point |
| `outline`       | TextField                | Stored as HTML (WYSIWYG input). May be diffed between versions in the change request review UI                     |

**Note:** The associations between a module specification and its learning
outcomes, assessments, and teaching activities are navigated via
`ModuleIdentifier`, not via FKs on this model. The module specification
page is a dashboard view that assembles the current approved state of all
governed objects anchored to the `ModuleIdentifier` — it does not own them.

**Integrity note:** `document.tenant` must equal `level`'s implied tenant
context, and `period.tenant` must equal `document.tenant` — enforced in
`clean()`.

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

### `TeachingActivityContent`

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
simultaneously for the same module — one per activity type (or more).

**Integrity note:** `document.tenant` must equal `module.tenant` — enforced
in `clean()`.

**Derived properties** (not persisted):

- `week_pattern` — digit string e.g. `'33333333333'`; position *i*
  (0-based) = occurrences in `PeriodUnitIdentifier` at sequence *i+1*.
  Suitable for display and timetabling export.
- `total_contact_hours` — sum of `occurrences × duration_hours` across all
  `TeachingActivityWeek` child records.

---

### `TeachingActivityWeek`

Child table of `TeachingActivityContent`. One row per
`PeriodUnitIdentifier` in which the activity occurs. Provides an indexed
query anchor for week-level queries that cannot be answered efficiently by
parsing the `week_pattern` string.

| Field         | Type                         | Notes                                                                                |
|---------------|------------------------------|--------------------------------------------------------------------------------------|
| `id`          | PK                           |                                                                                      |
| `activity`    | FK → TeachingActivityContent | `related_name='weeks'`, `on_delete=CASCADE`                                          |
| `period_unit` | FK → PeriodUnitIdentifier    | The specific week this row describes                                                 |
| `occurrences` | PositiveSmallIntegerField    | Number of times the activity occurs in this week. The digit value in `week_pattern`. |

**Constraints:** `unique_together = (activity, period_unit)`. The week set
must cover all units in the activity's period exactly once — enforced in
`TeachingActivityContent.clean()`.

**Week-level query pattern:**

```python
# All teaching activities occurring in a specific week
TeachingActivityWeek.objects.filter(
    period_unit=week,
    occurrences__gt=0,
).select_related('activity__module')
```

---

### `ProgrammeSpecificationContent`

No fields specified yet beyond the structural anchor. Stub pending
specification work with the curriculum team.

| Field      | Type                     | Notes                           |
|------------|--------------------------|---------------------------------|
| `id`       | PK                       |                                 |
| `document` | OneToOneField → Document | `related_name='programme_spec'` |

---

### `AssessmentContent`

No fields specified yet. Stub pending specification work. A FK to
`AssessmentSlotIdentifier` will be the first field added when fields are
settled — that is the stable identity anchor for assessments across
versions.

| Field      | Type                     | Notes                       |
|------------|--------------------------|-----------------------------|
| `id`       | PK                       |                             |
| `document` | OneToOneField → Document | `related_name='assessment'` |

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

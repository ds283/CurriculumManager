# Data Model Reference

All models are Django ORM models targeting PostgreSQL. Every tenant-scoped
model carries a `tenant` FK and is accessed via `TenantScopedManager`, which
raises `RuntimeError` on unscoped access. Append-only models are never
updated or deleted after creation.

---

## Core tenancy

### `Tenant`

Represents an institution. All scoped data is isolated per tenant.
Resolved from the request subdomain, not from the authenticated user.

| Field        | Type             | Notes                |
|--------------|------------------|----------------------|
| `id`         | PK               |                      |
| `name`       | CharField        | Display name         |
| `slug`       | SlugField unique | Subdomain identifier |
| `is_active`  | BooleanField     |                      |
| `created_at` | DateTimeField    | auto                 |

---

### `TenantMembership`

Links a global user to a tenant with a specific role. A user may
belong to multiple tenants (e.g. external examiners), and have
multiple roles within a single tenant (e.g. it's possible to have
`convenor`, `reviewer`, and possibly a combination of  `*_tlc` or
`*_tlc_chair` roles).

| Field       | Type         | Notes                                                                                                                                                                                                                                                                                                                                                     |
|-------------|--------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `id`        | PK           |                                                                                                                                                                                                                                                                                                                                                           |
| `user`      | FK → User    | Global Django user                                                                                                                                                                                                                                                                                                                                        |
| `tenant`    | FK → Tenant  |                                                                                                                                                                                                                                                                                                                                                           |
| `role`      | CharField    | `TextChoices`: `convenor`, `reviewer`, `admin`, `department_tlc`, `department_tlc_chair`, `school_tlc`, `school_tlc_chair`, `faculty_tlc`, `faculty_tlc_chair`, `manager`, `external_examiner`, `review_coordinator`, `review_lead`, `accreditation_coordinator`, `accreditation_lead`, `curriculum_officer`, `course_coordinator`, `publication_manager` |
| `is_active` | BooleanField |                                                                                                                                                                                                                                                                                                                                                           |

**Constraints:** `unique_together = (user, role, tenant)`

---

## Registry layer

Most registry models carry an explicit `tenant` FK even though tenant
is derivable by traversing the FK chain. This matches the
convention used throughout the data model and allows `TenantScopedManager`
to apply a uniform `.filter(tenant=)` guard without per-model JOIN logic.
Consistency between the local `tenant` FK and the tenant implied by the
parent FK is enforced in `clean()` and should be covered by a database-level
check constraint on each model.

### `FacultyIdentifier`

Organizational unit. An umbrella for a group of departments. May have its own
committees that approve certain types of document.

| Field    | Type        | Notes |
|----------|-------------|-------|
| `id`     | PK          |       |
| `tenant` | FK → Tenant |       |
| `name`   | CharField   |       |

**Constraints:** `unique_together = (name, tenant)`

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

---

### `SchoolIdentifier`

Organizational unit intermediate between Faculty and Department. May have its
own committees that approve certain types of document.

| Field     | Type                   | Notes                                               |
|-----------|------------------------|-----------------------------------------------------|
| `id`      | PK                     |                                                     | 
| `tenant`  | FK → Tenant            | Must equal `faculty.tenant` — enforced in `clean()` |
| `faculty` | FK → FacultyIdentifier |                                                     |
| `name`    | CharField              |                                                     |

**Constraints:** `unique_together = (name, faculty, tenant)`

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

---

### `DepartmentIdentifier`

Organizational unit. Owns courses and modules. May have its own committees that
approve certain types of document.

| Field     | Type                             | Notes                                                                                     |
|-----------|----------------------------------|-------------------------------------------------------------------------------------------|
| `id`      | PK                               |                                                                                           | 
| `tenant`  | FK → Tenant                      | Must equal `faculty.tenant`, and also `school.tenant` if not null — enforced in `clean()` |
| `faculty` | FK → FacultyIdentifier           |                                                                                           |
| `school`  | FK → SchoolIdentifier (nullable) | Optional. Schools may not be used.                                                        |
| `name`    | CharField                        |                                                                                           |

**Constraints:** `unique_together = (name, faculty, tenant)`,
`unique_together = (name, school)` where `school` is non-null — a
department name must be unique within its immediate parent regardless
of whether that parent is a faculty or a school

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

---

### `PeriodIdentifier`

A defined period within an academic year, such as a semester or term.
Periods are ordered per tenant by an explicit `sequence` value managed
through the `reorder()` and `append()` classmethods — not set directly
on save.

| Field      | Type                 | Notes                                  |
|------------|----------------------|----------------------------------------|
| `id`       | PK                   |                                        |
| `tenant`   | FK → Tenant          |                                        |
| `name`     | CharField            | e.g. `Autumn Semester`                 |
| `sequence` | PositiveIntegerField | Managed — use `reorder()` / `append()` |

**Constraints:** `unique_together = (name, tenant)`,
`UniqueConstraint(tenant, sequence)` — enforces no two periods share a
position within the same tenant

**Ordering:** `sequence ASC`

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

**Sequencing methods:**

```python
# Append a new period at the end of the sequence (cheap — one MAX query)
PeriodIdentifier.append(tenant, name="Spring Semester")

# Reassign positions 1..N from a caller-supplied ordered list of PKs.
# Two-pass update avoids transient UniqueConstraint violations on PostgreSQL.
PeriodIdentifier.reorder(tenant, ordered_ids=[3, 1, 2])
```

---

### `PeriodWeekIdentifier`

A defined interval within a period, taken to be a week.
Weeks are ordered within their parent period by `sequence`,
managed through the same two-pass pattern as `PeriodIdentifier`.

| Field      | Type                  | Notes                                              |
|------------|-----------------------|----------------------------------------------------|
| `id`       | PK                    |                                                    |
| `tenant`   | FK → Tenant           | Must equal `period.tenant` — enforced in `clean()` |
| `period`   | FK → PeriodIdentifier | `related_name='weeks'`                             |
| `name`     | CharField             | e.g. `Week 1`, `Intersemester Week`                |
| `sequence` | PositiveIntegerField  | Managed — use `reorder()` / `append()`             |

**Constraints:** `unique_together = (name, period)`,
`UniqueConstraint(period, sequence)` — enforces no two weeks share a
position within the same period

**Ordering:** `sequence ASC`

**Sequencing methods:**

```python
# Append a new unit at the end of this period
PeriodWeekIdentifier.append(period, name="Week 12")

# Reassign positions 1..N within a period
PeriodWeekIdentifier.reorder(period, ordered_ids=[5, 1, 2, 3, 4])
```

**Sequencing implementation note:**

Both models use a two-pass bulk update inside `transaction.atomic()` to
avoid transient constraint violations when renumbering. Pass 1 shifts
all existing positions to a high offset (`current + 1000`); pass 2
assigns the final 1..N values. Direct assignment to `sequence` outside
these methods is not safe and should be avoided.

Note `scope_field` below is pseudocode.

```python
@classmethod
def reorder(cls, scope, ordered_ids: list[int]) -> None:
    # `scope` is `tenant` for PeriodIdentifier,
    #            `period`  for PeriodWeekIdentifier
    with transaction.atomic():
        qs = cls.objects.filter(**{scope_field: scope})
        qs.update(sequence=models.F('sequence') + len(ordered_ids) + 1000)
        for position, pk in enumerate(ordered_ids, start=1):
            cls.objects.filter(pk=pk).update(sequence=position)
```

---

### `FHEQLevel`

Controlled vocabulary for FHEQ levels. Eight records (Entry through
Level 8), populated by a data migration at deployment. Not
user-editable through the admin in normal operation. Descriptor text
is stored separately as `RegulatoryDocument` records of type
`fheq_descriptor`, so that descriptor revisions are versioned without
modifying this table.

| Field   | Type                        | Notes                                        |
|---------|-----------------------------|----------------------------------------------|
| `id`    | PK                          |                                              |
| `code`  | CharField unique            | Stable identifier, e.g. `level_4`, `level_7` |
| `level` | PositiveIntegerField unique | Numeric level — Entry plus 3–8               |
| `name`  | CharField                   | Display name, e.g. `Level 6 (Bachelor's)`    |

**Note:** `FHEQLevel` is not tenant-scoped — levels are global and
shared across all tenants. No `TenantScopedManager` is needed.

---

### `RegulatoryDocument`

Reference library entry for external regulatory documents. Covers FHEQ
level descriptors, Subject Benchmark Statements, PSRB requirements, and
OfS conditions. Versioned — when a document is revised, a new record is
created and the previous record's `effective_to` is set in the same
transaction. Tenant-scoped documents (e.g. PSRB requirements specific
to a course) carry a non-null `tenant` FK; global documents (e.g.
FHEQ descriptors, Subject Benchmark Statements) have `tenant` null.

| Field            | Type                      | Notes                                                                       |
|------------------|---------------------------|-----------------------------------------------------------------------------|
| `id`             | PK                        |                                                                             |
| `tenant`         | FK → Tenant (nullable)    | NULL = global; non-null = tenant-specific                                   |
| `document_type`  | CharField                 | `fheq_descriptor`, `subject_benchmark`, `psrb_requirement`, `ofs_condition` |
| `fheq_level`     | FK → FHEQLevel (nullable) | Set for `fheq_descriptor` records only                                      |
| `code`           | CharField                 | Stable identifier across versions, e.g. `sbs_computing`                     |
| `title`          | CharField                 | Display title                                                               |
| `issuing_body`   | CharField                 | e.g. `QAA`, `OfS`                                                           |
| `version_label`  | CharField                 | Human-readable version, e.g. `2023 edition`                                 |
| `effective_from` | DateField                 |                                                                             |
| `effective_to`   | DateField (nullable)      | NULL = currently in force                                                   |
| `content`        | JSONField                 | Structured content for LLM input and UI display                             |
| `created_at`     | DateTimeField             | auto                                                                        |

**Constraints:** `unique_together = (code, effective_from)`

**Current version query:**

```python
RegulatoryDocument.objects.filter(
    code=code,
    effective_from__lte=date,
).filter(
    Q(effective_to__isnull=True) | Q(effective_to__gt=date)
)
```

**Note:** This versioning pattern mirrors `AssetVersion` —
`effective_to` is set when a new version supersedes the current one,
within the same database transaction. Global records are managed by
deployment; tenant-scoped records may be managed by tenant
administrators.

**Caution:** `RegulatoryDocument.tenant` is nullable, which means global
records (tenant null) will be excluded by a standard `TenantScopedManager`
filter. Queries against this model will typically need to combine
tenant-scoped records with global ones:

```python
RegulatoryDocument.objects.filter(
    Q(tenant=tenant) | Q(tenant__isnull=True)
)
```

---

### `AwardType`

Global registry of award types with their credit requirements. Not
tenant-scoped — values are framework-level constants shared across all
tenants, populated by fixture at deployment and updated only when the
Sussex Academic Framework is revised. `CourseIdentifier` carries a FK
to this table.

| Field                  | Type                            | Notes                                                                             |
|------------------------|---------------------------------|-----------------------------------------------------------------------------------|
| `id`                   | PK                              |                                                                                   |
| `code`                 | CharField unique                | Stable identifier, e.g. `bsc_hons`, `msc`, `pgdip`                                |
| `name`                 | CharField unique                | e.g. `Bachelor's Degree with Honours`                                             |
| `abbreviation`         | CharField                       | e.g. `BA`, `BSc`, `MEng`                                                          |
| `fheq_level`           | FK → FHEQLevel                  | Level of the award exit point                                                     |
| `min_credits`          | PositiveIntegerField            | Total credits required for the award                                              |
| `min_credits_at_level` | PositiveIntegerField (nullable) | Minimum credits that must be at the award's FHEQ level. NULL for research degrees |
| `has_component_rules`  | BooleanField                    | True if per-component credit rules apply (e.g. major/minor, joint degrees)        |

**Note:** `AwardType` is not tenant-scoped. No `TenantScopedManager` is
needed. Records are managed by deployment fixture, not by tenant
administrators.

**Note on component rules:** where `has_component_rules` is True, the
per-component credit minima (e.g. major ≥ 60 credits at Level 6, minor
≥ 30 credits at Level 6) are held in a related `AwardTypeComponentRule`
table. That model is **deferred** — stub only, pending implementation of
degree structure validation.

**Fixture values (Schedule 1):**

| code                   | name                                         | abbrev               | FHEQ | min_credits | min_at_level | component_rules |
|------------------------|----------------------------------------------|----------------------|------|-------------|--------------|-----------------|
| `bsc_hons`             | Bachelor's Degree with Honours               | BA/BSc/BEng etc.     | 6    | 360         | 90           | False           |
| `bsc_hons_major_minor` | Bachelor's Degree with Honours (Major/Minor) | BA/BSc etc.          | 6    | 360         | 60+30        | True            |
| `bsc_hons_joint`       | Bachelor's Degree with Honours (Joint)       | BA/BSc etc.          | 6    | 360         | 90           | True            |
| `meng`                 | Integrated Master's Degree                   | MEng/MChem etc.      | 7    | 480         | 120          | False           |
| `msc`                  | Master's Degree                              | MA/MSc/MBA/LLM etc.  | 7    | 180         | 150          | False           |
| `mres`                 | Master of Research                           | MRes                 | 7    | 180         | 150          | False           |
| `msc_by_research`      | Master's Degree by Research                  | LLM by Research etc. | 7    | 180         | 180          | False           |
| `pgdip`                | Postgraduate Diploma                         | PgDip                | 7    | 120         | 90           | False           |
| `pgcert`               | Postgraduate Certificate                     | PgCert               | 7    | 60          | 45           | False           |
| `grad_dip`             | Graduate Diploma                             | GradDip              | 6    | 120         | 120          | False           |
| `grad_cert`            | Graduate Certificate                         | GradCert             | 6    | 60          | 60           | False           |

---

### `CourseIdentifier`

| Field                           | Type                      | Notes                                                                                                                                                                                                                         |
|---------------------------------|---------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `id`                            | PK                        |                                                                                                                                                                                                                               |
| `tenant`                        | FK → Tenant               | Must equal `department.tenant` — enforced in `clean()`                                                                                                                                                                        |
| `department`                    | FK → DepartmentIdentifier | `related_name='courses'`                                                                                                                                                                                                      | 
| `code`                          | CharField                 | unique code, matches institutional code, e.g. F3058U                                                                                                                                                                          |
| `name`                          | CharField                 | course name                                                                                                                                                                                                                   |
| `moa`                           | CharField                 | `TextChoices`: 'FT', 'PT', 'Mixed'                                                                                                                                                                                            |
| `degree_structure`              | CharField                 | `TextChoices`: `single_honours`, `joint_honours`, `major_minor`                                                                                                                                                               |
| `award_type`                    | FK → AwardType            | `related_name='courses'`                                                                                                                                                                                                      |                                                                                                                                                                                                     
| `last_published_at`             | DateTimeField (nullable)  | Most recent publication of any governed document in this course's scope, including module-level events for modules belonging to this course                                                                                   |
| `last_published_document_type`  | CharField (nullable)      | `Document.DocumentType` value of that event                                                                                                                                                                                   |
| `has_in_flight_workflow`        | BooleanField              | True if any governed document anchored to this identifier has an open workflow. Default `False`. Maintained by `on_workflow_opened` and `on_workflow_closed` — not by `on_document_published`. See implementation note below. |
| `course_spec_last_published_at` | DateTimeField (nullable)  | Set only when `CourseSpecificationContent` publishes. Never updated by module-level events. Allows course coordinators to distinguish "something in the course changed" from "the course specification itself changed."       |

**Constraints:** `unique_together = (code, tenant)`

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

**Integrity note:** `department.tenant` must equal `tenant` — enforced in `clean()`.

**Implementation note:**

- The fields `last_published_at` and `last_published_document_type` are maintained by the FSM publication hook
  (`on_document_published` in `workflows/hooks.py`) and must never be written directly outside that hook. See §4 for
  the full update protocol and concurrency discipline.
- `has_in_flight_workflow`: This field has a different update trigger from last_published_at and
  last_published_document_type, which are updated only on publication. `has_in_flight_workflow` is updated by two
  distinct FSM hooks:
    - `on_workflow_opened(document)` — fired when a new `WorkflowInstance` is created for any governed document anchored
      to this identifier. Sets `has_in_flight_workflow = True` unconditionally.
    - `on_workflow_closed(document)` — fired when any `WorkflowInstance` for an anchored document reaches a terminal
      state (`approved`, `withdrawn`, `rejected`, or any other configured terminal state). **Does not set
      `has_in_flight_workflow = False` directly**. Instead, recomputes by checking whether any other anchored document
      still has an open workflow, and sets the field accordingly.

In `on_workflow_closed(document)`, the recomputation on close is necessary because other documents for the same module
may still have open workflows. Setting False blindly on close would produce incorrect results when two workflows close
concurrently or in rapid succession. Recomputation query:

```python
has_open = Document.objects.filter(
    module_identifier=module,
).exclude(
    current_workflow__isnull=True,
).filter(
    current_workflow__state__in=NON_TERMINAL_STATES,
).exists()
```

Both hooks are subject to the same `select_for_update()` discipline as `last_published_at`. See §4 in`DESIGN.md` for the
concurrency rules.

---

### `ModuleIdentifier`

| Field                          | Type                      | Notes                                                                                                                                                                                                                         |
|--------------------------------|---------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `id`                           | PK                        |                                                                                                                                                                                                                               |
| `tenant`                       | FK → Tenant               | Must equal `department.tenant` — enforced in `clean()`                                                                                                                                                                        |
| `department`                   | FK → DepartmentIdentifier |                                                                                                                                                                                                                               |
| `code`                         | CharField                 | unique code, matches institutional code, e.g. F3202                                                                                                                                                                           |
| `name`                         | CharField                 | module name (mutable)                                                                                                                                                                                                         |
| `level`                        | FK → FHEQLevel            | FHEQ level descriptor                                                                                                                                                                                                         |
| `is_elective`                  | BooleanField              | Elective is a property of the module. Elective modules go in protected timetabling periods. Core vs Option is a property of the `CourseModuleMembership`.                                                                     |
| `last_published_at`            | DateTimeField (nullable)  | Timestamp of the most recent publication of any governed document anchored to this module                                                                                                                                     |
| `last_published_document_type` | CharField (nullable)      | `Document.DocumentType` value of that publication event                                                                                                                                                                       |
| `has_in_flight_workflow`       | BooleanField              | True if any governed document anchored to this identifier has an open workflow. Default `False`. Maintained by `on_workflow_opened` and `on_workflow_closed` — not by `on_document_published`. See implementation note below. |

**Constraints:** `unique_together = (code, tenant)`. Elective modules with `is_elective` set to `True`
must have `credit_points == 15`, enforced by `ModuleSpecificationContent.clean()`.

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

**Integrity note:** `department.tenant` must equal `tenant` — enforced in `clean()`.

**Implementation note:**

- The fields `last_published_at` and `last_published_document_type` are maintained by the FSM publication hook
  (`on_document_published` in `workflows/hooks.py`) and must never be written directly outside that hook. See §4 for
  the full update protocol and concurrency discipline.
- `has_in_flight_workflow` is maintained by `on_workflow_opened` and `on_workflow_closed`. See the identical
  implementation note on CourseIdentifier for full details, including the recomputation query and concurrency
  discipline.

---

### `CourseModuleMembership`

Join table expressing the many-to-many relationship between courses
and modules. A module may belong to multiple courses; a course
contains multiple modules. This table is the anchor for discovery
queries ("all modules on this course") and for course-level
denormalised field propagation (`last_published_at` on
`CourseIdentifier`).

| Field       | Type                  | Notes                                                                                                                                                                                                                                                                                                                                    |
|-------------|-----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `id`        | PK                    |                                                                                                                                                                                                                                                                                                                                          |
| `tenant`    | FK → Tenant           | Must equal both `course.tenant` and `module.tenant` — enforced in `clean()`                                                                                                                                                                                                                                                              |
| `course`    | FK → CourseIdentifier | `related_name='module_memberships'`                                                                                                                                                                                                                                                                                                      |
| `module`    | FK → ModuleIdentifier | `related_name='course_memberships'`                                                                                                                                                                                                                                                                                                      |
| `stage`     | CharField             | `TextChoices`: `foundation`, `certificate`, `diploma`, `honours`. Stage of study. Note this is distinct from FHEQ level; a language module at a lower FHEQ may still be taught in stage 2. Note there is a well-defined canonical ordering `foundation` < `certificate` < `diploma` < `honours`. Does not apply to postgraduate courses. |
| `component` | CharField             | `TextChoices`: `core`, `option`, `major`, `minor`. Role of this module within the course.                                                                                                                                                                                                                                                |                                                                                             

**Constraints:** `unique_together = (course, module)`

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

**Query patterns:**

```python
# All modules on a course
ModuleIdentifier.objects.filter(
    course_memberships__course=course
)

# All courses a module belongs to
CourseIdentifier.objects.filter(
    module_memberships__module=module
)
```

---

### `AssessmentSlotIdentifier`

A typed registry entry for each assessment within a module. Created when
a module is established. Retired rather than deleted when an assessment
is removed, so that historical assessment documents retain a valid
reference.

| Field             | Type                  | Notes                                                                                                                  |
|-------------------|-----------------------|------------------------------------------------------------------------------------------------------------------------|
| `id`              | PK                    |                                                                                                                        |
| `tenant`          | FK → Tenant           | Must equal `module.tenant` — enforced in `clean()`                                                                     |
| `module`          | FK → ModuleIdentifier | `related_name='assessment_slots'`                                                                                      |
| `assessment_type` | CharField             | `exam`, `cwk`, `prb` — controlled vocabulary, see note                                                                 |
| `sequence`        | PositiveIntegerField  | Position within type for this module, e.g. `1`, `2`                                                                    |
| `is_examination`  | BooleanField          | `True` if the assessment is delivered in an examination room under invigilation (academic framework §12.5 distinction) |
| `retired_at`      | DateField (nullable)  | NULL = active; set when slot is retired                                                                                |

**Constraints:** `unique_together = (module, assessment_type, sequence)`

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

**Derived property:**

```python
@property
def code(self) -> str:
    return f"{self.assessment_type}-{self.sequence:02d}"
    # → 'cwk-01', 'exam-02'
```

**Note:** a retired slot occupies its sequence position permanently, e.g., if
`cwk-01` is retired, then it cannot be re-used; a new `cwk` entry would become
`cwk-02`. Codes on historical documents will be stable and non-ambiguous.

**Active slots query:**

```python
ModuleIdentifier.assessment_slots.filter(retired_at__isnull=True)
```

**Note:** `assessment_type` is a `TextChoices` enum defined in code,
not a free CharField, to prevent inconsistent codes across modules. The
controlled vocabulary (`exam`, `cwk`, `prb`, etc.) is defined at the
application level and does not require a database table.

---

### `PathwayIdentifier`

A validated, academically coherent combination of modules that students
may select on arrival. Pathways are approved curriculum objects — not
informal groupings. Append-only: a pathway is closed by setting
`valid_to` rather than deleting the record.

Elective modules (`is_elective=True`) may appear as pathway members via
`PathwayModuleMembership`. This is the only join table through which
electives are associated with a course-level structure; electives do not
appear in `CourseModuleMembership`.

| Field           | Type                      | Notes                                              |
|-----------------|---------------------------|----------------------------------------------------|
| `id`            | PK                        |                                                    |
| `tenant`        | FK → Tenant               | Must equal `course.tenant` — enforced in `clean()` |
| `course`        | FK → CourseIdentifier     | `related_name='pathways'`                          |
| `name`          | CharField                 | e.g. `Language Pathway (French)`                   |
| `credit_volume` | PositiveSmallIntegerField | Typically 60 or 90                                 |
| `valid_from`    | DateField                 |                                                    |
| `valid_to`      | DateField (nullable)      | NULL = currently approved                          |

**Constraints:** `unique_together = (course, name)`

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

**Integrity note:** `tenant` must equal `course.tenant` — enforced in
`clean()`.

**Current pathways query:**

```python
PathwayIdentifier.objects.for_tenant(tenant).filter(
    course=course,
    valid_to__isnull=True,
)
```

---

### `PathwayModuleMembership`

Join table expressing which modules constitute a pathway. A module may
appear in multiple pathways. Elective modules (`is_elective=True`) are
permitted members; non-elective modules are also permitted where the
pathway is composed of core or option modules.

| Field     | Type                   | Notes                                                |
|-----------|------------------------|------------------------------------------------------|
| `id`      | PK                     |                                                      |
| `tenant`  | FK → Tenant            | Must equal both `pathway.tenant` and `module.tenant` |
| `pathway` | FK → PathwayIdentifier | `related_name='module_memberships'`                  |
| `module`  | FK → ModuleIdentifier  | `related_name='pathway_memberships'`                 |

**Constraints:** `unique_together = (pathway, module)`

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

**Integrity note:** `tenant` must equal `pathway.tenant` and
`module.tenant` — enforced in `clean()`.

---

### `FacultyAssessmentLoadPolicy`

Stores faculty-level assessment load equivalency bands, keyed by FHEQ
level and credit value. Defines the expected range of notional hours
attributable to assessment for a module of a given credit value at a
given level, consistent with the 1 credit = 10 notional learning hours
convention. Also carries the cohort-level weekly threshold used by the
deadline congestion checker.

One policy record per (faculty, fheq_level, credit_value) combination.
Records are managed by faculty administrators, not by deployment fixture,
since norms vary by faculty.

| Field                    | Type                      | Notes                                                                                       |
|--------------------------|---------------------------|---------------------------------------------------------------------------------------------|
| `id`                     | PK                        |                                                                                             |
| `tenant`                 | FK → Tenant               | Must equal `faculty.tenant` — enforced in `clean()`                                         |
| `faculty`                | FK → FacultyIdentifier    | `related_name='assessment_load_policies'`                                                   |
| `fheq_level`             | FK → FHEQLevel            |                                                                                             |
| `credit_value`           | PositiveSmallIntegerField | e.g. `15`, `30`                                                                             |
| `min_assessment_hours`   | PositiveSmallIntegerField | Lower bound of expected assessment effort in notional hours                                 |
| `max_assessment_hours`   | PositiveSmallIntegerField | Upper bound of expected assessment effort in notional hours                                 |
| `weekly_hours_threshold` | PositiveSmallIntegerField | Maximum acceptable cohort assessment hours in any single week before congestion flag raised |
| `terminal_exam_hours`    | PositiveSmallIntegerField | Assumed notional hours for any terminal examination (revision + sitting). Default `30`      |

**Constraints:** `unique_together = (faculty, fheq_level, credit_value)`

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

**Integrity note:** `tenant` must equal `faculty.tenant` — enforced in
`clean()`.

**Load check query:**

```python
# Retrieve the applicable policy for a given module
policy = FacultyAssessmentLoadPolicy.objects.get(
    faculty=module.department.faculty,
    fheq_level=module.level,
    credit_value=module_spec.credit_points,
)

# Sum estimated assessment hours across active slots
from django.db.models import Sum

total = AssessmentContent.objects.filter(
    module=module,
    slot__retired_at__isnull=True,
).aggregate(total=Sum('estimated_hours'))['total'] or 0

# Flag if outside band
if total < policy.min_assessment_hours or total > policy.max_assessment_hours:
# raise validation warning
```

**Note:** `estimated_hours` on `AssessmentContent` is nullable. The load
check should degrade gracefully when not all assessments have been
annotated — flagging incomplete data rather than treating missing values
as zero.

---

### `CommitteeIdentifier`

A governance committee. Referenced by `CommitteeMeeting` and by FSM
`Transition` definitions via `slug`. Committees sit at faculty, school,
department, or tenant level — exactly one of `faculty`, `school`, or
`is_tenant_level` is set, enforced in `clean()`.

| Field             | Type                                 | Notes                                                                              |
|-------------------|--------------------------------------|------------------------------------------------------------------------------------|
| `id`              | PK                                   |                                                                                    |
| `tenant`          | FK → Tenant                          |                                                                                    |
| `slug`            | SlugField                            | Stable identifier used in FSM transition definitions, e.g. `teaching_and_learning` |
| `name`            | CharField                            | Display name                                                                       |
| `faculty`         | FK → FacultyIdentifier (nullable)    | Set if committee is faculty-level                                                  |
| `school`          | FK → SchoolIdentifier (nullable)     | Set if committee is school-level                                                   |
| `department`      | FK → DepartmentIdentifier (nullable) | Set if committee is department-level                                               |
| `is_tenant_level` | BooleanField                         | True if committee operates across the whole tenant                                 |

**Constraints:** `unique_together = (slug, tenant)`

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

**Integrity note:** Exactly one of `faculty`, `school`, `department`, or
`is_tenant_level` must be set — enforced in `clean()`.

---

### `AccreditingBodyIdentifier`

A professional or statutory accrediting body. Global — not
tenant-scoped. Course-level accreditation is expressed via
`CourseAccreditationScope`.

| Field          | Type                | Notes                       |
|----------------|---------------------|-----------------------------|
| `id`           | PK                  |                             |
| `name`         | CharField unique    | e.g. `Institute of Physics` |
| `abbreviation` | CharField           | e.g. `IoP`                  |
| `url`          | URLField (nullable) |                             |

---

## Scoping layer

### `CourseConvenorScope`

| Field        | Type                  | Notes                   |
|--------------|-----------------------|-------------------------|
| `id`         | PK                    |                         |
| `tenant`     | FK → Tenant           |
| `course`     | FK → CourseIdentifier |                         |
| `user`       | FK → User             |                         |
| `valid_from` | DateField             |                         |
| `valid_to`   | DateField (nullable)  | NULL = currently active |

**Constraints:**

```python
UniqueConstraint(
    fields=['course', 'user'],
    condition=Q(valid_to__isnull=True),
    name='unique_active_course_convenor',
)
```

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

**Integrity note:** `tenant` must equal `course.tenant` – enforced in `clean()`.

**Implementation note:** `CourseConvenorScope` records should not be created directly. Use
`CourseConvenorScope.create_for_user(course, user, valid_from)`, which atomically creates the scope record and
ensures the user has a convenor `TenantMembership` for the tenant, creating one if absent. The `clean()` method
validates that the membership exists but cannot enforce the atomic creation.

---

### `ModuleConvenorScope`

| Field        | Type                  | Notes                   |
|--------------|-----------------------|-------------------------|
| `id`         | PK                    |                         |
| `tenant`     | FK → Tenant           |
| `module`     | FK → ModuleIdentifier |                         |
| `user`       | FK → User             |                         |
| `valid_from` | DateField             |                         |
| `valid_to`   | DateField (nullable)  | NULL = currently active |

**Constraints:**

```python
UniqueConstraint(
    fields=['module', 'user'],
    condition=Q(valid_to__isnull=True),
    name='unique_active_module_convenor',
)
```

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

**Integrity note:** `tenant` must equal `module.tenant` – enforced in `clean()`.

**Implementation note:** `ModuleConvenorScope` records should not be created directly. Use
`ModuleConvenorScope.create_for_user(module, user, valid_from)`, which atomically creates the scope record and
ensures the user has a convenor `TenantMembership` for the tenant, creating one if absent. The `clean()` method
validates that the membership exists but cannot enforce the atomic creation.

---

### `CourseAccreditationScope`

Records the accreditation of a course by an accrediting body for a
defined period. Append-only — accreditation is closed by setting
`valid_to` rather than deleting the record. PSRB-specific
`RegulatoryDocument` records applicable to this accreditation are
linked via the M2M.

| Field                  | Type                           | Notes                                              |
|------------------------|--------------------------------|----------------------------------------------------|
| `id`                   | PK                             |                                                    |
| `tenant`               | FK → Tenant                    | Must equal `course.tenant` — enforced in `clean()` |
| `course`               | FK → CourseIdentifier          |                                                    |
| `accrediting_body`     | FK → AccreditingBodyIdentifier |                                                    |
| `valid_from`           | DateField                      |                                                    |
| `valid_to`             | DateField (nullable)           | NULL = currently accredited                        |
| `regulatory_documents` | M2M → RegulatoryDocument       | PSRB requirements applicable to this accreditation |
| `notes`                | TextField (blank)              |                                                    |

**Constraints:**

```python
UniqueConstraint(
    fields=['course', 'accrediting_body'],
    condition=Q(valid_to__isnull=True),
    name='unique_active_course_accreditation',
)
```

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

**Current accreditations query:**

```python
CourseAccreditationScope.objects.for_tenant(tenant).filter(
    course=course,
    valid_to__isnull=True,
)
```

---

## Document layer

### `Document` (envelope)

Shared envelope for all document and form types. All cross-cutting
queries operate on this table. Type-specific content lives in a
separate table linked by `OneToOneField`.

| Field              | Type                                        | Notes                                                                                                                                                                                                           |
|--------------------|---------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `id`               | PK                                          |                                                                                                                                                                                                                 |
| `tenant`           | FK → Tenant                                 |                                                                                                                                                                                                                 |
| `document_type`    | CharField                                   | `DocumentType` choices — see registry                                                                                                                                                                           |
| `classification`   | CharField                                   | `TextChoices`: `published_asset`, `workflow_form`, `internal`                                                                                                                                                   |
| `parent`           | FK → Document (nullable)                    | Self-referential ownership hierarchy                                                                                                                                                                            |
| `title`            | CharField                                   |                                                                                                                                                                                                                 |
| `reference_code`   | CharField                                   | Unique per tenant                                                                                                                                                                                               |
| `change_rationale` | TextField (nullable, blank)                 | Formal governance justification for the change. Populated at workflow initiation. Nullable because not all document workflows are change-driven (e.g. initial CLO introduction at programme inception). See D2. |
| `due_date`         | DateTimeField (nullable)                    | Updated by scheduler                                                                                                                                                                                            |
| `current_workflow` | OneToOneField → WorkflowInstance (nullable) | Denormalised for fast reads                                                                                                                                                                                     |
| `current_version`  | OneToOneField → AssetVersion (nullable)     | Denormalised for fast reads                                                                                                                                                                                     |
| `created_at`       | DateTimeField                               | auto                                                                                                                                                                                                            |
| `updated_at`       | DateTimeField                               | auto                                                                                                                                                                                                            |

**Constraints:** `unique_together = (tenant, reference_code)`

**Indexes:** `(tenant, document_type)`, `(tenant, due_date)`, `(parent,)`

**Managers:** `objects = TenantScopedManager()`, `all_tenants = TenantScopedManager(require_tenant=False)`

---

### `Document.DocumentType` values

| Value                   | Content model                | Notes                                                                                                                                                              |
|-------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `submission_cover_note` | `SubmissionCoverNoteContent` | `classification=INTERNAL`. Coordinator-authored covering document for committee submissions. Scheduler-invisible — `target_document_type` never equals this value. |
| `module_spec`           | `ModuleSpecificationContent` |                                                                                                                                                                    |
| `course_spec`           | `CourseSpecificationContent` |                                                                                                                                                                    |
| `assessment`            | `AssessmentContent`          | Covers all assessment formats (exam, coursework, problem set, etc.)                                                                                                |
| `course_outcome`        | `CourseOutcomeContent`       |                                                                                                                                                                    |
| `module_outcome`        | `ModuleOutcomeContent`       |                                                                                                                                                                    |
| `teaching_activity`     | `TeachingActivityContent`    | Multiple may be approved simultaneously per module                                                                                                                 |
| `reaccreditation`       | `ReaccreditationContent`     |                                                                                                                                                                    |
| `module_requisite`      | `ModuleRequisiteContent`     | **Deferred** — governance model TBD with curriculum team                                                                                                           |
| `curriculum_review`     | `CurriculumReviewContent`    | **Deferred** — follows same pattern as `reaccreditation`                                                                                                           |

---

### `[Type]Content` (one table per document type)

Each document type has exactly one content model. The pattern is
always a `OneToOneField(Document, related_name='[type]')`.
Content models carry only type-specific fields.

Please read `document_models.md` for detailed descriptions of the
content models.

---

## Workflow layer

### `WorkflowInstance`

Current FSM state of a document. One instance per document.
Uses `GenericForeignKey` so it can reference any document type
without requiring per-type FK columns.

| Field              | Type                 | Notes                                                                                                                                                                                  |
|--------------------|----------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `id`               | PK                   |                                                                                                                                                                                        |
| `content_type`     | FK → ContentType     | For GenericForeignKey                                                                                                                                                                  |
| `object_id`        | PositiveIntegerField | For GenericForeignKey                                                                                                                                                                  |
| `form`             | GenericForeignKey    | Points to the document                                                                                                                                                                 |
| `workflow_type`    | CharField            | e.g. `module_specification`                                                                                                                                                            |
| `state`            | CharField            | Stores `enum.name`, e.g. `FACULTY_REVIEW`                                                                                                                                              |
| `originating_task` | FK → Task (nullable) | Set when this `WorkflowInstance` was created in response to a convenor nomination under a dispatched `Task`. NULL for all self-initiated workflows. `related_name='spawned_instances'` |
| `created_at`       | DateTimeField        | auto                                                                                                                                                                                   |
| `updated_at`       | DateTimeField        | auto                                                                                                                                                                                   |

**Indexes:** `(content_type, object_id)`, `(workflow_type, state)`

---

### `WorkflowEvent` *(append-only)*

Full audit trail of every FSM transition. Never updated or deleted.

| Field               | Type                  | Notes                                 |
|---------------------|-----------------------|---------------------------------------|
| `id`                | PK                    |                                       |
| `workflow_instance` | FK → WorkflowInstance | `related_name='events'`               |
| `event`             | CharField             | Transition event name, e.g. `approve` |
| `from_state`        | CharField             | State name before transition          |
| `to_state`          | CharField             | State name after transition           |
| `actor`             | FK → User             | Who performed the transition          |
| `actor_role`        | CharField             | Role at time of transition            |
| `timestamp`         | DateTimeField         | auto                                  |
| `metadata`          | JSONField             | Comments, context, rejection reasons  |

**Ordering:** `timestamp ASC`

**Indexes:** `(workflow_instance, timestamp)`

---

### `Task`

A pending action for a role pool or individual. Created and superseded by FSM
transitions. One task per pool — not one per user.

**Model ordering note:** `Task` must be defined before `ScheduledMilestone` in
`models.py`. `Task.scheduled_milestone` uses a string forward reference
`'ScheduledMilestone'`; `ScheduledMilestone.originating_task` may reference
`Task` directly.

| Field                 | Type                                   | Notes                                                                                                                                     |
|-----------------------|----------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| `id`                  | PK                                     |                                                                                                                                           |
| `workflow_instance`   | FK → WorkflowInstance                  | `related_name='tasks'`                                                                                                                    |
| `event`               | CharField                              | The FSM event this task enables                                                                                                           |
| `assignee_pool`       | CharField                              | Role name — who can act                                                                                                                   |
| `claimed_by`          | FK → User (nullable)                   | Set atomically on claim                                                                                                                   |
| `claimed_at`          | DateTimeField (nullable)               |                                                                                                                                           |
| `status`              | CharField (TextChoices)                | `PENDING \| CLAIMED \| IN_PROGRESS \| COMPLETED \| ABANDONED \| SUPERSEDED`                                                               |
| `scheduled_milestone` | FK → `'ScheduledMilestone'` (nullable) | Set when this task is dispatched against a pre-existing milestone (CLO-driven orchestrated workflows). NULL for ordinary FSM-gated tasks. |
| `context_note`        | TextField (nullable, blank)            | Coordinator's module-specific guidance to the convenor. Populated as a Stage 3 side effect; not authored at task creation.                |
| `priority_score`      | IntegerField                           | Computed at write time                                                                                                                    |
| `due_date`            | DateTimeField (nullable)               | Set and updated by scheduler                                                                                                              |
| `created_at`          | DateTimeField                          | auto                                                                                                                                      |

**Status vocabulary:**

| Value         | Meaning                                                                                                                                                |
|---------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| `PENDING`     | Dispatched, not yet claimed                                                                                                                            |
| `CLAIMED`     | Claimed by a user; no document nominated yet                                                                                                           |
| `IN_PROGRESS` | Convenor has nominated at least one document; one or more child milestones exist                                                                       |
| `COMPLETED`   | All obligations under this task are resolved                                                                                                           |
| `ABANDONED`   | Explicitly cancelled by the coordinator or administrator, e.g. module removed from scope. Independent of the state of the driving `PublicationTarget`. |
| `SUPERSEDED`  | Closed by an FSM transition; no longer the relevant work item. Retained for audit.                                                                     |

**Indexes:** `(status, assignee_pool, priority_score)`

**Claim guard (atomic):**

```sql
UPDATE tasks
SET claimed_by = $user,
    status     = 'CLAIMED'
WHERE id = $id
  AND claimed_by IS NULL
-- Zero rows affected → another user claimed first
```

**`clean()` rules:**

- If `scheduled_milestone` is set, `scheduled_milestone.document` must be NULL.
  A task cannot be dispatched against an already-live milestone.
- `claimed_by` and `claimed_at` must both be set or both NULL.

---

### `ScheduledMilestone`

One record per document required by a publication target. Carries the computed
deadline, initiation date, status, and uncertainty envelope. Rebuilt atomically
when any scheduling input changes, except for terminal-state records which are
retained for audit.

**Model ordering note:** `ScheduledMilestone` must be defined after `Task` in
`models.py`. `originating_task` may reference `Task` directly.

| Field                    | Type                     | Notes                                                                                                                                                      |
|--------------------------|--------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `id`                     | PK                       |                                                                                                                                                            |
| `publication_target`     | FK → PublicationTarget   | Originating target. `related_name='milestones'`                                                                                                            |
| `document`               | FK → Document (nullable) | NULL if workflow not yet initiated                                                                                                                         |
| `document_type`          | CharField                | Required document type                                                                                                                                     |
| `required_state`         | CharField                | State the document must reach                                                                                                                              |
| `status`                 | CharField (TextChoices)  | `ACTIVE \| AT_RISK \| OVERDUE \| COMPLETED \| ABANDONED \| SUPERSEDED`                                                                                     |
| `originating_task`       | FK → Task (nullable)     | The task whose nomination created this milestone. NULL for scheduler-generated milestones not driven by a convenor task. `related_name='child_milestones'` |
| `must_be_complete_by`    | DateField                | Computed point estimate — minimum across all linked targets                                                                                                |
| `earliest_must_complete` | DateField (nullable)     | Computed from earliest plausible target date. NULL when `date_confidence = CONFIRMED`.                                                                     |
| `latest_must_complete`   | DateField (nullable)     | Computed from latest plausible target date. NULL when `date_confidence = CONFIRMED`.                                                                       |
| `must_be_initiated_by`   | DateField                | Computed point estimate by backwards scheduler                                                                                                             |
| `earliest_must_initiate` | DateField (nullable)     | Computed from earliest plausible target date. NULL when `date_confidence = CONFIRMED`.                                                                     |
| `latest_must_initiate`   | DateField (nullable)     | Computed from latest plausible target date. NULL when `date_confidence = CONFIRMED`.                                                                       |
| `date_confidence`        | CharField (TextChoices)  | `WINDOW \| PROVISIONAL \| INDICATIVE \| CONFIRMED`. MIN across all linked targets.                                                                         |
| `is_critical_path`       | BooleanField             |                                                                                                                                                            |
| `computed_at`            | DateTimeField            | auto — when the scheduler last ran                                                                                                                         |

**Status vocabulary:**

| Value        | Meaning                                                                                            |
|--------------|----------------------------------------------------------------------------------------------------|
| `ACTIVE`     | In-flight; scheduler is computing against it; no deadline concern                                  |
| `AT_RISK`    | A deadline concern has been detected and flagged by the scheduler                                  |
| `OVERDUE`    | `must_be_complete_by` is past and `required_state` not yet reached                                 |
| `COMPLETED`  | Document has reached `required_state`                                                              |
| `ABANDONED`  | Module removed from scope; milestone retained for audit. Independent of `PublicationTarget` state. |
| `SUPERSEDED` | Replaced by a new milestone, e.g. due to a rescoping exercise. Retained for audit.                 |

`ACTIVE → AT_RISK → OVERDUE` is a strict severity progression. `COMPLETED`,
`ABANDONED`, and `SUPERSEDED` are terminal states. A milestone may not regress
from a terminal state. Enforced in `clean()`.

**Constraints:** `unique_together = (publication_target, document_type, document)`

**Indexes:** `(publication_target, must_be_complete_by)`, `(document,)`,
`(document_type,)`, `(date_confidence,)`, `(status,)`

**Scheduling note:** `must_be_complete_by` and the uncertainty envelope are
recomputed whenever a `MilestonePublicationTarget` row is added or removed,
or whenever a linked `CommitteeMeeting.date_confidence` or
`CommitteeMeeting.meeting_date` changes. Terminal-state milestones are excluded
from recomputation.

**Envelope semantics:** `earliest_must_complete` and `earliest_must_initiate`
are computed from the earliest plausible target date (tightest deadline);
`latest_must_complete` and `latest_must_initiate` from the latest plausible
target date (most relaxed deadline). Point estimate fields are always computed
against `PublicationTarget.target_date` as the primary anchor.

---

## Asset versioning

### `Asset`

Stable identity of a publishable asset. Persists across versions.
Other models reference this rather than a specific version.

| Field             | Type                                    | Notes                                        |
|-------------------|-----------------------------------------|----------------------------------------------|
| `id`              | PK                                      |                                              |
| `tenant`          | FK → Tenant                             |                                              |
| `asset_type`      | CharField                               |                                              |
| `code`            | CharField                               | Stable external identifier                   |
| `current_version` | OneToOneField → AssetVersion (nullable) | Denormalised pointer; updated on publication |

**Constraints:** `unique_together = (tenant, code)`

---

### `AssetVersion` *(append-only)*

Immutable snapshot of an asset. The `valid_from` / `valid_to`
window supports compliance queries.

| Field            | Type                     | Notes                                                                               |
|------------------|--------------------------|-------------------------------------------------------------------------------------|
| `id`             | PK                       |                                                                                     |
| `asset`          | FK → Asset               | `related_name='versions'`                                                           |
| `version_number` | PositiveIntegerField     |                                                                                     |
| `status`         | CharField                | `TextChoices`: `draft`, `release_candidate`, `published`, `superseded`, `withdrawn` |
| `valid_from`     | DateTimeField (nullable) | Set on publication                                                                  |
| `valid_to`       | DateTimeField (nullable) | Set when superseded; NULL = still in force                                          |
| `content`        | JSONField                | Content snapshot                                                                    |
| `created_at`     | DateTimeField            | auto                                                                                |
| `created_by`     | FK → User                |                                                                                     |

**Constraints:** `unique_together = (asset, version_number)`

**Indexes:** `(asset, status)`, `(asset, valid_from, valid_to)`

**Compliance query:**

```python
AssetVersion.objects.get(
    asset=asset,
    status='published',
    valid_from__lte=date,
).filter(Q(valid_to__isnull=True) | Q(valid_to__gt=date))
```

---

### `WorkflowVersionEvent` *(append-only)*

Provenance join table. Records which specific FSM transition
caused which version action.

| Field               | Type                  | Notes                                                         |
|---------------------|-----------------------|---------------------------------------------------------------|
| `id`                | PK                    |                                                               |
| `workflow_instance` | FK → WorkflowInstance | `related_name='version_events'`                               |
| `workflow_event`    | FK → WorkflowEvent    | Specific transition that caused this action                   |
| `asset_version`     | FK → AssetVersion     | The version acted upon                                        |
| `action`            | CharField             | `created`, `advanced`, `published`, `superseded`, `withdrawn` |
| `timestamp`         | DateTimeField         | auto                                                          |

---

## Scheduling

### `PublicationTarget` (revised)

A desired outcome with a target date. Drives the backwards scheduler.
All scheduled milestones are grouped under this entity.

`target_date` is coordinator-owned and is never written by the scheduler.
Every change to `target_date` is recorded in `PublicationTargetDateChange`.
`schedule_status` is the only field written by the scheduler.

| Field                       | Type                              | Notes                                                                                                                                                                                                                                                                   |
|-----------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `id`                        | PK                                |                                                                                                                                                                                                                                                                         |
| `tenant`                    | FK → Tenant                       |                                                                                                                                                                                                                                                                         |
| `label`                     | CharField                         | e.g. `AY 2029/30 course review`                                                                                                                                                                                                                                         |
| `description`               | TextField                         |                                                                                                                                                                                                                                                                         |
| `target_document_type`      | CharField                         | The terminal document type                                                                                                                                                                                                                                              |
| `target_state`              | CharField                         | The required terminal state                                                                                                                                                                                                                                             |
| `target_date`               | DateField                         | Coordinator-owned. Never written by the scheduler.                                                                                                                                                                                                                      |
| `date_confidence`           | CharField (TextChoices)           | `WINDOW \| PROVISIONAL \| INDICATIVE \| CONFIRMED`. Unidirectional toward CONFIRMED; enforced in `clean()`                                                                                                                                                              |
| `owner`                     | FK → User                         | Usually the course coordinator; could also be TLC chair or a curriculum review/accreditation coordinator                                                                                                                                                                |
| `course`                    | FK → CourseIdentifier (nullable)  | If set, this target belongs to the specified course. Unanchored targets are allowed.                                                                                                                                                                                    |
| `schedule_status`           | CharField (TextChoices)           | `ON_TRACK \| AT_RISK \| CRITICAL \| UNACHIEVABLE \| COMPLETED \| ABANDONED \| SUPERSEDED`. Written by scheduler except `ABANDONED`/`SUPERSEDED`.                                                                                                                        |
| `cover_note`                | FK → Document (nullable)          | `related_name='cover_note_for_target'`. Points to a `Document` of type `submission_cover_note`, `classification=INTERNAL`. NULL until the coordinator creates a cover note. Creation of a cover note document is not required at target creation. `on_delete=SET_NULL`. |
| `scope_anchor`              | CharField (TextChoices, nullable) | `DEPARTMENT \| SCHOOL \| FACULTY \| MANUAL`. NULL for non-impact target types.                                                                                                                                                                                          |
| `scope_anchor_content_type` | FK → ContentType (nullable)       | ContentType of the anchor organisational unit. NULL when `scope_anchor = MANUAL` or NULL.                                                                                                                                                                               |
| `scope_anchor_object_id`    | PositiveIntegerField (nullable)   | PK of the anchor organisational unit. NULL when `scope_anchor = MANUAL` or NULL.                                                                                                                                                                                        |
| `abandoned_rationale`       | TextField (blank)                 | Required when `schedule_status = ABANDONED`. Records the governance justification for closing the target without publication. See S8.                                                                                                                                   |
| `created_at`                | DateTimeField                     | auto                                                                                                                                                                                                                                                                    |

**Integrity note:** `cover_note.tenant` must equal `tenant` when
`cover_note` is non-null — enforced in `clean()`.

**Computed property:**

- `is_active` returns `True` when `schedule_status` is not
in `{COMPLETED, ABANDONED, SUPERSEDED}`. Not a stored field.
- `scope_anchor_object` — GenericForeignKey over
  `scope_anchor_content_type` and `scope_anchor_object_id`.

**Constraints:**

- If `course` is not null, `tenant` must equal `course.tenant`. Enforced in `clean()`.
- `date_confidence` may only transition toward `CONFIRMED`. Enforced in `clean()`.
- `abandoned_rationale` must be non-empty when `schedule_status = ABANDONED`. Enforced in `clean()`.
- `scope_anchor_content_type` and `scope_anchor_object_id` must both be
  set or both NULL.
- When `scope_anchor` is not NULL and not `MANUAL`, both
  `scope_anchor_content_type` and `scope_anchor_object_id` must be set.
- When `scope_anchor = MANUAL` or NULL, both must be NULL.

**Note on reverse navigation:** the `related_name='cover_note_for_target'`
Django reverse relation provides navigation from the cover note `Document`
back to its `PublicationTarget` without requiring a FK on the content
model. No circular dependency in migrations.

**Gate:** attachment of the `PublicationTarget` to a `CommitteeMeeting`
as an `AgendaItem` is conditional on `cover_note` being non-null and
`cover_note.current_workflow.state == SUBMITTED`. Enforced in the
`AgendaItem` creation view and the Stage 8 FSM pre-transition guard.


---

### `PublicationTargetDateChange` *(append-only — new model)*

Audited log of every change to `PublicationTarget.target_date`. One record
per change. Never updated or deleted.

| Field                | Type                              | Notes                                                     |
|----------------------|-----------------------------------|-----------------------------------------------------------|
| `id`                 | PK                                |                                                           |
| `publication_target` | FK → PublicationTarget            | `related_name='date_changes'`                             |
| `previous_date`      | DateField                         |                                                           |
| `new_date`           | DateField                         |                                                           |
| `changed_by`         | FK → User                         |                                                           |
| `changed_at`         | DateTimeField                     | auto                                                      |
| `rationale`          | TextField (blank)                 | Optional — populated when confirming a scheduler advisory |
| `advisory`           | FK → SchedulerAdvisory (nullable) | The advisory that prompted this change, if any            |

**Indexes:** `(publication_target, changed_at)`

---

### `PublicationScopeItem` — model specification

#### Core framing

`PublicationScopeItem` is the coordinator's scoping canvas for a
`PublicationTarget`. It is a first-class model in the scheduling layer,
not a sub-type or output of the LLM layer. One row exists per candidate
module, regardless of whether an LLM has run. The LLM, when enabled,
populates the `coverage_class` and `llm_explanation` fields on existing
rows and sets `populated_by = LLM`. When the LLM is disabled, those
fields remain blank and `populated_by = COORDINATOR`, and the coordinator
makes the scoping decision without automated classification.

This design means the system is fully operable without an LLM. Enabling
the LLM is an enrichment step that assists but does not gate the
coordinator's workflow.

#### LLM-disabled operation

When the LLM feature flag is off:

1. The coordinator opens the scoping canvas. The system queries the
   candidate pool from the `scope_anchor` rule and creates
   `PublicationScopeItem` rows with `populated_by = COORDINATOR`,
   `coverage_class = NULL`, `llm_result = NULL`.
2. The coordinator reviews each row and sets `coordinator_decision`
   directly, optionally adding a `coordinator_note`.
3. The coordinator may add rows for modules outside the candidate pool
   using the same add-module action as in the LLM-enabled path.
4. Once all rows are decided, the D8 gate is satisfied and the
   backwards scheduler runs.

No `LLMResult` record is created. No fake records of any kind are
required.

#### LLM-enabled operation

When the LLM feature flag is on:

1. The coordinator opens the scoping canvas. The system creates
   `PublicationScopeItem` rows as above (`populated_by = COORDINATOR`,
   `coverage_class = NULL`), then enqueues the Celery inference task.
2. The inference task runs, writes `coverage_class` and `llm_explanation`
   into each row, sets `populated_by = LLM`, creates a single `LLMResult`
   record for the run, and sets `llm_result` on each row to that record.
3. The coordinator reviews the pre-populated canvas, overrides as needed,
   adds any missing modules, and sets `coordinator_decision` on each row.
4. Once all rows are decided, the D8 gate is satisfied and the
   backwards scheduler runs.

#### Field table

One row per candidate module per `PublicationTarget`. Created either
by the system when the coordinator opens the scoping canvas (populated
from the candidate pool query), or manually by the coordinator adding
a module not in the candidate pool.

| Field                  | Type                              | Notes                                                                                                                              |
|------------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| `id`                   | PK                                |                                                                                                                                    |
| `tenant`               | FK → Tenant                       |                                                                                                                                    |
| `publication_target`   | FK → PublicationTarget            | `related_name='scope_items'`                                                                                                       |
| `module`               | FK → ModuleIdentifier             |                                                                                                                                    |
| `populated_by`         | CharField (TextChoices)           | `LLM \| COORDINATOR`. Records which agent created or last substantively updated this row.                                          |
| `coverage_class`       | CharField (TextChoices, nullable) | `FULLY_COVERED \| PARTIALLY_COVERED \| NOT_COVERED \| INDETERMINATE`. NULL until the LLM runs or the coordinator sets it manually. |
| `llm_explanation`      | TextField (blank=True)            | LLM-authored rationale for the classification. Not editable by the coordinator. Empty when `populated_by = COORDINATOR`.           |
| `llm_result`           | FK → LLMResult (nullable)         | The inference run that populated `coverage_class` and `llm_explanation`. NULL when `populated_by = COORDINATOR`.                   |
| `coordinator_decision` | CharField (TextChoices)           | `PENDING \| IN_SCOPE \| OUT_OF_SCOPE`. Default `PENDING`.                                                                          |
| `coordinator_note`     | TextField (blank=True)            | Coordinator's free-text note recorded at the review step. Always editable.                                                         |
| `decided_by`           | FK → User (nullable)              | Set when `coordinator_decision` leaves `PENDING`.                                                                                  |
| `decided_at`           | DateTimeField (nullable)          | Set when `coordinator_decision` leaves `PENDING`.                                                                                  |

**`populated_by` vocabulary:**

| Value         | Meaning                                                                                                                                                     |
|---------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `LLM`         | Row was created and classified by the LLM inference task. `coverage_class` and `llm_explanation` are populated; `llm_result` is set.                        |
| `COORDINATOR` | Row was created manually by the coordinator, or the LLM is disabled. `coverage_class` may be NULL or set by the coordinator directly. `llm_result` is NULL. |

**`coverage_class` vocabulary:**

| Value               | Meaning                                                                             |
|---------------------|-------------------------------------------------------------------------------------|
| `FULLY_COVERED`     | Existing MLO text already satisfies the CLO; no revision likely required.           |
| `PARTIALLY_COVERED` | MLO text partially addresses the CLO; revision is probably needed.                  |
| `NOT_COVERED`       | No existing MLO addresses the CLO; new outcome or significant revision required.    |
| `INDETERMINATE`     | Classification is uncertain. Coordinator must make the scoping decision explicitly. |

**`coordinator_decision` vocabulary:**

| Value          | Meaning                                                                                                     |
|----------------|-------------------------------------------------------------------------------------------------------------|
| `PENDING`      | Coordinator has not yet reviewed this row.                                                                  |
| `IN_SCOPE`     | Module is confirmed in scope. The backwards scheduler will generate a `ScheduledMilestone` for this module. |
| `OUT_OF_SCOPE` | Module is confirmed out of scope. No milestone will be generated.                                           |

> **Simplification note:** The previous five-value vocabulary
> (`ACCEPTED`, `REJECTED`, `OVERRIDDEN_IN`, `OVERRIDDEN_OUT`) has been
> collapsed to three values. The override direction (whether the
> coordinator agreed or disagreed with the LLM) is recoverable at query
> time by comparing `coordinator_decision` with `coverage_class` — a
> module with `coverage_class = NOT_COVERED` and
> `coordinator_decision = OUT_OF_SCOPE` is an LLM-recommended inclusion
> that was overridden out, and so on. The audit trail is fully preserved
> without encoding the override logic into the vocabulary. The
> `coordinator_note` field carries the coordinator's rationale in either
> case.

**Constraints:**

- `tenant` must agree with `publication_target.tenant`. Enforced in `clean()`.
- `tenant` must agree with `module.tenant`. Enforced in `clean()`.
- `unique_together = (publication_target, module)` — one row per module per target.
- `decided_by` and `decided_at` must both be set or both NULL. Enforced in `clean()`.
- When `coordinator_decision` leaves `PENDING`, `decided_by` and `decided_at`
  must be set. Enforced in `clean()`.
- `llm_result` must be NULL when `populated_by = COORDINATOR`. Enforced in `clean()`.
- `llm_explanation` must be blank when `populated_by = COORDINATOR`. Enforced in `clean()`.

**Indexes:** `(publication_target,)`, `(module,)`,
`(publication_target, coordinator_decision)` — supports the D8 gate
query checking whether all rows for a target have been decided.

---

### `SchedulerAdvisory` *(new model)*

Advisory notification raised by the scheduler when a condition requiring
coordinator attention is detected. Deduplicated: at most one open advisory
per `(publication_target, advisory_type)` pair. The coordinator dismisses
the advisory once actioned; a new advisory may be raised if the condition
recurs.

| Field                  | Type                     | Notes                                                                 |
|------------------------|--------------------------|-----------------------------------------------------------------------|
| `id`                   | PK                       |                                                                       |
| `tenant`               | FK → Tenant              |                                                                       |
| `publication_target`   | FK → PublicationTarget   | `related_name='advisories'`                                           |
| `advisory_type`        | CharField (TextChoices)  | `SUGGESTED_RESCHEDULE \| MILESTONE_CRITICAL \| TARGET_UNACHIEVABLE`  |
| `suggested_target_date`| DateField (nullable)     | Populated for `SUGGESTED_RESCHEDULE` type only                        |
| `detail`               | TextField (blank)        | Plain-English explanation of the condition, generated by the scheduler|
| `generated_at`         | DateTimeField            | auto                                                                  |
| `dismissed_by`         | FK → User (nullable)     |                                                                       |
| `dismissed_at`         | DateTimeField (nullable) |                                                                       |

**Constraints:**
- `UniqueConstraint(publication_target, advisory_type, condition=Q(dismissed_at__isnull=True))` —
  at most one open advisory per target per type.
- `dismissed_by` and `dismissed_at` must both be set or both NULL. Enforced in `clean()`.
- `suggested_target_date` must be non-NULL when `advisory_type = SUGGESTED_RESCHEDULE`. Enforced in `clean()`.

**Indexes:** `(publication_target, advisory_type, dismissed_at)`,
`(dismissed_at,)` — supports querying all open advisories.

---

### `MilestonePublicationTarget`

Join table linking a `ScheduledMilestone` to additional `PublicationTarget`
instances beyond its originating target. Created when a coordinator decides
to fold an existing in-flight milestone into a second publication target.

When a row is added or removed, the scheduling queue task is enqueued for
both the affected target and the milestone's originating target, since the
shared milestone's effective deadline may shift and alter both critical paths.

| Field                | Type                    | Notes                                     |
|----------------------|-------------------------|-------------------------------------------|
| `id`                 | PK                      |                                           |
| `milestone`          | FK → ScheduledMilestone | `related_name='additional_targets'`       |
| `publication_target` | FK → PublicationTarget  | `related_name='shared_milestones'`        |
| `linked_by`          | FK → User               | Coordinator who made the sharing decision |
| `linked_at`          | DateTimeField           | auto                                      |
| `rationale`          | TextField               | Recorded for audit                        |

**Constraints:** `unique_together = (milestone, publication_target)`

**Integrity note:** `publication_target` must not equal the milestone's
originating `publication_target` — enforced in `clean()`. The originating
relationship is the FK on `ScheduledMilestone` itself; this table carries
only the additional targets.

---

### `MilestoneOverlap`

The coordinator's awareness and coordination surface. One record per
relationship identified by the awareness pass — between a
`ScheduledMilestone` and either a peer milestone from a different target
or a free-standing in-flight `WorkflowInstance` that has no milestone yet.

Records persist after a linking decision is made. The `resolution` field
and `resolved_milestone` FK record what was decided and where the work
went. This means the full coordination history is visible in one place:
what was identified, what the relationship was judged to be, and what
action (if any) was taken.

Exactly one of `milestone_b` or `workflow_instance` is set.

| Field                  | Type                               | Notes                                                                                    |
|------------------------|------------------------------------|------------------------------------------------------------------------------------------|
| `id`                   | PK                                 |                                                                                          |
| `milestone_a`          | FK → ScheduledMilestone            | The milestone being coordinated — always set. `related_name='overlaps_as_a'`             |
| `milestone_b`          | FK → ScheduledMilestone (nullable) | Set when overlap is with a milestone from another target. `related_name='overlaps_as_b'` |
| `workflow_instance`    | FK → WorkflowInstance (nullable)   | Set when overlap is with a free-standing in-flight workflow                              |
| `character`            | CharField                          | `TextChoices`: `compatible`, `incompatible`, `deadline_risk`                             |
| `explanation`          | TextField                          | Human-readable; surfaced in coordinator UI                                               |
| `decision_required_by` | DateField (nullable)               | Populated for `deadline_risk` cases                                                      |
| `resolution`           | CharField                          | `TextChoices`: `pending`, `linked`, `monitoring`, `separate`, `dismissed`                |
| `resolved_milestone`   | FK → ScheduledMilestone (nullable) | The milestone created or extended as a result of a `linked` decision                     |
| `resolved_by`          | FK → User (nullable)               |                                                                                          |
| `resolved_at`          | DateTimeField (nullable)           |                                                                                          |
| `resolution_rationale` | TextField                          | Recorded for audit; required when resolution is not `pending`                            |
| `computed_at`          | DateTimeField                      | auto                                                                                     |

**Constraints:**

- Exactly one of `milestone_b` / `workflow_instance` must be set — enforced
  in `clean()` and by a database check constraint
- `resolved_milestone` must be null unless `resolution = 'linked'` —
  enforced in `clean()`
- `milestone_a` and `milestone_b` form a unique pair when `milestone_b` is not null.
  Enforced in `clean()`.

```python
UniqueConstraint(
    fields=['milestone_a', 'milestone_b'],
    condition=Q(milestone_b__isnull=False),
    name='unique_milestone_pair_overlap',
)
```

- `milestone_a` and `workflow_instance)` form a unique pair when `workflow_instance` is not null
  Enforced in `clean()`.

```python
UniqueConstraint(
    fields=['milestone_a', 'workflow_instance'],
    condition=Q(workflow_instance__isnull=False),
    name='unique_milestone_workflow_overlap',
)
```

**Character vocabulary:**

| Value           | Meaning                                                             |
|-----------------|---------------------------------------------------------------------|
| `compatible`    | The in-flight work can satisfy both targets; no deadline conflict   |
| `incompatible`  | Separate documents are needed — the same document cannot serve both |
| `deadline_risk` | Sharing is feasible but one target's deadline would be at risk      |

**Resolution vocabulary:**

| Value        | Meaning                                                                     |
|--------------|-----------------------------------------------------------------------------|
| `pending`    | No decision made yet                                                        |
| `linked`     | Folded into scheduling — `resolved_milestone` points at the result          |
| `monitoring` | Coordinator is aware and watching but not linking for scheduling            |
| `separate`   | A parallel document is needed. The in-flight work cannot serve this target. |
| `dismissed`  | Out of scope; no further action needed                                      |

**Indexes:** `(milestone_a,)`, `(milestone_b,)`, `(workflow_instance,)`,
`(character, decision_required_by)`, `(resolution,)`

---

### `TransitionDuration`

Stores estimated duration for each FSM transition on the primary
pathway. Used by the backwards scheduler. Updated over time from
historical `WorkflowEvent` timestamps.

| Field              | Type                                | Notes                 |
|--------------------|-------------------------------------|-----------------------|
| `id`               | PK                                  |                       |
| `tenant`           | FK → Tenant                         |                       |
| `committee`        | FK → CommitteeIdentifier (nullable) | For `committee` type  |
| `workflow_type`    | CharField                           | FSM identifier        |
| `transition_event` | CharField                           | Event name            |
| `duration_type`    | CharField                           | `days` or `committee` |
| `estimated_days`   | PositiveIntegerField (nullable)     | For `days` type       |

**Constraints:** If `committee` is not null, `tenant` must agree with
`committee.tenant`. Enforced in `clean()`

---

## LLM outputs

### `LLMResult`

Persisted output of any LLM inference task. LLM outputs are
never used transiently — they are always stored before being
surfaced in the UI, and always labelled with provenance.

| Field          | Type                     | Notes                                                                                          |
|----------------|--------------------------|------------------------------------------------------------------------------------------------|
| `id`           | PK                       |                                                                                                |
| `tenant`       | FK → Tenant              |                                                                                                |
| `task_type`    | CharField                | `committee_screening`, `impact_analysis`, `form_draft`, `regulatory_check`, `schedule_summary` |
| `subject_type` | ContentType              | ContentType of the related object                                                              |
| `subject_id`   | PositiveIntegerField     | GenericForeignKey target                                                                       |
| `model_name`   | CharField                | e.g. `qwen2.5:32b-q4`                                                                          |
| `prompt_hash`  | CharField                | SHA256 of the prompt, for cache/dedup                                                          |
| `output`       | JSONField                | Structured result                                                                              |
| `generated_at` | DateTimeField            | auto                                                                                           |
| `reviewed_by`  | FK → User (nullable)     |                                                                                                |
| `reviewed_at`  | DateTimeField (nullable) |                                                                                                |
| `accepted`     | BooleanField (nullable)  | NULL = not yet reviewed                                                                        |

**Impact analysis:** for `task_type = impact_analysis`, `LLMResult` is
provenance only. The primary scoping record is `PublicationScopeItem`;
each `PublicationScopeItem` row carries a nullable FK to the `LLMResult`
record for the inference run that populated it. Do not query `LLMResult`
directly for impact analysis output — query `PublicationScopeItem` instead.

---

## Meetings and agenda

### `CommitteeMeeting` (revised)

Meeting dates and submission deadlines for governance committees. Used by
the backwards scheduler to resolve committee-gated transition durations.
Supports both standing recurring meetings and exceptional ad hoc meetings.

| Field                   | Type                     | Notes                                                                  |
|-------------------------|--------------------------|------------------------------------------------------------------------|
| `id`                    | PK                       |                                                                        |
| `tenant`                | FK → Tenant              |                                                                        |
| `committee`             | FK → CommitteeIdentifier |                                                                        |
| `academic_year`         | CharField                | e.g. `2028/29`                                                         |
| `meeting_date`          | DateField (nullable)     | NULL when `date_confidence = WINDOW`. Required for INDICATIVE/CONFIRMED.|
| `earliest_expected_date`| DateField (nullable)     | Lower bound of scheduling window. Populated for standing meetings.     |
| `latest_expected_date`  | DateField (nullable)     | Upper bound of scheduling window. Populated for standing meetings.     |
| `date_confidence`       | CharField (TextChoices)  | `WINDOW \| PROVISIONAL \| INDICATIVE \| CONFIRMED`                    |
| `submission_deadline`   | DateField (nullable)     | NULL when `meeting_date` is NULL. Latest date for paper submission.    |
| `is_exceptional`        | BooleanField             | Ad hoc meeting; not part of standing calendar. Default False.          |
| `notes`                 | TextField (blank)        |                                                                        |
| `confirmed_by`          | FK → User (nullable)     | Set when `date_confidence` transitions to CONFIRMED                    |
| `confirmed_at`          | DateTimeField (nullable) |                                                                        |

**Constraints:**
- `tenant` must agree with `committee.tenant`. Enforced in `clean()`.
- `meeting_date` must be non-NULL when `date_confidence` is `INDICATIVE` or
  `CONFIRMED`. Enforced in `clean()`.
- `submission_deadline` must be non-NULL when `meeting_date` is non-NULL.
  Enforced in `clean()`.
- At least one of `meeting_date` or the pair (`earliest_expected_date`,
  `latest_expected_date`) must be set. Enforced in `clean()`.
- `earliest_expected_date < latest_expected_date` when both are set.
  Enforced in `clean()`.
- `confirmed_by` and `confirmed_at` must both be set or both NULL.
  Enforced in `clean()`.

**Scheduler trigger:** when `date_confidence` transitions to `CONFIRMED`, the
scheduler is enqueued for all `PublicationTarget` records whose critical path
includes this meeting.

---

### `AgendaItem`

Links a document to a meeting agenda, regardless of document type.
Demonstrates the cross-cutting query benefit of the envelope model.

| Field          | Type                    | Notes                                                                       |
|----------------|-------------------------|-----------------------------------------------------------------------------|
| `id`           | PK                      |                                                                             |
| `meeting`      | FK → CommitteeMeeting   | `related_name='agenda_items'`                                               |
| `document`     | FK → Document           | `related_name='agenda_items'`                                               |
| `order`        | PositiveIntegerField    |                                                                             |
| `notes`        | TextField (blank=True)  |                                                                             |
| `outcome`      | CharField (TextChoices) | `PENDING \| APPROVED \| REJECTED \| DEFERRED`. Default `PENDING`.           |
| `outcome_note` | TextField (blank=True)  | Committee's reasons for rejection or deferral. Populated after the meeting. |

**Constraints:** `unique_together = (meeting, document)`

**Ordering:** `order ASC`

### `CommitteeFeedbackItem`

Structured feedback from a committee on a specific document within a
rejected submission. Hangs off `AgendaItem`; one record per document
commented on. Replaces the informal practice of embedding per-document
feedback in `WorkflowEvent.metadata`, which was designed for lightweight
transition annotations and is not appropriate for structured committee
responses to substantive curriculum proposals.

`WorkflowEvent.metadata` on the bundle-level rejection transition retains
its role as the authoritative record of the transition event itself (who
acted, when, in what role). `CommitteeFeedbackItem` carries the content
of the committee's feedback, navigable by document.

| Field         | Type                    | Notes                                                                                                                                                                                                             |
|---------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `id`          | PK                      |                                                                                                                                                                                                                   |
| `agenda_item` | FK → AgendaItem         | `related_name='feedback_items'`. The agenda item for the bundle-level CLO or primary document.                                                                                                                    |
| `document`    | FK → Document           | `related_name='committee_feedback_items'`. The specific document this feedback concerns. May be the same document as `agenda_item.document` for bundle-level feedback, or a dependent document (MLO, assessment). |
| `feedback`    | TextField               | The committee's feedback. Plain text. No length limit — committee responses to substantive proposals may be extensive.                                                                                            |
| `severity`    | CharField (TextChoices) | `ADVISORY \| REQUIRED`. `ADVISORY` — committee recommends a change but will not block approval. `REQUIRED` — change must be made before resubmission. Default `REQUIRED`.                                         |
| `created_at`  | DateTimeField           | auto                                                                                                                                                                                                              |

**Constraints:** no uniqueness constraint — a committee may record multiple
feedback items for the same document at the same meeting (e.g. separate
items for outcome language and assessment alignment).

**Ordering:** `agenda_item ASC, created_at ASC`

**Integrity note:** `document.tenant` must equal
`agenda_item.meeting.tenant` — enforced in `clean()`.

**Query helper on `Document`:**

```python
def committee_feedback_items(self):
    """
    Returns all CommitteeFeedbackItem records concerning this document,
    across all agenda items and all meetings. Ordered by meeting date
    then creation order, giving a chronological feedback history.
    """
    return CommitteeFeedbackItem.objects.filter(
        document=self,
    ).select_related(
        'agenda_item__meeting',
    ).order_by(
        'agenda_item__meeting__meeting_date',
        'created_at',
    )
```

This helper is the primary access path when surfacing prior committee
feedback during a revision cycle — for example, when a document has moved
to a new `PublicationTarget` following a rescoping exercise. Feedback
attribution is preserved across rescoping because `CommitteeFeedbackItem`
belongs to the meeting and the submission event, not to the target.

---

## FSM (in code, not the database)

The following structures exist in Python code and are not
persisted as database tables. They are documented here for
completeness.

### `Transition` dataclass

```python
@dataclass
class Transition:
    target: StateEnum  # target state
    permitted_roles: set[str]  # roles that can fire this event
    side_effects: list[Callable]  # task creation, versioning, notifications
    generates_milestone: bool = False  # True for terminal approval transitions
    milestone_label: str = ''
    committee: str | None = None  # committee slug if gated on a meeting
    estimated_days: int | None = None  # processing time if not committee-gated
    is_rejection_branch: bool = False  # marks non-primary paths
```

### `DOCUMENT_DEPENDENCIES` dict

```python
# workflows/dependencies.py
DOCUMENT_DEPENDENCIES = {
    DocumentType.COURSE_SPECIFICATION: [
        Dependency(
            dependent_transition='submit_for_approval',
            requires_type=DocumentType.MODULE_SPECIFICATION,
            requires_state=ModuleSpecState.APPROVED,
            resolution='all',
            stale_limit=timedelta(days=365 * 3),  # routine update cycle
        ),
    ],
    DocumentType.REACCREDITATION: [
        Dependency(
            dependent_transition='submit_to_body',
            requires_type=DocumentType.MODULE_SPECIFICATION,
            requires_state=ModuleSpecState.APPROVED,
            resolution='all',
            stale_limit=timedelta(days=365 * 1),  # tighter for PSRB submission
        ),
        Dependency(
            dependent_transition='submit_to_body',
            requires_type=DocumentType.MODULE_OUTCOME,
            requires_state=ModuleOutcomeState.APPROVED,
            resolution='all',
            stale_limit=None,  # never auto-initiate — convenor decides
        ),
    ],
}
```

Dependencies are in code (not the database) because:

- They change only on deployment, not at runtime
- They require developer review when changed
- They benefit from version control and startup validation

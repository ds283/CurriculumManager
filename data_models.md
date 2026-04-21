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
`convenor` and `tlc` or `tlc_chair` roles).

| Field       | Type         | Notes                                                                                                                                                                                                                                                                                                                     |
|-------------|--------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `id`        | PK           |                                                                                                                                                                                                                                                                                                                           |
| `user`      | FK → User    | Global Django user                                                                                                                                                                                                                                                                                                        |
| `tenant`    | FK → Tenant  |                                                                                                                                                                                                                                                                                                                           |
| `role`      | CharField    | `TextChoices`: `convenor`, `admin`, `department_tlc`, `department_tlc_chair`, `school_tlc`, `school_tlc_chair`, `faculty_tlc`, `faculty_tlc_chair`, `manager`, `external_examiner`, `review_coordinator`, `review_lead`, `accreditation_coordinator`, `accreditation_lead`, `curriculum_officer`, `programme_coordinator` |
| `is_active` | BooleanField |                                                                                                                                                                                                                                                                                                                           |

**Constraints:** `unique_together = (user, tenant, role)`

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

Organizational unit. Owns programmes and modules. May have its own committees that
approve certain types of document.

| Field     | Type                             | Notes                                               |
|-----------|----------------------------------|-----------------------------------------------------|
| `id`      | PK                               |                                                     | 
| `tenant`  | FK → Tenant                      | Must equal `faculty.tenant` — enforced in `clean()` |
| `faculty` | FK → FacultyIdentifier           |                                                     |
| `school`  | FK → SchoolIdentifier (nullable) | Optional. Schools may not be used.                  |
| `name`    | CharField                        |                                                     |

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

### `PeriodUnitIdentifier`

A defined interval within a period — usually a week, but the model
does not assume this. Units are ordered within their parent period by
`sequence`, managed through the same two-pass pattern as
`PeriodIdentifier`.

| Field      | Type                  | Notes                                              |
|------------|-----------------------|----------------------------------------------------|
| `id`       | PK                    |                                                    |
| `tenant`   | FK → Tenant           | Must equal `period.tenant` — enforced in `clean()` |
| `period`   | FK → PeriodIdentifier | `related_name='units'`                             |
| `name`     | CharField             | e.g. `Week 1`, `Intersemester Week`                |
| `sequence` | PositiveIntegerField  | Managed — use `reorder()` / `append()`             |

**Constraints:** `unique_together = (name, period)`,
`UniqueConstraint(period, sequence)` — enforces no two units share a
position within the same period

**Ordering:** `sequence ASC`

**Sequencing methods:**

```python
# Append a new unit at the end of this period
PeriodUnitIdentifier.append(period, name="Week 12")

# Reassign positions 1..N within a period
PeriodUnitIdentifier.reorder(period, ordered_ids=[5, 1, 2, 3, 4])
```

**Sequencing implementation note:**

Both models use a two-pass bulk update inside `transaction.atomic()` to
avoid transient constraint violations when renumbering. Pass 1 shifts
all existing positions to a high offset (`current + 1000`); pass 2
assigns the final 1..N values. Direct assignment to `sequence` outside
these methods is not safe and should be avoided.

```python
@classmethod
def reorder(cls, scope, ordered_ids: list[int]) -> None:
    # `scope` is `tenant` for PeriodIdentifier,
    #            `period`  for PeriodUnitIdentifier
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
to a programme) carry a non-null `tenant` FK; global documents (e.g.
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

### `ProgrammeIdentifier`

| Field        | Type                      | Notes                                                  |
|--------------|---------------------------|--------------------------------------------------------|
| `id`         | PK                        |                                                        |
| `tenant`     | FK → Tenant               | Must equal `department.tenant` — enforced in `clean()` |
| `department` | FK → DepartmentIdentifier |                                                        | 
| `code`       | CharField                 | unique code, matches institutional code, e.g. F3058U   |
| `name`       | CharField                 | programme name                                         |
| `type`       | CharField                 | `TextChoices`:'UG', 'PGT', 'PGR'                       |
| `moa`        | CharField                 | `TextChoices`: 'FT', 'PT', 'Mixed'                     |

**Constraints:** `unique_together = (code, tenant)`

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

**Integrity note:** `department.tenant` must equal `tenant` — enforced in `clean()`.

---

### `ModuleIdentifier`

| Field        | Type                      | Notes                                                  |
|--------------|---------------------------|--------------------------------------------------------|
| `id`         | PK                        |                                                        |
| `tenant`     | FK → Tenant               | Must equal `department.tenant` — enforced in `clean()` |
| `department` | FK → DepartmentIdentifier |                                                        |
| `code`       | CharField                 | unique code, matches institutional code, e.g. F3202    |
| `name`       | CharField                 | module name (mutable)                                  |
| `level`      | FK → FHEQLevel            | FHEQ level descriptor                                  |

**Constraints:** `unique_together = (code, tenant)`

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

**Integrity note:** `department.tenant` must equal `tenant` — enforced in `clean()`.

---

### `AssessmentSlotIdentifier`

A typed registry entry for each assessment within a module. Created when
a module is established. Retired rather than deleted when an assessment
is removed, so that historical assessment brief documents retain a valid
reference.

| Field             | Type                  | Notes                                                  |
|-------------------|-----------------------|--------------------------------------------------------|
| `id`              | PK                    |                                                        |
| `tenant`          | FK → Tenant           | Must equal `module.tenant` — enforced in `clean()`     |
| `module`          | FK → ModuleIdentifier | `related_name='assessment_slots'`                      |
| `assessment_type` | CharField             | `exam`, `cwk`, `prb` — controlled vocabulary, see note |
| `sequence`        | PositiveIntegerField  | Position within type for this module, e.g. `1`, `2`    |
| `retired_at`      | DateField (nullable)  | NULL = active; set when slot is retired                |

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
tenant-scoped. Programme-level accreditation is expressed via
`ProgrammeAccreditation`.

| Field          | Type                | Notes                       |
|----------------|---------------------|-----------------------------|
| `id`           | PK                  |                             |
| `name`         | CharField unique    | e.g. `Institute of Physics` |
| `abbreviation` | CharField           | e.g. `IoP`                  |
| `url`          | URLField (nullable) |                             |

---

## Scoping layer

### `ProgrammeConvenorScope`

| Field        | Type                     | Notes                   |
|--------------|--------------------------|-------------------------|
| `id`         | PK                       |                         |
| `tenant`     | FK → Tenant              |
| `programme`  | FK → ProgrammeIdentifier |                         |
| `user`       | FK → User                |                         |
| `valid_from` | DateField                |                         |
| `valid_to`   | DateField (nullable)     | NULL = currently active |

**Constraints:**

```python
UniqueConstraint(
    fields=['programme', 'user'],
    condition=Q(valid_to__isnull=True),
    name='unique_active_programme_convenor',
)
```

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

**Integrity note:** `tenant` must equal `programme.tenant` – enforced in `clean()`

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

**Integrity note:** `tenant` must equal `module.tenant` – enforced in `clean()`

---

### `ProgrammeAccreditationScope`

Records the accreditation of a programme by an accrediting body for a
defined period. Append-only — accreditation is closed by setting
`valid_to` rather than deleting the record. PSRB-specific
`RegulatoryDocument` records applicable to this accreditation are
linked via the M2M.

| Field                  | Type                           | Notes                                                 |
|------------------------|--------------------------------|-------------------------------------------------------|
| `id`                   | PK                             |                                                       |
| `tenant`               | FK → Tenant                    | Must equal `programme.tenant` — enforced in `clean()` |
| `programme`            | FK → ProgrammeIdentifier       |                                                       |
| `accrediting_body`     | FK → AccreditingBodyIdentifier |                                                       |
| `valid_from`           | DateField                      |                                                       |
| `valid_to`             | DateField (nullable)           | NULL = currently accredited                           |
| `regulatory_documents` | M2M → RegulatoryDocument       | PSRB requirements applicable to this accreditation    |
| `notes`                | TextField (blank)              |                                                       |

**Constraints:**

```python
UniqueConstraint(
    fields=['programme', 'accrediting_body'],
    condition=Q(valid_to__isnull=True),
    name='unique_active_programme_accreditation',
)
```

**Managers:** `objects = TenantScopedManager()`,
`all_tenants = TenantScopedManager(require_tenant=False)`

**Current accreditations query:**

```python
ProgrammeAccreditationScope.objects.for_tenant(tenant).filter(
    programme=programme,
    valid_to__isnull=True,
)
```

---

## Document layer

### `Document` (envelope)

Shared envelope for all document and form types. All cross-cutting
queries operate on this table. Type-specific content lives in a
separate table linked by `OneToOneField`.

| Field              | Type                                        | Notes                                                         |
|--------------------|---------------------------------------------|---------------------------------------------------------------|
| `id`               | PK                                          |                                                               |
| `tenant`           | FK → Tenant                                 |                                                               |
| `document_type`    | CharField                                   | `DocumentType` choices — see registry                         |
| `classification`   | CharField                                   | `TextChoices`: `published_asset`, `workflow_form`, `internal` |
| `parent`           | FK → Document (nullable)                    | Self-referential ownership hierarchy                          |
| `title`            | CharField                                   |                                                               |
| `reference_code`   | CharField                                   | Unique per tenant                                             |
| `due_date`         | DateTimeField (nullable)                    | Updated by scheduler                                          |
| `current_workflow` | OneToOneField → WorkflowInstance (nullable) | Denormalised for fast reads                                   |
| `current_version`  | OneToOneField → AssetVersion (nullable)     | Denormalised for fast reads                                   |
| `created_at`       | DateTimeField                               | auto                                                          |
| `updated_at`       | DateTimeField                               | auto                                                          |

**Constraints:** `unique_together = (tenant, reference_code)`

**Indexes:** `(tenant, document_type)`, `(tenant, due_date)`, `(parent,)`

**Managers:** `objects = TenantScopedManager()`, `all_tenants = TenantScopedManager(require_tenant=False)`

---

### `[Type]Content` (one table per document type)

Each document type has exactly one content model. The pattern is
always a `OneToOneField(Document, related_name='[type]')`.
Content models carry only type-specific fields.

**Example: `ModuleSpecificationContent`**

| Field           | Type                     | Notes                        |
|-----------------|--------------------------|------------------------------|
| `id`            | PK                       |                              |
| `document`      | OneToOneField → Document | `related_name='module_spec'` |
| `credit_points` | PositiveIntegerField     |                              |
| `level`         | CharField                | FHEQ level                   |
| `subject_area`  | CharField                |                              |

**Other content models follow the same pattern:**
`ProgrammeSpecificationContent`, `AssessmentBriefContent`,
`LearningOutcomeContent`, `ConsultationFormContent`, etc.

**Accessing content:**

```python
doc = Document.objects.select_related('module_spec').get(pk=pk)
content = doc.module_spec  # single indexed join, no second query
```

---

## Workflow layer

### `WorkflowInstance`

Current FSM state of a document. One instance per document.
Uses `GenericForeignKey` so it can reference any document type
without requiring per-type FK columns.

| Field           | Type                 | Notes                                     |
|-----------------|----------------------|-------------------------------------------|
| `id`            | PK                   |                                           |
| `content_type`  | FK → ContentType     | For GenericForeignKey                     |
| `object_id`     | PositiveIntegerField | For GenericForeignKey                     |
| `form`          | GenericForeignKey    | Points to the document                    |
| `workflow_type` | CharField            | e.g. `module_specification`               |
| `state`         | CharField            | Stores `enum.name`, e.g. `FACULTY_REVIEW` |
| `created_at`    | DateTimeField        | auto                                      |
| `updated_at`    | DateTimeField        | auto                                      |

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

A pending action for a role pool or individual. Created and
superseded by FSM transitions. One task per pool — not one per user.

| Field               | Type                     | Notes                                                          |
|---------------------|--------------------------|----------------------------------------------------------------|
| `id`                | PK                       |                                                                |
| `workflow_instance` | FK → WorkflowInstance    | `related_name='tasks'`                                         |
| `event`             | CharField                | The FSM event this task enables                                |
| `assignee_pool`     | CharField                | Role name — who can act                                        |
| `claimed_by`        | FK → User (nullable)     | Set atomically on claim                                        |
| `claimed_at`        | DateTimeField (nullable) |                                                                |
| `status`            | CharField                | `TextChoices`: `pending`, `claimed`, `completed`, `superseded` |
| `priority_score`    | IntegerField             | Computed at write time                                         |
| `due_date`          | DateTimeField (nullable) | Set and updated by scheduler                                   |
| `created_at`        | DateTimeField            | auto                                                           |

**Indexes:** `(status, assignee_pool, priority_score)`

**Claim guard (atomic):**

```sql
UPDATE tasks
SET claimed_by=$user,
    status='claimed'
WHERE id = $id
  AND claimed_by IS NULL
-- Zero rows affected → another user claimed first
```

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

### `PublicationTarget`

A desired outcome with a target date. Drives the backwards
scheduler. All scheduled milestones are grouped under this entity.

| Field                  | Type                     | Notes                              |
|------------------------|--------------------------|------------------------------------|
| `id`                   | PK                       |                                    |
| `tenant`               | FK → Tenant              |                                    |
| `label`                | CharField                | e.g. `AY 2029/30 programme review` |
| `description`          | TextField                |                                    |
| `target_document_type` | CharField                | The terminal document type         |
| `target_state`         | CharField                | The required terminal state        |
| `target_date`          | DateField                |                                    |
| `owner`                | FK → User                | Programme coordinator              |
| `programme`            | FK → Document (nullable) | Anchor document if applicable      |
| `is_active`            | BooleanField             |                                    |
| `created_at`           | DateTimeField            | auto                               |

---

### `ScheduledMilestone`

One record per document required by a publication target. Carries the
computed deadline, initiation date, risk status, and linking decision.
Rebuilt atomically when any scheduling input changes.

A milestone is always anchored to a `PublicationTarget`. That target may
be a conventional publication cycle (prospectus, programme review) or an
event-driven coordination effort with a real external deadline, such as a
reaccreditation submission. In both cases the target date drives the
backwards scheduler. A milestone may be shared across multiple
`PublicationTarget` instances via `MilestonePublicationTarget`; its
effective due date for scheduling purposes is always `MIN(target_date)`
across all linked targets.

| Field                  | Type                     | Notes                                           |
|------------------------|--------------------------|-------------------------------------------------|
| `id`                   | PK                       |                                                 |
| `publication_target`   | FK → PublicationTarget   | Originating target. `related_name='milestones'` |
| `document`             | FK → Document (nullable) | NULL if workflow not yet initiated              |
| `document_type`        | CharField                | Required document type                          |
| `required_state`       | CharField                | State the document must reach                   |
| `must_be_complete_by`  | DateField                | Computed — minimum across all linked targets    |
| `must_be_initiated_by` | DateField                | Computed by backwards scheduler                 |
| `is_critical_path`     | BooleanField             |                                                 |
| `at_risk`              | BooleanField             | Current trajectory cannot meet deadline         |
| `computed_at`          | DateTimeField            | auto — when the scheduler last ran              |

**Constraints:** `unique_together = (publication_target, document_type, document)`

**Indexes:** `(publication_target, must_be_complete_by)`, `(document,)`,
`(document_type)`

**Scheduling note:** `must_be_complete_by` is recomputed whenever a
`MilestonePublicationTarget` row is added or removed for this milestone.
The scheduler task is enqueued for all linked targets when this happens,
since a shared milestone with an earlier effective deadline may alter the
critical path of every target it belongs to.

**Note:** The link status and linking decision fields previously on this
model have moved to `MilestoneOverlap`, which is now the sole coordination
and awareness surface. A milestone is created or extended as a *result* of
a linking decision; it does not record the decision itself.

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

Exactly one of `milestone_b` or `workflow_instance` is set, except when
`character` is `gap` — indicating a document identified as stale by the
dependency graph for which no workflow is yet active — in which case both
are null.

| Field                  | Type                               | Notes                                                                                    |
|------------------------|------------------------------------|------------------------------------------------------------------------------------------|
| `id`                   | PK                                 |                                                                                          |
| `milestone_a`          | FK → ScheduledMilestone            | The milestone being coordinated — always set. `related_name='overlaps_as_a'`             |
| `milestone_b`          | FK → ScheduledMilestone (nullable) | Set when overlap is with a milestone from another target. `related_name='overlaps_as_b'` |
| `workflow_instance`    | FK → WorkflowInstance (nullable)   | Set when overlap is with a free-standing in-flight workflow                              |
| `character`            | CharField                          | `compatible`, `incompatible`, `deadline_risk`, `gap`                                     |
| `explanation`          | TextField                          | Human-readable; surfaced in coordinator UI                                               |
| `decision_required_by` | DateField (nullable)               | Populated for `deadline_risk` cases                                                      |
| `resolution`           | CharField                          | `pending`, `linked`, `monitoring`, `dismissed`                                           |
| `resolved_milestone`   | FK → ScheduledMilestone (nullable) | The milestone created or extended as a result of a `linked` decision                     |
| `resolved_by`          | FK → User (nullable)               |                                                                                          |
| `resolved_at`          | DateTimeField (nullable)           |                                                                                          |
| `resolution_rationale` | TextField                          | Recorded for audit; required when resolution is not `pending`                            |
| `computed_at`          | DateTimeField                      | auto                                                                                     |

**Constraints:**

- At most one of `milestone_b` / `workflow_instance` may be set — enforced
  in `clean()` and by a database check constraint
- `character = 'gap'` requires both `milestone_b` and `workflow_instance`
  to be null — enforced in `clean()`
- `resolved_milestone` must be null unless `resolution = 'linked'` —
  enforced in `clean()`
- `unique_together = (milestone_a, milestone_b)` where `milestone_b`
  is non-null
- `unique_together = (milestone_a, workflow_instance)` where
  `workflow_instance` is non-null

**Character vocabulary:**

| Value           | Meaning                                                             |
|-----------------|---------------------------------------------------------------------|
| `compatible`    | The in-flight work can satisfy both targets; no deadline conflict   |
| `incompatible`  | Separate documents are needed — the same document cannot serve both |
| `deadline_risk` | Sharing is feasible but one target's deadline would be at risk      |
| `gap`           | A stale document has no active workflow; action may be needed       |

**Resolution vocabulary:**

| Value        | Meaning                                                            |
|--------------|--------------------------------------------------------------------|
| `pending`    | No decision made yet                                               |
| `linked`     | Folded into scheduling — `resolved_milestone` points at the result |
| `monitoring` | Coordinator is aware and watching but not linking for scheduling   |
| `dismissed`  | Out of scope; no further action needed                             |

**Indexes:** `(milestone_a,)`, `(milestone_b,)`, `(workflow_instance,)`,
`(character, decision_required_by)`, `(resolution,)`


---

### `CommitteeMeeting`

Meeting dates and submission deadlines for governance committees.
Used by the backwards scheduler to resolve committee-gated
transition durations.

| Field                 | Type           | Notes                            |
|-----------------------|----------------|----------------------------------|
| `id`                  | PK             |                                  |
| `committee`           | FK → Committee |                                  |
| `meeting_date`        | DateField      |                                  |
| `submission_deadline` | DateField      | Latest date for paper submission |
| `academic_year`       | CharField      | e.g. `2028/29`                   |

---

### `TransitionDuration`

Stores estimated duration for each FSM transition on the primary
pathway. Used by the backwards scheduler. Updated over time from
historical `WorkflowEvent` timestamps.

| Field              | Type                            | Notes                 |
|--------------------|---------------------------------|-----------------------|
| `id`               | PK                              |                       |
| `workflow_type`    | CharField                       | FSM identifier        |
| `transition_event` | CharField                       | Event name            |
| `duration_type`    | CharField                       | `days` or `committee` |
| `estimated_days`   | PositiveIntegerField (nullable) | For `days` type       |
| `committee`        | FK → Committee (nullable)       | For `committee` type  |

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
| `subject_type` | CharField                | ContentType of the related object                                                              |
| `subject_id`   | PositiveIntegerField     | GenericForeignKey target                                                                       |
| `model_name`   | CharField                | e.g. `qwen2.5:32b-q4`                                                                          |
| `prompt_hash`  | CharField                | SHA256 of the prompt, for cache/dedup                                                          |
| `output`       | JSONField                | Structured result                                                                              |
| `generated_at` | DateTimeField            | auto                                                                                           |
| `reviewed_by`  | FK → User (nullable)     |                                                                                                |
| `reviewed_at`  | DateTimeField (nullable) |                                                                                                |
| `accepted`     | BooleanField (nullable)  | NULL = not yet reviewed                                                                        |

---

## Meetings and agenda

### `Meeting`

A governance committee meeting instance. Documents are linked
to meetings via `AgendaItem`.

| Field    | Type        | Notes |
|----------|-------------|-------|
| `id`     | PK          |       |
| `tenant` | FK → Tenant |       |
| `title`  | CharField   |       |
| `date`   | DateField   |       |

---

### `AgendaItem`

Links a document to a meeting agenda, regardless of document type.
Demonstrates the cross-cutting query benefit of the envelope model.

| Field      | Type                 | Notes                         |
|------------|----------------------|-------------------------------|
| `id`       | PK                   |                               |
| `meeting`  | FK → Meeting         | `related_name='items'`        |
| `document` | FK → Document        | `related_name='agenda_items'` |
| `order`    | PositiveIntegerField |                               |
| `notes`    | TextField            |                               |

**Constraints:** `unique_together = (meeting, document)`

**Ordering:** `order ASC`

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
    DocumentType.PROGRAMME_SPECIFICATION: [
        Dependency(
            dependent_transition='submit_for_approval',
            requires_type=DocumentType.MODULE_SPECIFICATION,
            requires_state=ModuleSpecState.APPROVED,
            resolution='all',
        ),
    ],
    ...
}
```

Dependencies are in code (not the database) because:

- They change only on deployment, not at runtime
- They require developer review when changed
- They benefit from version control and startup validation

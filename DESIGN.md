# DESIGN.md — Curriculum Governance Workflow System

**Read this file in full before writing any code.**
All architectural decisions here take precedence over inferred conventions.

Part 1 (Sections 1–6) drives immediate skeleton generation.
Part 2 (Sections 7–9) describes planned subsystems — design for
compatibility but do not generate implementation code for those sections.

**Note:** The system uses internal Sussex vocabulary to refer to curriculum
entities. In particular, 'course' is used to refer to programmes of student,
in alignment with the Sussex 2025/26 Academic Framework. This is synonymous
with 'programme' used more widely in the sector.

---

## Part 1 — Generate now

---

## 1. Technology stack

| Component      | Choice             | Notes                                                 |
|----------------|--------------------|-------------------------------------------------------|
| Language       | Python 3.12+       |                                                       |
| Framework      | Django 5.x         | Not Flask. Use integrated migrations, admin, ORM.     |
| Database       | PostgreSQL 16+     | jsonb, recursive CTEs, transactional DDL required.    |
| Task queue     | Celery 5.x         | Redis broker. Named queues — see §6.                  |
| Cache / broker | Redis              |                                                       |
| Auth           | Okta SSO           | django-allauth with Okta provider. Custom middleware. |
| Frontend       | HTMX + Bootstrap 5 | No React SPA. Django templates throughout.            |
| WSGI           | Waitress           |                                                       |

Key packages: `networkx` (graph algorithms), `pydantic` (JSON schema
validation), `ollama` Python SDK (deferred), `django-guardian`
(object-level permissions — install but do not configure until §6 is
in place).

Do **not** use `django-fsm` or `django-simple-history`. Both are
implemented custom.

---

## 2. Multi-tenancy

Strategy: **shared schema with tenant FK**. Every tenant-scoped model
carries a `tenant` FK. Isolation is enforced via a custom manager that
raises `RuntimeError` on unscoped access.

### 2.1 Tenant model

```python
class Tenant(models.Model):
    name = models.CharField(max_length=255)
    slug = models.SlugField(unique=True)  # subdomain identifier
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
```

### 2.2 Tenant resolution

Tenant is resolved from the **subdomain** on every request, before
authentication. The logged-in user does not identify the tenant.

- `oxford.app.com` → slug `oxford` → `Tenant` lookup
- Resolved tenant stored on `request.tenant` and in a `ContextVar`
- Use `contextvars.ContextVar` — safe under async Django and Celery

### 2.3 Custom manager

All tenant-scoped models use `TenantScopedManager`. The default
queryset raises `RuntimeError` if `.for_tenant()` has not been called.
Expose `all_tenants` manager for admin/management use only.

```python
class TenantScopedManager(models.Manager):
    def for_tenant(self, tenant):
        return self.get_queryset().filter(tenant=tenant)
```

### 2.4 User–tenant relationship

User records are global (`AUTH_USER_MODEL`). Roles are per-tenant.

```python

class TenantMembership(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    tenant = models.ForeignKey(Tenant, on_delete=models.CASCADE)
    role = models.CharField(max_length=100)
    is_active = models.BooleanField(default=True)

    class Meta:
        unique_together = [('user', 'role', 'tenant')]
```

`request.tenant_roles` is set by `TenantMembershipMiddleware` after
authentication. The FSM uses this for permission checks.

---

## 3. Core data model

### 3.1 Document envelope

All document and form types share a single envelope table.
Type-specific content lives in a separate table with a
`OneToOneField` back to the envelope. **Do not deviate from this.**

```python
class Document(models.Model):
    class DocumentType(models.TextChoices):
        MODULE_SPECIFICATION = 'module_spec', 'Module Specification'
        PROGRAMME_SPECIFICATION = 'course_spec', 'Course Specification'
        ASSESSMENT = 'assessment', 'Assessment'
        LEARNING_OUTCOME = 'learning_outcome', 'Learning Outcome'
        TEACHING_ACTIVITY = 'teaching_activity', 'Teaching Activity'
        REACCREDITATION = 'reaccreditation', 'Reaccreditation'
        MODULE_REQUISITE = 'module_requisite', 'Module Requisite'  # deferred — governance model TBD with curriculum team
        CURRICULUM_REVIEW = 'curriculum_review', 'Curriculum Review'  # deferred — follows same pattern as REACCREDITATION        

    # Add new types here alongside their content model

    class Classification(models.TextChoices):
        PUBLISHED_ASSET = 'published_asset'
        WORKFLOW_FORM = 'workflow_form'
        INTERNAL = 'internal'

    tenant = models.ForeignKey(Tenant, on_delete=models.PROTECT)
    document_type = models.CharField(max_length=50, choices=DocumentType.choices)
    classification = models.CharField(max_length=20, choices=Classification.choices)
    parent = models.ForeignKey(
        'self', null=True, blank=True,
        on_delete=models.PROTECT, related_name='children')
    title = models.CharField(max_length=255)
    reference_code = models.CharField(max_length=100)
    due_date = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    # Denormalised for fast reads — updated on transition / publication
    current_workflow = models.OneToOneField(
        'WorkflowInstance', null=True, blank=True, on_delete=models.PROTECT)
    current_version = models.OneToOneField(
        'AssetVersion', null=True, blank=True, on_delete=models.PROTECT)

    objects = TenantScopedManager()
    all_tenants = TenantScopedManager(require_tenant=False)

    class Meta:
        unique_together = [('tenant', 'reference_code')]
        indexes = [
            models.Index(fields=['tenant', 'document_type']),
            models.Index(fields=['tenant', 'due_date']),
            models.Index(fields=['parent']),
        ]
```

### 3.2 Content models (one per document type)

Pattern: always a `OneToOneField` named `document`.
Generate `ModuleSpecificationContent` as the reference implementation.

```python
class ModuleSpecificationContent(models.Model):
    document = models.OneToOneField(
        Document, on_delete=models.CASCADE,
        related_name='module_spec')
    credit_points = models.PositiveIntegerField()
    level = models.ForeignKey(FHEQLevel, on_delete=models.PROTECT)  # FHEQ level
    subject_area = models.CharField(max_length=100)
```

Accessing content (single indexed join, no N+1):

```python
doc = Document.objects.select_related('module_spec').get(pk=pk)
content = doc.module_spec
```

**Extension pattern** — see the Extension checklist at the end of this
document for the full step-by-step procedure.

### 3.3 Workflow instance and audit trail

```python
class WorkflowInstance(models.Model):
    content_type = models.ForeignKey(ContentType, on_delete=models.PROTECT)
    object_id = models.PositiveIntegerField()
    form = GenericForeignKey('content_type', 'object_id')
    workflow_type = models.CharField(max_length=100)
    state = models.CharField(max_length=100)  # stores enum.name
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        indexes = [
            models.Index(fields=['content_type', 'object_id']),
            models.Index(fields=['workflow_type', 'state']),
        ]


class WorkflowEvent(models.Model):  # append-only — never update or delete
    workflow_instance = models.ForeignKey(
        WorkflowInstance, on_delete=models.PROTECT, related_name='events')
    event = models.CharField(max_length=100)
    from_state = models.CharField(max_length=100)
    to_state = models.CharField(max_length=100)
    actor = models.ForeignKey(User, on_delete=models.PROTECT)
    actor_role = models.CharField(max_length=100)
    timestamp = models.DateTimeField(auto_now_add=True)
    metadata = models.JSONField(default=dict)

    class Meta:
        ordering = ['timestamp']
        indexes = [models.Index(fields=['workflow_instance', 'timestamp'])]
```

### 3.4 Task model

One task per pool — not one per user. A claim mechanism handles
concurrent access atomically.

```python
class Task(models.Model):
    workflow_instance = models.ForeignKey(
        WorkflowInstance, on_delete=models.PROTECT, related_name='tasks')
    event = models.CharField(max_length=100)
    assignee_pool = models.CharField(max_length=100)  # role name
    claimed_by = models.ForeignKey(
        User, null=True, blank=True, on_delete=models.PROTECT)
    claimed_at = models.DateTimeField(null=True, blank=True)
    status = models.CharField(max_length=20)
    # pending | claimed | completed | superseded
    priority_score = models.IntegerField(default=0)
    due_date = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        indexes = [
            models.Index(fields=['status', 'assignee_pool', 'priority_score']),
        ]
```

Claim guard (atomic — zero rows = already claimed):

```sql
UPDATE tasks
SET claimed_by=$user,
    status='claimed'
WHERE id = $id
  AND claimed_by IS NULL
```

### 3.5 Asset versioning

```python
class Asset(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.PROTECT)
    asset_type = models.CharField(max_length=50)
    code = models.CharField(max_length=100)
    current_version = models.OneToOneField(
        'AssetVersion', null=True, blank=True,
        on_delete=models.SET_NULL, related_name='+')

    class Meta:
        unique_together = [('tenant', 'code')]


class AssetVersion(models.Model):  # append-only
    asset = models.ForeignKey(
        Asset, on_delete=models.PROTECT, related_name='versions')
    version_number = models.PositiveIntegerField()
    status = models.CharField(max_length=30)
    # draft | release_candidate | published | superseded | withdrawn
    valid_from = models.DateTimeField(null=True, blank=True)
    valid_to = models.DateTimeField(null=True, blank=True)
    content = models.JSONField()
    created_at = models.DateTimeField(auto_now_add=True)
    created_by = models.ForeignKey(User, on_delete=models.PROTECT)

    class Meta:
        unique_together = [('asset', 'version_number')]
        indexes = [
            models.Index(fields=['asset', 'status']),
            models.Index(fields=['asset', 'valid_from', 'valid_to']),
        ]


class WorkflowVersionEvent(models.Model):  # append-only
    workflow_instance = models.ForeignKey(
        WorkflowInstance, on_delete=models.PROTECT, related_name='version_events')
    workflow_event = models.ForeignKey(
        WorkflowEvent, on_delete=models.PROTECT, related_name='version_events')
    asset_version = models.ForeignKey(
        AssetVersion, on_delete=models.PROTECT, related_name='workflow_events')
    action = models.CharField(max_length=20)
    # created | advanced | published | superseded | withdrawn
    timestamp = models.DateTimeField(auto_now_add=True)
```

Compliance query (version in force on a date):

```python
AssetVersion.objects.get(
    asset=asset,
    status='published',
    valid_from__lte=date,
).filter(Q(valid_to__isnull=True) | Q(valid_to__gt=date))
```

---

## 4. Denormalised aggregation updates

### 4.1 Purpose

`ModuleIdentifier` and `CourseIdentifier` carry denormalised fields
summarising the most recent publication event across all governed objects
anchored to them. These fields support at-a-glance summary pages and module
and course list views without requiring a union query across multiple
document types at read time.

| Field                          | Type                     | Notes                                              |
|--------------------------------|--------------------------|----------------------------------------------------|
| `last_published_at`            | DateTimeField (nullable) | Timestamp of the most recent publication of any    |
|                                |                          | governed object anchored to this identifier        |
| `last_published_document_type` | CharField (nullable)     | `Document.DocumentType` value of that object       |
| `has_in_flight_workflow`       | BooleanField             | True if any anchored document has an open workflow |

On `CourseIdentifier`, two variants of the timestamp are maintained:

| Field                           | Type                     | Notes                                                 |
|---------------------------------|--------------------------|-------------------------------------------------------|
| `last_published_at`             | DateTimeField (nullable) | Any publication in the course's scope (including      |
|                                 |                          | module-level events for modules belonging to this     |
|                                 |                          | course)                                               |
| `course_spec_last_published_at` | DateTimeField (nullable) | Set only when `CourseSpecificationContent` publishes. |
|                                 |                          | Never updated by module-level events.                 |

This distinction allows a course coordinator to see both "anything
changed" and "the course specification itself changed" without
conflating the two.

### 4.2 Update protocol

All three fields are updated by a shared post-transition hook fired
whenever a governed document reaches its publication state. The hook
must never be called directly — it is invoked by the FSM machinery
only.

```python
# workflows/hooks.py

from django.db import transaction
from django.utils.timezone import now


def on_document_published(document):
    """
    Called by the FSM publication transition for every document type.
    Updates denormalised aggregation fields on ModuleIdentifier and,
    where applicable, CourseIdentifier.
    """
    module = resolve_module_identifier(document)  # returns None if not applicable

    if module:
        with transaction.atomic():
            # Lock the module row before updating to prevent lost updates
            # from concurrent publication events on the same module.
            locked = ModuleIdentifier.objects.select_for_update().get(pk=module.pk)
            locked.last_published_at = now()
            locked.last_published_document_type = document.document_type
            locked.save(update_fields=[
                'last_published_at',
                'last_published_document_type',
            ])

        # Propagate to all courses that include this module.
        with transaction.atomic():
            course_ids = CourseModuleMembership.objects.filter(
                module=module
            ).values_list('course_id', flat=True)

            # Lock all affected course rows simultaneously.
            # Rows are locked in pk order by the database to avoid deadlocks.
            CourseIdentifier.objects.select_for_update().filter(
                pk__in=course_ids
            ).update(
                last_published_at=now(),
                last_published_document_type=document.document_type,
            )
```

`resolve_module_identifier` is a registry function in
`documents/registry.py` that maps each `DocumentType` to the FK path
that reaches its `ModuleIdentifier`, if one exists. Document types with
no module anchor (e.g. `CourseSpecificationContent`) return `None`
from this function and are excluded from module-level propagation.
`CourseSpecificationContent` instead updates
`course_spec_last_published_at` directly in its own FSM publication
transition side effect.

### 4.3 Concurrency discipline

`select_for_update()` must be used whenever these fields are written.
It generates a `SELECT ... FOR UPDATE` SQL statement, acquiring a
row-level lock for the duration of the enclosing `transaction.atomic()`
block. The lock is released automatically on commit or rollback.

Rules that must be followed at every call site:

- `select_for_update()` must always be called inside `transaction.atomic()`.
  Calling it outside a transaction has no effect and will raise
  `TransactionManagementError` in development.
- When locking multiple rows (the course propagation case), always
  use a bulk `.filter().update()` rather than locking and saving rows
  individually in a loop. The database acquires locks in a consistent
  internal order, avoiding deadlock.
- Never acquire a lock on `ModuleIdentifier` and `CourseIdentifier`
  in the same `atomic()` block. Use separate blocks as shown above.
  This eliminates the cross-table deadlock risk.
- The FSM publication transition is already wrapped in `atomic()` for
  its own state change and audit event. The hook runs inside that same
  transaction — do not open a new outer `atomic()` around the hook call.

### 4.4 `has_in_flight_workflow` — separate hook pair

`has_in_flight_workflow` on `ModuleIdentifier` and `CourseIdentifier`
is maintained by a dedicated pair of hooks, distinct from
`on_document_published`:

```python
# workflows/hooks.py

def on_workflow_opened(document):
    """
    Called by the FSM when a WorkflowInstance is first created for
    any governed document. Sets has_in_flight_workflow = True on the
    anchored ModuleIdentifier and affected CourseIdentifiers.
    """
    module = resolve_module_identifier(document)
    if module:
        with transaction.atomic():
            ModuleIdentifier.objects.select_for_update().filter(
                pk=module.pk
            ).update(has_in_flight_workflow=True)

        with transaction.atomic():
            course_ids = CourseModuleMembership.objects.filter(
                module=module
            ).values_list('course_id', flat=True)

            CourseIdentifier.objects.select_for_update().filter(
                pk__in=course_ids
            ).update(has_in_flight_workflow=True)


def on_workflow_closed(document):
    """
    Called by the FSM when a WorkflowInstance reaches any terminal
    state (approved, withdrawn, rejected, etc.). Recomputes
    has_in_flight_workflow rather than setting False directly, because
    other documents anchored to the same module may still have open
    workflows.
    """
    module = resolve_module_identifier(document)
    if module:
        has_open = Document.objects.filter(
            module_identifier=module,
        ).exclude(
            current_workflow__isnull=True,
        ).filter(
            current_workflow__state__in=NON_TERMINAL_STATES,
        ).exists()

        with transaction.atomic():
            ModuleIdentifier.objects.select_for_update().filter(
                pk=module.pk
            ).update(has_in_flight_workflow=has_open)

        if not has_open:
            with transaction.atomic():
                course_ids = CourseModuleMembership.objects.filter(
                    module=module
                ).values_list('course_id', flat=True)

                # Only recompute course-level flag if the module
                # flag cleared. If other modules on the course still
                # have open workflows the course flag stays True —
                # recompute per course only when needed.
                for course_id in course_ids:
                    prog_has_open = Document.objects.filter(
                        module_identifier__course_memberships__course_id=course_id,
                    ).exclude(
                        current_workflow__isnull=True,
                    ).filter(
                        current_workflow__state__in=NON_TERMINAL_STATES,
                    ).exists()

                    CourseIdentifier.objects.select_for_update().filter(
                        pk=course_id
                    ).update(has_in_flight_workflow=prog_has_open)
```

**Why `on_workflow_closed` recomputes rather than sets `False`**

Setting `has_in_flight_workflow = False` directly on close would
produce incorrect results when two workflows for the same module close
in rapid succession, or when one of several concurrent open workflows
closes. The recomputation ensures the field always reflects the true
current state regardless of concurrency.

**Course-level recomputation is conditional**

The course flag recomputation only runs when the module flag clears.
If the module still has open workflows, the course necessarily has
at least one open workflow via that module — no recomputation needed.
This avoids an unnecessary cross-module query on the common case where
a module has several workflows closing in sequence.

**Extension checklist obligation**

`on_workflow_opened` and `on_workflow_closed` must be called from the
FSM for every document type, not just publication transitions. This is
a separate checklist item from the `on_document_published` hook. Both
are required for any document type anchored to a `ModuleIdentifier`.

---

## 5. Finite state machine

### 5.1 Principles

- States are Python enums using `auto()` — integer values never appear
  in application code. State is stored and retrieved as `enum.name`.
- Transition graph is a dict-of-dicts in code, not the database.
- Document dependencies are also in code — see §7.2.
- All transition logic executes inside `transaction.atomic()`: state
  change, audit event, task supersession, and task creation are atomic.

### 5.2 Transition dataclass

```python
from dataclasses import dataclass, field
from typing import Callable


@dataclass
class Transition:
    target: StateEnum
    permitted_roles: set[str]
    side_effects: list[Callable] = field(default_factory=list)
    generates_milestone: bool = False  # True on terminal approval transitions
    milestone_label: str = ''
    committee: str | None = None  # committee slug if gated on a meeting
    estimated_days: int | None = None  # processing time if not committee-gated
    is_rejection_branch: bool = False  # marks non-primary paths
```

### 5.3 Example FSM

```python
from enum import Enum, auto


class ModuleSpecState(Enum):
    DRAFT = auto()
    SUBMITTED = auto()
    FACULTY_REVIEW = auto()
    APPROVED = auto()
    WITHDRAWN = auto()


class ModuleSpecFSM:
    DOCUMENT_TYPE = Document.DocumentType.MODULE_SPECIFICATION
    INITIAL_STATE = ModuleSpecState.DRAFT
    TERMINAL_STATES = {ModuleSpecState.APPROVED, ModuleSpecState.WITHDRAWN}

    TRANSITIONS = {
        ModuleSpecState.DRAFT: {
            'submit': Transition(
                target=ModuleSpecState.SUBMITTED,
                permitted_roles={'convenor'},
                side_effects=[create_review_task],
                estimated_days=14,
            ),
        },
        ModuleSpecState.SUBMITTED: {
            'assign_review': Transition(
                target=ModuleSpecState.FACULTY_REVIEW,
                permitted_roles={'department_tlc_chair'},
                side_effects=[create_faculty_review_task],
                estimated_days=7,
            ),
        },
        ModuleSpecState.FACULTY_REVIEW: {
            'approve': Transition(
                target=ModuleSpecState.APPROVED,
                permitted_roles={'faculty_tlc_chair'},
                side_effects=[publish_asset_version],
                generates_milestone=True,
                milestone_label='Module specification approved',
                committee='teaching_and_learning',
            ),
            'refer_back': Transition(
                target=ModuleSpecState.DRAFT,
                permitted_roles={'reviewer'},
                is_rejection_branch=True,
                estimated_days=0,
            ),
        },
    }
```

### 5.4 perform_transition

```python
from django.db import transaction


def perform_transition(workflow_instance, event, actor, actor_role):
    fsm = get_fsm(workflow_instance.workflow_type)
    with transaction.atomic():
        from_state = workflow_instance.state
        new_state = fsm.transition(workflow_instance, event, actor_role)

        workflow_instance.state = new_state.name
        workflow_instance.save(update_fields=['state', 'updated_at'])

        WorkflowEvent.objects.create(
            workflow_instance=workflow_instance,
            event=event,
            from_state=from_state,
            to_state=new_state.name,
            actor=actor,
            actor_role=actor_role,
        )

        workflow_instance.tasks.filter(status='pending').update(
            status='superseded')
        fsm.create_tasks(workflow_instance, new_state)
```

### 5.5 State persistence

```python
# Writing
record.state_col = instance.state.name  # → 'FACULTY_REVIEW'

# Reading
instance.state = ModuleSpecState[record.state_col]
```

---

## 6. Django application structure

### 6.1 App layout

The application is divided into eight Django apps. The dependency
direction is strictly one-way: `registry` imports only from `core`;
`documents` imports from `core` and `registry`; `workflows` imports
from `core`, `registry`, and `documents`; and so on downward. No
upward imports.

| App          | Responsibility                                                   |
|--------------|------------------------------------------------------------------|
| `core`       | `Tenant`, `TenantScopedManager`, `TenantMiddleware`,             |
|              | `TenantMembershipMiddleware`, `ContextVar`, base views           |
| `accounts`   | `TenantMembership`, Okta SSO via `mozilla-django-oidc`,          |
|              | role resolution, `CourseConvenorScope`, `ModuleConvenorScope`    |
| `registry`   | Stable, administrative, non-governed reference models.           |
|              | No workflows, no versioning, no FSM. Deactivated rather than     |
|              | deleted. Includes: `FacultyIdentifier`, `SchoolIdentifier`,      |
|              | `DepartmentIdentifier`, `CourseIdentifier`,                      |
|              | `ModuleIdentifier`, `AssessmentSlotIdentifier`,                  |
|              | `FHEQLevel`, `CommitteeIdentifier`, `AccreditingBodyIdentifier`, |
|              | `CourseAccreditationScope`, `RegulatoryDocument`,                |
|              | `PeriodIdentifier`, `PeriodWeekIdentifier`                       |
| `documents`  | `Document` envelope, all `[Type]Content` content models,         |
|              | `documents/registry.py` (type → content model + relation name    |
|              | mapping, `resolve_module_identifier`), document-type FSM         |
|              | definitions                                                      |
| `workflows`  | `WorkflowInstance`, `WorkflowEvent`, `Task`,                     |
|              | `perform_transition`, FSM base class, `DOCUMENT_DEPENDENCIES`,   |
|              | publication hooks (`on_document_published`)                      |
| `assets`     | `AssetVersion`, `WorkflowVersionEvent`, publication logic        |
| `scheduling` | `PublicationTarget`, `ScheduledMilestone`, `MilestoneOverlap`,   |
|              | `TransitionDuration` — scaffold only in Phase 1                  |
| `api`        | Internal JSON endpoints for HTMX and pipeline visualisation      |

**Migration order:** `core` → `accounts` → `registry` → `documents`
→ `workflows` → `assets` → `scheduling` → `api`. Every content model
in `documents` carries FKs into `registry`; the registry must be
migrated first.

**Validate at startup:** every `Document.DocumentType` value must have
a registered content model in `documents/registry.py` and a registered
FSM in `workflows/fsm/`. A startup check should raise `ImproperlyConfigured`
if any are missing.

---

### 6.2 The registry app

The `registry` app is a distinct population from the `documents` app.
The distinction is architectural, not just organisational:

- **Registry models** answer "what exists?" They are stable identity
  records — created by administrators, deactivated rather than deleted,
  carrying no `WorkflowInstance` and no `AssetVersion`. They form the
  fixed roots of the document dependency graph. All cross-document FKs
  point at registry models, never at other document models; this is
  what keeps the dependency graph acyclic.

- **Document models** answer "what has been decided, and when, and by
  whom?" They have lifecycles, audit trails, FSM states, and dependency
  relationships declared in `DOCUMENT_DEPENDENCIES`.

Most registry models inherit from a common abstract base:

```python
class RegistryModel(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.PROTECT)
    code = models.CharField(max_length=50)
    name = models.CharField(max_length=255)
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    objects = TenantScopedManager()
    all_tenants = TenantScopedManager(require_tenant=False)

    class Meta:
        abstract = True
        unique_together = [('tenant', 'code')]
```

Not every registry model fits this exactly — `PeriodIdentifier` and
`PeriodWeekIdentifier` use sequence-managed ordering rather than a
`code` field; `FHEQLevel` and `AccreditingBodyIdentifier` are global
rather than tenant-scoped — but the pattern is the baseline.

---

### 6.3 The accounts app and scoping layer

`accounts` owns `TenantMembership` (the global role grant) and the two
scoping models that express ongoing institutional responsibility:

- `CourseConvenorScope` — scopes a person to a course with a
  `valid_from` / `valid_to` validity window. Uses a partial
  `UniqueConstraint` to enforce at most one active record per
  `(course, user, role)` combination.
- `ModuleConvenorScope` — same pattern for modules.

These are append-only records: convenorships are ended by setting
`valid_to`, not by deletion. Users with an active scoping record must
hold at least the `author` role in `TenantMembership` so that they
can initiate workflows in their own right.

`CourseAccreditationScope` (scoping an `AccreditingBodyIdentifier` to a
course with a validity window) follows the same structural pattern
and also lives in `accounts` as part of the scoping layer.

---

### 6.4 Settings split

- `settings/base.py` — installed apps, middleware stack, database,
  Celery, logging
- `settings/dev.py` — `DEBUG=True`, Waitress, local Redis
- `settings/prod.py` — Gunicorn, secure cookies, HSTS

### 6.5 Middleware order

```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'core.middleware.TenantMiddleware',  # resolves tenant from subdomain, sets ContextVar
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'core.middleware.TenantMembershipMiddleware',  # validates membership, sets request.tenant_roles
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
]
```

---

## 7. Celery configuration

### 7.1 Named queues

| Queue           | Purpose                          | Concurrency |
|-----------------|----------------------------------|-------------|
| `transitions`   | FSM transition side-effects      | 4           |
| `notifications` | Email dispatch                   | 4           |
| `scheduling`    | Schedule recomputation           | 2           |
| `llm`           | LLM inference tasks (deferred)   | 1           |
| `reports`       | PDF/report generation (deferred) | 2           |

### 7.2 Tenant context in tasks

Pass `tenant_id` explicitly to all tasks. Restore `ContextVar` via a
base task class:

```python
class TenantTask(celery.Task):
    def __call__(self, tenant_id, *args, **kwargs):
        tenant = Tenant.objects.get(pk=tenant_id)
        token = _current_tenant.set(tenant)
        try:
            return super().__call__(*args, **kwargs)
        finally:
            _current_tenant.reset(token)
```

---

## Part 2 — Planned subsystems

Sections 7–9 describe subsystems built in later phases. Do not generate
implementation code. Ensure referenced fields exist on core models.

---

## 8. Document dependency graph and scheduling

### 8.1 Overview

Publication targets drive backwards scheduling across the document
dependency graph to generate initiation tasks and deadlines.
Built on top of NetworkX.

### 8.2 Dependencies in code

```python
# workflows/dependencies.py
DOCUMENT_DEPENDENCIES = {
    DocumentType.PROGRAMME_SPECIFICATION: [
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
            requires_type=DocumentType.LEARNING_OUTCOME,
            requires_state=LearningOutcomeState.APPROVED,
            resolution='all',
            stale_limit=None,  # never auto-initiate — convenor decides
        ),
    ],
}
```

Dependencies are in code (not the database): they change only on
deployment, require developer review, and benefit from version control.
Validate at startup: every `DocumentType` value must have a registered
content model and a registered FSM.

### 8.3 Scaffold models (Phase 1 — fields only, no logic)

```python
class PublicationTarget(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.PROTECT)
    label = models.CharField(max_length=255)
    description = models.TextField(blank=True)
    target_document_type = models.CharField(max_length=100)
    target_state = models.CharField(max_length=100)
    target_date = models.DateField()
    owner = models.ForeignKey(User, on_delete=models.PROTECT)
    course = models.ForeignKey(
        CourseIdentifier, null=True, blank=True,
        on_delete=models.PROTECT,
        related_name='publication_targets')
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)


class ScheduledMilestone(models.Model):
    publication_target = models.ForeignKey(
        PublicationTarget, on_delete=models.CASCADE,
        related_name='milestones')
    document = models.ForeignKey(
        Document, null=True, blank=True,
        on_delete=models.SET_NULL,
        related_name='milestones')
    document_type = models.CharField(max_length=100)
    required_state = models.CharField(max_length=100)
    must_be_complete_by = models.DateField()
    must_be_initiated_by = models.DateField()
    is_critical_path = models.BooleanField(default=False)
    at_risk = models.BooleanField(default=False)
    computed_at = models.DateTimeField(auto_now=True)


class MilestonePublicationTarget(models.Model):
    milestone = models.ForeignKey(
        ScheduledMilestone, on_delete=models.CASCADE,
        related_name="additional_targets")
    publication_target = models.ForeignKey(
        PublicationTarget, on_delete=models.CASCADE,
        related_name="shared_milestones")
    linked_by = models.ForeignKey(
        User, on_delete=models.PROTECT)
    linked_at = models.DateTimeField(auto_now_add=True)
    rationale = models.TextField(blank=True)


class MilestoneOverlap(models.Model):
    class Character(models.TextChoices):
        COMPATIBLE = 'compatible'
        INCOMPATIBLE = 'incompatible'
        DEADLINE_RISK = 'deadline_risk'

    class Resolution(models.TextChoices):
        PENDING = 'pending'
        LINKED = 'linked'
        MONITORING = 'monitoring'
        SEPARATE = 'separate'
        DISMISSED = 'dismissed'

    milestone_a = models.ForeignKey(
        ScheduledMilestone, on_delete=models.CASCADE,
        related_name='overlaps_as_a')
    milestone_b = models.ForeignKey(
        ScheduledMilestone, null=True, blank=True,
        on_delete=models.PROTECT,
        related_name='overlaps_as_b')
    workflow_instance = models.ForeignKey(
        WorkflowInstance, null=True, blank=True,
        on_delete=models.PROTECT,
        related_name='milestone_overlaps')
    character = models.CharField(max_length=20, choices=Character.choices)
    explanation = models.TextField()
    decision_required_by = models.DateField(null=True, blank=True)
    resolution = models.CharField(
        max_length=20, choices=Resolution.choices,
        default=Resolution.PENDING)
    resolved_milestone = models.ForeignKey(
        ScheduledMilestone, null=True, blank=True,
        on_delete=models.PROTECT,
        related_name='resolved_from_overlaps')
    resolved_by = models.ForeignKey(
        User, null=True, blank=True,
        on_delete=models.PROTECT)
    resolved_at = models.DateTimeField(null=True, blank=True)
    resolution_rationale = models.TextField(blank=True)
    computed_at = models.DateTimeField(auto_now=True)

    class Meta:
        constraints = [
            models.CheckConstraint(
                check=(
                        Q(milestone_b__isnull=False, workflow_instance__isnull=True) |
                        Q(milestone_b__isnull=True, workflow_instance__isnull=False)
                ),
                name='milestone_overlap_exactly_one_counterpart',
            ),
            models.CheckConstraint(
                check=(
                        Q(resolution__in=['pending', 'monitoring', 'separate', 'dismissed'],
                          resolved_milestone__isnull=True) |
                        Q(resolution='linked',
                          resolved_milestone__isnull=False)
                ),
                name='milestone_overlap_resolved_milestone_only_when_linked',
            ),
            models.UniqueConstraint(
                fields=['milestone_a', 'milestone_b'],
                condition=Q(milestone_b__isnull=False),
                name='unique_milestone_overlap_pair',
            ),
            models.UniqueConstraint(
                fields=['milestone_a', 'workflow_instance'],
                condition=Q(workflow_instance__isnull=False),
                name='unique_milestone_workflow_overlap',
            ),
        ]

    def clean(self):
        has_b = self.milestone_b_id is not None
        has_workflow = self.workflow_instance_id is not None

        if has_b == has_workflow:
            raise ValidationError(
                'Exactly one of milestone_b or workflow_instance must be set.'
            )

        if self.resolution == self.Resolution.LINKED and not self.resolved_milestone_id:
            raise ValidationError(
                'resolved_milestone must be set when resolution is linked.'
            )

        if self.resolution != self.Resolution.LINKED and self.resolved_milestone_id:
            raise ValidationError(
                'resolved_milestone must be null unless resolution is linked.'
            )


class CommitteeMeeting(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.PROTECT)
    committee = models.ForeignKey(
        'CommitteeIdentifier', on_delete=models.PROTECT)
    meeting_date = models.DateField()
    submission_deadline = models.DateField()
    academic_year = models.CharField(max_length=9)  # e.g. '2028/29'


class TransitionDuration(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.PROTECT)
    workflow_type = models.CharField(max_length=100)
    transition_event = models.CharField(max_length=100)
    duration_type = models.CharField(max_length=20)  # days | committee
    estimated_days = models.PositiveIntegerField(null=True, blank=True)
    committee = models.ForeignKey(
        'CommitteeIdentifier', null=True, blank=True,
        on_delete=models.PROTECT)
```

> **Compatibility note:** `Document.due_date` and `Task.due_date` are
> updated by the scheduling subsystem. Both fields must be nullable on
> the skeleton models — they are.

---

## 9. Frontend and visualisation

### 9.1 Pipeline visualisation

Workflow stages are rendered as a horizontal pipeline.
Reference implementation: `static/js/workflow_pipeline.js`.

- Stage states: `completed` (green), `current` (blue + "Current"
  badge), `pending` (grey)
- Branch forks: approve/reject pill labels. Maximum one level.
- Progress bar above, legend below.
- Reinitialise after HTMX swap: listen for `htmx:afterSwap`.
- Stage nodes accept `required_documents` array for Bootstrap popover.
- All colours via Bootstrap CSS variables for theme consistency.

### 9.2 Pipeline JSON contract

Django views produce this shape. The renderer is a pure function of it.

```json
{
  "title": "string",
  "type": "string",
  "current_stage": "string",
  "stages": [
    {
      "id": "string",
      "label": "string",
      "sublabel": "string",
      "status": "completed | current | pending",
      "required_documents": [
        {
          "type": "string",
          "label": "string",
          "state": "string",
          "satisfied": true,
          "url": "/documents/42/"
        }
      ]
    }
  ],
  "transitions": [
    {
      "from": "string",
      "to": "string",
      "label": "string",
      "active": true
    },
    {
      "from": "string",
      "branches": [
        {
          "to": "string",
          "label": "string",
          "type": "approve | reject",
          "active": false
        }
      ]
    }
  ]
}
```

---

## 10. LLM integration (deferred — Phase 5)

All LLM tasks are Celery tasks on the `llm` queue. They are async,
never blocking the request cycle. Outputs are always persisted to
`LLMResult` before being surfaced. The human decision point always
follows the LLM output.

| Task type              | Trigger                                  |
|------------------------|------------------------------------------|
| `committee_screening`  | Submission deadline passes for a meeting |
| `impact_analysis`      | Course-level change initiated            |
| `form_draft`           | Initiation task created for a convenor   |
| `pre_submission_check` | Document submitted for review            |
| `regulatory_check`     | Document reaches review state            |
| `schedule_summary`     | PublicationTarget created or updated     |

```python
class LLMResult(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.PROTECT)
    task_type = models.CharField(max_length=50)
    subject_type = models.ForeignKey(ContentType, on_delete=models.PROTECT)
    subject_id = models.PositiveIntegerField()
    model_name = models.CharField(max_length=100)
    prompt_hash = models.CharField(max_length=64)
    output = models.JSONField()
    generated_at = models.DateTimeField(auto_now_add=True)
    reviewed_by = models.ForeignKey(
        User, null=True, blank=True, on_delete=models.PROTECT)
    reviewed_at = models.DateTimeField(null=True, blank=True)
    accepted = models.BooleanField(null=True)  # None = not yet reviewed
```

**Model:** Qwen 2.5 32B at Q4 quantisation via Ollama.
**Data sovereignty:** No content is sent to external APIs. Hard
architectural constraint.

Failure handling: retry with exponential backoff (3 attempts).
Absence of an LLM result never blocks a workflow transition.

---

## Extension checklist

### Adding a new document type

1. Add value to `Document.DocumentType`
2. Create content model in `documents/models/`, following
   `ModuleSpecificationContent`
3. Register in `documents/registry.py` — including the FK path to
   `ModuleIdentifier` if the document type is anchored to a module
   (see `resolve_module_identifier` in `workflows/hooks.py`)
4. Define FSM in `workflows/fsm/` — include `generates_milestone` and
   `estimated_days` on all primary pathway transitions
5. Declare document dependencies in `workflows/dependencies.py`
6. **Register the publication hook** — ensure the FSM publication
   transition calls `on_document_published(document)`. If this step
   is omitted, `ModuleIdentifier.last_published_at` and
   `CourseIdentifier.last_published_at` will not reflect changes
   made via this document type. This is a silent correctness failure,
   not a runtime error.
7. **Register the workflow lifecycle hooks** — if the document type
   is anchored to a `ModuleIdentifier`, ensure the FSM calls
   `on_workflow_opened(document)` when a `WorkflowInstance` is first
   created, and `on_workflow_closed(document)` when any
   `WorkflowInstance` reaches a terminal state. If either hook is
   omitted, `ModuleIdentifier.has_in_flight_workflow` and
   `CourseIdentifier.has_in_flight_workflow` will be incorrect.
   This is a silent correctness failure, not a runtime error. Add a
   test that opening and closing a workflow on a document of this type
   correctly sets and clears `has_in_flight_workflow` on the associated
   `ModuleIdentifier`.
8. Add admin registration
9. Generate and apply migration
10. Add tests following `tests/documents/test_module_spec.py`, including
    a test that publication of a document of this type updates
    `last_published_at` on the associated `ModuleIdentifier`

### Adding a new workflow form

Forms are `Document` records with `classification=WORKFLOW_FORM`.
Follow the document type checklist above, then additionally:

1. Add `target_document` FK on the content model pointing to the
   document the form acts upon
2. Define the FSM side effect that creates or versions the target
   document on completion
3. Add form view and template following existing form patterns

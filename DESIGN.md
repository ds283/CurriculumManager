# DESIGN.md — Curriculum Governance Workflow System

**Read this file in full before writing any code.**
All architectural decisions here take precedence over inferred conventions.

Part 1 (Sections 1–6) drives immediate skeleton generation.
Part 2 (Sections 7–9) describes planned subsystems — design for
compatibility but do not generate implementation code for those sections.

---

## Part 1 — Generate now

---

## 1. Technology stack

| Component      | Choice                           | Notes                                                 |
|----------------|----------------------------------|-------------------------------------------------------|
| Language       | Python 3.12+                     |                                                       |
| Framework      | Django 5.x                       | Not Flask. Use integrated migrations, admin, ORM.     |
| Database       | PostgreSQL 16+                   | jsonb, recursive CTEs, transactional DDL required.    |
| Task queue     | Celery 5.x                       | Redis broker. Named queues — see §6.                  |
| Cache / broker | Redis                            |                                                       |
| Auth           | Okta SSO                         | django-allauth with Okta provider. Custom middleware. |
| Frontend       | HTMX + Bootstrap 5               | No React SPA. Django templates throughout.            |
| WSGI           | Waitress (dev) / Gunicorn (prod) | Set per environment in settings split.                |

Key packages: `networkx` (graph algorithms), `pydantic` (JSON schema
validation), `ollama` Python SDK (deferred), `django-guardian`
(object-level permissions — install but do not configure until §5 is
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
        MODULE_SPECIFICATION = 'module_spec'
        PROGRAMME_SPECIFICATION = 'programme_spec'
        ASSESSMENT_BRIEF = 'assessment_brief'
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
    level = models.CharField(max_length=10)  # FHEQ level
    subject_area = models.CharField(max_length=100)
```

Accessing content (single indexed join, no N+1):

```python
doc = Document.objects.select_related('module_spec').get(pk=pk)
content = doc.module_spec
```

**Extension pattern** — to add a new document type:

1. Add value to `Document.DocumentType`
2. Create content model in `documents/models/`
3. Register in `documents/registry.py`
4. Define FSM in `workflows/fsm/`
5. Add admin registration
6. Generate and apply migration
7. Add tests following `tests/documents/test_module_spec.py`

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

## 4. Finite state machine

### 4.1 Principles

- States are Python enums using `auto()` — integer values never appear
  in application code. State is stored and retrieved as `enum.name`.
- Transition graph is a dict-of-dicts in code, not the database.
- Document dependencies are also in code — see §7.2.
- All transition logic executes inside `transaction.atomic()`: state
  change, audit event, task supersession, and task creation are atomic.

### 4.2 Transition dataclass

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

### 4.3 Example FSM

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
                permitted_roles={'author'},
                side_effects=[create_review_task],
                estimated_days=14,
            ),
        },
        ModuleSpecState.SUBMITTED: {
            'assign_review': Transition(
                target=ModuleSpecState.FACULTY_REVIEW,
                permitted_roles={'coordinator'},
                side_effects=[create_faculty_review_task],
                estimated_days=7,
            ),
        },
        ModuleSpecState.FACULTY_REVIEW: {
            'approve': Transition(
                target=ModuleSpecState.APPROVED,
                permitted_roles={'reviewer'},
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

### 4.4 perform_transition

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

### 4.5 State persistence

```python
# Writing
record.state_col = instance.state.name  # → 'FACULTY_REVIEW'

# Reading
instance.state = ModuleSpecState[record.state_col]
```

---

## 5. Django application structure

### 5.1 App layout

| App          | Responsibility                                                      |
|--------------|---------------------------------------------------------------------|
| `core`       | Tenant, TenantScopedManager, middleware, ContextVar, base views     |
| `documents`  | Document envelope, content models, registry, document-type FSMs     |
| `workflows`  | WorkflowInstance, WorkflowEvent, Task, perform_transition, FSM base |
| `assets`     | Asset, AssetVersion, WorkflowVersionEvent, publication logic        |
| `scheduling` | PublicationTarget, ScheduledMilestone — scaffold only in Phase 1    |
| `accounts`   | TenantMembership, Okta SSO, role resolution                         |
| `api`        | JSON endpoints for HTMX and pipeline visualisation                  |

### 5.2 Settings split

- `settings/base.py` — installed apps, middleware, database, Celery, logging
- `settings/dev.py` — DEBUG=True, Waitress, local Redis
- `settings/prod.py` — Gunicorn, secure cookies, HSTS

### 5.3 Middleware order

```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'core.middleware.TenantMiddleware',  # resolves tenant, sets ContextVar
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'core.middleware.TenantMembershipMiddleware',  # sets request.tenant_roles
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
]
```

---

## 6. Celery configuration

### 6.1 Named queues

| Queue           | Purpose                          | Concurrency |
|-----------------|----------------------------------|-------------|
| `transitions`   | FSM transition side-effects      | 4           |
| `notifications` | Email dispatch                   | 4           |
| `scheduling`    | Schedule recomputation           | 2           |
| `llm`           | LLM inference tasks (deferred)   | 1           |
| `reports`       | PDF/report generation (deferred) | 2           |

### 6.2 Tenant context in tasks

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

## 7. Document dependency graph and scheduling

### 7.1 Overview

Publication targets drive backwards scheduling across the document
dependency graph to generate initiation tasks and deadlines.
Built on top of NetworkX.

### 7.2 Dependencies in code

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
}
```

Dependencies are in code (not the database): they change only on
deployment, require developer review, and benefit from version control.
Validate at startup: every `DocumentType` value must have a registered
content model and a registered FSM.

### 7.3 Scaffold models (Phase 1 — fields only, no logic)

```python
class PublicationTarget(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.PROTECT)
    label = models.CharField(max_length=255)
    description = models.TextField(blank=True)
    target_document_type = models.CharField(max_length=100)
    target_state = models.CharField(max_length=100)
    target_date = models.DateField()
    owner = models.ForeignKey(User, on_delete=models.PROTECT)
    programme = models.ForeignKey(
        Document, null=True, blank=True,
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
    link_status = models.CharField(max_length=20, default='unmatched')
    # unmatched | candidate | linked | separate | deferred
    link_decided_by = models.ForeignKey(
        User, null=True, blank=True, on_delete=models.PROTECT)
    link_decided_at = models.DateTimeField(null=True, blank=True)
    link_rationale = models.TextField(blank=True)
    computed_at = models.DateTimeField(auto_now=True)


class MilestoneOverlap(models.Model):
    milestone_a = models.ForeignKey(
        ScheduledMilestone, on_delete=models.CASCADE,
        related_name='overlaps_as_a')
    milestone_b = models.ForeignKey(
        ScheduledMilestone, on_delete=models.CASCADE,
        related_name='overlaps_as_b')
    character = models.CharField(max_length=20)
    # compatible | incompatible | deadline_risk
    explanation = models.TextField()
    decision_required_by = models.DateField(null=True, blank=True)
    computed_at = models.DateTimeField(auto_now=True)

    class Meta:
        unique_together = [('milestone_a', 'milestone_b')]


class CommitteeMeeting(models.Model):
    committee = models.ForeignKey(
        'Committee', on_delete=models.PROTECT)
    meeting_date = models.DateField()
    submission_deadline = models.DateField()
    academic_year = models.CharField(max_length=9)  # e.g. '2028/29'


class TransitionDuration(models.Model):
    workflow_type = models.CharField(max_length=100)
    transition_event = models.CharField(max_length=100)
    duration_type = models.CharField(max_length=20)  # days | committee
    estimated_days = models.PositiveIntegerField(null=True, blank=True)
    committee = models.ForeignKey(
        'Committee', null=True, blank=True,
        on_delete=models.PROTECT)
```

> **Compatibility note:** `Document.due_date` and `Task.due_date` are
> updated by the scheduling subsystem. Both fields must be nullable on
> the skeleton models — they are.

---

## 8. Frontend and visualisation

### 8.1 Pipeline visualisation

Workflow stages are rendered as a horizontal pipeline.
Reference implementation: `static/js/workflow_pipeline.js`.

- Stage states: `completed` (green), `current` (blue + "Current"
  badge), `pending` (grey)
- Branch forks: approve/reject pill labels. Maximum one level.
- Progress bar above, legend below.
- Reinitialise after HTMX swap: listen for `htmx:afterSwap`.
- Stage nodes accept `required_documents` array for Bootstrap popover.
- All colours via Bootstrap CSS variables for theme consistency.

### 8.2 Pipeline JSON contract

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

## 9. LLM integration (deferred — Phase 5)

All LLM tasks are Celery tasks on the `llm` queue. They are async,
never blocking the request cycle. Outputs are always persisted to
`LLMResult` before being surfaced. The human decision point always
follows the LLM output.

| Task type              | Trigger                                  |
|------------------------|------------------------------------------|
| `committee_screening`  | Submission deadline passes for a meeting |
| `impact_analysis`      | Programme-level change initiated         |
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
3. Register in `documents/registry.py`
4. Define FSM in `workflows/fsm/` — include `generates_milestone` and
   `estimated_days` on all primary pathway transitions
5. Declare document dependencies in `workflows/dependencies.py`
6. Add admin registration
7. Generate and apply migration
8. Add tests following `tests/documents/test_module_spec.py`

### Adding a new workflow form

Forms are `Document` records with `classification=WORKFLOW_FORM`.
Follow the document type checklist, then additionally:

1. Add `target_document` FK on the content model pointing to the
   document the form acts upon
2. Define the FSM side effect that creates or versions the target
   document on completion
3. Add form view and template following existing form patterns

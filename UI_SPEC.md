# CurriculumManager — UI Specification

**Version 1.0 — derived from approved mockups**
**Include this file as context with every UI-related Claude Code prompt.**

---

## 1. How to use this document

This document defines the visual and structural contract for all CurriculumGov templates. Claude Code must follow it
precisely. When implementing any template or component:

1. Extend `base.html` unless the template is a standalone page
2. Use only the components defined in §5 — do not invent new component patterns
3. Apply only the tokens defined in §2 — do not introduce new colours or type sizes
4. Implement interactive behaviour using HTMX unless noted otherwise — no custom JavaScript except for trivial UI
   toggles
5. Consult the layout rules in §4 before placing any element

If a design need arises that is not covered by this document, stop and flag it rather than inventing a solution.

---

## 2. Design tokens

All tokens are defined as CSS custom properties on `:root` in `static/css/tokens.css`. Reference them by variable name
throughout — never use raw hex values in templates or component CSS.

### 2.1 Colour palette

#### Backgrounds

| Token   | Value     | Use                                                               |
|---------|-----------|-------------------------------------------------------------------|
| `--bg`  | `#ffffff` | Primary surface (cards, sidebar, topbar)                          |
| `--bg2` | `#f7f8fa` | Secondary surface (metric cards, table headers, read-only inputs) |
| `--bg3` | `#f0f2f5` | Page background                                                   |

#### Borders

| Token       | Value     | Use                                                       |
|-------------|-----------|-----------------------------------------------------------|
| `--border`  | `#e2e5ea` | Standard dividers, card edges, input borders              |
| `--borderL` | `#edf0f4` | Subtle internal dividers (table rows, card body sections) |

#### Text

| Token  | Value     | Use                                                           |
|--------|-----------|---------------------------------------------------------------|
| `--t1` | `#111827` | Primary text — headings, values, labels                       |
| `--t2` | `#6b7280` | Secondary text — metadata, descriptions, sidebar items        |
| `--t3` | `#9ca3af` | Tertiary text — section labels, placeholders, disabled states |

#### Semantic colours (each has three tones: text, light bg, mid accent)

| Semantic | Text token           | Light bg              | Mid accent            | Use                                  |
|----------|----------------------|-----------------------|-----------------------|--------------------------------------|
| Blue     | `--blue` `#185FA5`   | `--blueL` `#E6F1FB`   | `--blueM` `#85B7EB`   | Primary actions, active states, info |
| Green    | `--green` `#3B6D11`  | `--greenL` `#EAF3DE`  | `--greenM` `#639922`  | Published / approved / complete      |
| Amber    | `--amber` `#854F0B`  | `--amberL` `#FAEEDA`  | `--amberM` `#FAC775`  | In review / warning / due soon       |
| Red      | `--red` `#A32D2D`    | `--redL` `#FCEBEB`    | —                     | Blocked / urgent / overdue           |
| Purple   | `--purple` `#5B21B6` | `--purpleL` `#EDE9FE` | `--purpleM` `#A78BFA` | AI-generated content exclusively     |
| Teal     | `--tealD` `#0F6E56`  | `--tealL` `#E1F5EE`   | —                     | Programme coordinator role accents   |

**Rule:** Purple is reserved exclusively for AI-related UI elements. Do not use it for any other purpose.

### 2.2 Typography

Base font stack: `-apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif`
Monospace stack: `'Courier New', Courier, monospace` — used for module codes, slot codes, reference numbers, outcome
codes only.

| Role                      | Size    | Weight | Colour   | Notes                                             |
|---------------------------|---------|--------|----------|---------------------------------------------------|
| Page title                | 15–16px | 600    | `--t1`   | One per page                                      |
| Card title                | 12px    | 600    | `--t1`   | Card header                                       |
| Section label (uppercase) | 10px    | 600    | `--t3`   | Letter-spacing: 0.07em, text-transform: uppercase |
| Body text                 | 12–13px | 400    | `--t1`   | Form fields, descriptions                         |
| Secondary text / metadata | 11–12px | 400    | `--t2`   | Timestamps, sub-labels                            |
| Tertiary text             | 10–11px | 400    | `--t3`   | Hints, disclaimers, counts                        |
| Monospace codes           | 11px    | 700    | `--t2`   | Module codes, reference numbers                   |
| Badge / pill text         | 10px    | 600    | Semantic | See §5.1                                          |

### 2.3 Spacing

| Token                     | Value            | Use                                                         |
|---------------------------|------------------|-------------------------------------------------------------|
| `--r`                     | `8px`            | Standard border-radius (cards, larger components)           |
| `--rsm`                   | `5px`            | Small border-radius (pills, badges, buttons, table corners) |
| Content padding           | `16–20px`        | Main content area, card bodies                              |
| Card header padding       | `10px 14px`      | `.card-h`                                                   |
| Card body padding         | `10px 14px 12px` | `.card-b`                                                   |
| Sidebar item padding      | `6px 14px`       | `.nav-item`                                                 |
| Gap between cards         | `10–12px`        | Grid and flex gaps                                          |
| Gap between page sections | `14px`           | Main content flex gap                                       |

### 2.4 Elevation

Only one level of shadow is used throughout:

```css
box-shadow:

0
1
px

3
px

rgba
(
0
,
0
,
0
,
0.04
)
;
```

Applied to: `.card`, `.spec-card`, `.prog-card`. Not applied to sidebar, topbar, or metric cards.

---

## 3. Page shell

Every page uses a fixed shell that does not scroll. Scrolling happens only within the `.content` region.

```
┌─────────────────────────────────────────────────┐  height: 48px, flex-shrink: 0
│  TOPBAR                                         │  bg: --bg, border-bottom: --border
├──────────┬──────────────────────────────────────┤
│          │  [optional: page-head bar]            │  flex-shrink: 0
│          ├──────────────────────────────────────┤
│ SIDEBAR  │  [optional: tabs bar]                 │  flex-shrink: 0
│ 196px    ├──────────────────────────────────────┤
│ bg:--bg  │  CONTENT (scrollable)                 │  flex: 1, overflow-y: auto
│ border-r │                                       │  padding: 16px 20px
│          │                                       │
└──────────┴──────────────────────────────────────┘
```

**Shell CSS:**

```css
.shell {
    display: flex;
    flex-direction: column;
    height: 100vh;
    max-width: 1300px;
    margin: 0 auto;
}

.body {
    display: flex;
    flex: 1;
    min-height: 0;
}
```

**Rule:** Never make the shell, sidebar, or topbar scroll. Never put `overflow: auto` on the body wrapper. Only
`.content` scrolls.

### 3.1 Topbar

Height: 48px. Fixed elements left to right:

- Brand name (13px, 600, `--t1`, letter-spacing: -0.02em)
- 1px divider (height: 16px)
- Context subtitle — tenant or section name (12px, `--t2`)
- `flex: 1` spacer
- Optional: view switcher (segmented control, see §5.8)
- `flex: 1` spacer
- User avatar (28×28px circle) + user name (12px, `--t2`) + role badge

### 3.2 Sidebar

Width: 196px. Structure:

- Section labels: `.ns` — 10px, 600, `--t3`, uppercase, 0.07em spacing, padding: 8px 14px 3px
- Nav items: `.ni` — 12px, `--t2`, padding: 6px 14px, 2px left border (transparent default)
- Active item: left border colour matches role accent, background at 13% opacity, font-weight: 500
- Dot indicator: 6×6px circle, `--border` default, accent colour when active
- Badges: pushed right with `margin-left: auto` — see §5.2

**Role accent colours for active nav items:**

- Convenor / faculty member: `--blue`
- Programme coordinator: `--tealD`
- Committee / TLC chair: `--purple`

### 3.3 Three-column layout (document edit view)

Some views use a three-column layout: context panel (260px) + form area (flex: 1). The context panel is a fixed-width
left panel distinct from the navigation sidebar. It appears only on document creation and edit views.

---

## 4. Layout patterns

### 4.1 Dashboard content layout

Top of content area, in order:

1. Page header (`.ph`) — title (16px, 600) + subtitle (12px, `--t2`)
2. Metric card row — 4-column grid, gap: 10px
3. Primary two-column section — grid `1.4fr 1fr`, gap: 12px
4. Full-width section (workflow pipeline, publication targets, etc.)
5. Secondary two-column section

**Rule:** Always lead with metrics, then the most action-requiring content (tasks), then status content (module list,
committee dates), then supporting content (timeline, pipeline detail).

### 4.2 List view layout

1. Page head bar (outside scroll area) — title + count + tab strip
2. Toolbar bar (outside scroll area) — filter pills + sort control
3. Scrollable list area — cards or grouped tables

### 4.3 Form view layout

Left context panel (260px, fixed) + right form area (flex). Form area structure:

1. Form header bar — title, status pill, task reference
2. AI banner (if AI pre-fill is present)
3. Scrollable form body — form sections
4. Actions footer bar — buttons + submission note

### 4.4 Committee view layout

1. Meeting header bar — title, status pill, metadata row
2. Tab strip — Agenda / AI Screening / Milestones
3. Scrollable tab content

### 4.5 Two-column grid

```css
.two {
    display: grid;
    grid-template-columns: 1.4fr 1fr;
    gap: 12px;
    margin-bottom: 12px;
}
```

Use ratio `1.4fr 1fr` when the left column contains a task list or primary action content. Use `1fr 1fr` for
equal-weight columns (e.g. two committee date lists).

---

## 5. Components

Components are implemented as Django `{% include %}` partials in `templates/components/`. Each include receives context
variables as named parameters. Do not inline component markup in feature templates.

### 5.1 Status pill

```html
{% include "components/pill.html" with state="published" %}
```

| State          | Background  | Text colour | Label          |
|----------------|-------------|-------------|----------------|
| `published`    | `--greenL`  | `--green`   | Published      |
| `in_review`    | `--amberL`  | `--amber`   | In review      |
| `blocked`      | `--redL`    | `--red`     | Blocked        |
| `in_workflow`  | `--amberL`  | `--amber`   | In workflow    |
| `review_due`   | `--amberL`  | `--amber`   | Review due     |
| `draft`        | `--amberL`  | `--amber`   | Draft          |
| `upcoming`     | `--bg2`     | `--t2`      | Upcoming       |
| `at_risk`      | `--amberL`  | `--amber`   | At risk        |
| `ai_draft`     | `--purpleL` | `--purple`  | ✦ AI draft     |
| `ai_generated` | `--purpleL` | `--purple`  | ✦ AI generated |

Font: 10px, 600, border-radius: 10px, padding: 2px 8px, display: inline-flex.
Custom labels can override the default via `label="..."`.

### 5.2 Sidebar badge

Pushed right with `margin-left: auto`. Three colour variants:

| Variant  | Background  | Text       | Use                   |
|----------|-------------|------------|-----------------------|
| `blue`   | `#B5D4F4`   | `#0C447C`  | Informational count   |
| `amber`  | `--amberM`  | `#633806`  | Warning count         |
| `red`    | `#F7C1C1`   | `#791F1F`  | Action required count |
| `purple` | `--purpleL` | `--purple` | AI flags              |

Font: 10px, 600, border-radius: 10px, padding: 1px 6px.

### 5.3 Metric card

```html
{% include "components/metric_card.html" with label="Open workflows" value="3" sub="1 awaiting you" value_colour="blue" %}
```

Structure: label (11px, `--t2`) / value (22px, 600, `--t1` or semantic colour) / sub-label (10px, `--t3`).
Background: `--bg2`. Border: `--borderL`. No shadow. Border-radius: `--r`.
`value_colour` maps to semantic colour tokens: `blue`, `green`, `amber`, `red`, or omit for `--t1`.

### 5.4 Task card

```html
{% include "components/task_card.html" with
urgency="urgent"
title="Review TLC feedback — ENG4021 module spec"
meta="Dept TLC returned with comments · CR-2025-041"
due_label="2 days"
due_urgency="red" %}
```

Left border: 3px solid, colour by urgency:

- `urgent` → `#E24B4A`
- `warn` → `#EF9F27`
- `info` → `#378ADD`
- `none` → `--border`

Icon square: 22×22px, border-radius: 4px, semantic background/colour matching urgency.
Title: 12px, 500, `--t1`, single line with overflow ellipsis.
Meta: 11px, `--t2`.
Due badge: 10px, 600, border-radius: 10px — red/amber/gray variants.

### 5.5 Standard card

```html
{% include "components/card.html" with title="My modules" link_label="View all" link_url="#" %}
{# card body content as block #}
{% endinclude %}
```

Header: title (12px, 600, `--t1`) + optional right-aligned link (11px, `--blue`).
Background: `--bg`. Border: `--border`. Border-radius: `--r`. Shadow: standard elevation.
Header padding: `10px 14px 0`. Body padding: `10px 14px 12px`.

### 5.6 Workflow pipeline (full)

Used on dashboard workflow detail and committee milestone views.

Six stages rendered as dot + label pairs connected by lines:

- Done: `--greenL` background, `--greenM` border, checkmark
- Current: `--blueL` background, `--blue` border, edit icon; "Current" badge above dot
- Pending: `--bg2` background, `--border` border, middle dot

Connecting lines: 2px height, flex: 1. Done lines: `--greenM`. Pending lines: `--border`.
Stage labels: 9px, `--t2`, centred, line-height: 1.3. Current label: `--blue`, 500.

```html
{% include "components/pipeline.html" with stages=workflow.stages %}
```

`stages` is a list of dicts: `[{label, state}]` where state is `done`, `current`, or `pending`.

### 5.7 Mini pipeline (inline)

Used in spec card workflow footers and sidebar context panels.

Horizontal dot-and-line strip. Dots: 10px circles. Lines: 1px × 16px. Current stage shows label inline. Labels for
pending stages shown only for the immediately next stage.

```html
{% include "components/pipeline_mini.html" with stages=workflow.stages show_current_label=True %}
```

### 5.8 Segmented control (view switcher)

Used in topbar only when a view has two distinct personas or major modes.

```css
.view-tabs {
    display: flex;
    gap: 2px;
    background: var(--bg2);
    border: 1px solid var(--border);
    border-radius: var(--r);
    padding: 2px;
}

.vt {
    font-size: 11px;
    font-weight: 500;
    padding: 4px 12px;
    border-radius: 6px;
}

.vt.on {
    background: var(--bg);
    color: var(--t1);
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.08);
}
```

### 5.9 AI screening flag

Used in the committee screening report only.

Three severity levels: `blocking`, `significant`, `advisory`.

| Severity      | Background | Border     | Badge bg               |
|---------------|------------|------------|------------------------|
| `blocking`    | `#FEF2F2`  | `#FCA5A5`  | `--red` (white text)   |
| `significant` | `--amberL` | `--amberM` | `--amber` (white text) |
| `advisory`    | `--blueL`  | `--blueM`  | `--blue` (white text)  |

Each flag contains: severity badge (uppercase, 9px, 700) + description text + affected item codes (monospace tags) +
action buttons (Acknowledge / Mark for action).

AI disclaimer footer: 10px, `--t3`, `--bg2` background, border: `--borderL`. Always present. Always states: model name,
on-premises, advisory only, no external data transmission.

```html
{% include "components/ai_flag.html" with flag=flag %}
```

### 5.10 AI field indicator

Applied to form fields that have been pre-filled by the LLM.

The field wrapper adds a top-right badge reading "✦ AI DRAFT — please review" in purple. The input/textarea border is
`--purpleM`, background is `#FDFCFF`.

On user edit (via HTMX `hx-trigger="input"` or JavaScript fallback): border transitions to `--blue`, background returns
to `--bg`, badge opacity reduces to 0.3, and a small note "✎ You have edited this field" appears below the field.

Fields that are submitted unchanged are tagged with `llm_result_unmodified=True` on the form submission for additional
reviewer scrutiny.

```html
{% include "components/form_field_ai.html" with field=form.outcome_text llm_draft=llm_result.outcome_text %}
```

### 5.11 Progress bar

Used on programme cards in the coordinator dashboard.

```html
{% include "components/progress_bar.html" with label="Module spec currency" value=22 total=24 colour="green" %}
```

Track height: 6px. Border-radius: 3px. Label row: 11px, `--t2`, space-between flex.
Colours: `green` → `--greenM`, `amber` → `#BA7517`, `blue` → `--blue`.

### 5.12 Spec card

Used in the module specifications list view.

Three sections: head (grid: code col + name/meta + status pill + action buttons) / body (3-column field grid, `--bg2`
background) / optional workflow footer (white background, mini pipeline + due date + action link).

Stale variant: card border becomes `--amberM`, footer background `#FAEEDA33`.

```html
{% include "components/spec_card.html" with module=module document=document workflow=workflow %}
```

### 5.13 Assessment group table

Used in the assessments list view. Module group header + table.

Group header: flex row, `--bg2` background, module code (monospace) + module name (600) + status pill + metadata (
right-aligned).
Table: fixed layout, 1px `--borderL` row separators. Retired rows at 0.5 opacity.
MLO tags: 10px, monospace, `--tealL` background, `--teal` text, 3px border-radius.

```html
{% include "components/assessment_group.html" with module=module slots=slots %}
```

### 5.14 Timeline

Used in dashboard recent activity. Vertical list with connecting line.

Dot: 20×20px circle with 2–3 letter code (e.g. TLC, PUB, SUB). Line: 1px `--borderL`, absolute positioned left of items.
Each item: dot + title (12px, `--t1`) + meta (11px, `--t2`).

```html
{% include "components/timeline.html" with events=recent_events %}
```

### 5.15 Compliance flag

Used in coordinator dashboard and committee AI screening.

Three variants: `red`, `amber`, `blue`. Each is a flex row: icon (⚑ / ⚠ / ℹ) + title (12px, 600) + sub-text (11px).

```html
{% include "components/flag.html" with severity="red" title="..." detail="..." %}
```

---

## 6. Form conventions

### 6.1 Field structure

```html

<div class="field">
    <label class="field-label">Field name <span class="req">*</span></label>
    <span class="field-hint">Hint text if needed</span>
    <!-- input or AI wrapper -->
</div>
```

Field label: 11px, 600, `--t2`. Required marker: `--red`. Hint: 10px, `--t3`.

### 6.2 Read-only fields

Background: `--bg2`. Colour: `--t2`. `cursor: default`. Use for context fields derived from registry (module name, FHEQ
level, tenant).

### 6.3 Field rows

Two-column layout: `display: grid; grid-template-columns: 1fr 1fr; gap: 14px`. Use for related pairs (module + level,
submission day + time).

### 6.4 Form sections

Each logical group of fields is a `.form-section` card with a section header (`.fs-head`) in `--bg2` and a body (
`.fs-body`) in `--bg`.

Section header: title (12px, 600, `--t1`) + optional right-aligned note (11px, `--t2`). If the section contains
AI-pre-filled fields, add the AI pill to the header.

### 6.5 Form actions footer

Fixed to bottom of form area. Contains (left to right): Save draft (ghost button) / Preview (secondary button) /
Submit (primary button) / submission note (11px, `--t3`, right-aligned).

### 6.6 Buttons

| Variant       | Background  | Border      | Text     |
|---------------|-------------|-------------|----------|
| Primary       | `--blue`    | `--blue`    | White    |
| Secondary     | `--bg`      | `--border`  | `--t2`   |
| Primary light | `--blueL`   | `--blueM`   | `--blue` |
| Ghost         | transparent | transparent | `--t2`   |

Font: 12px, 500. Padding: 7px 16px. Border-radius: `--rsm`.

---

## 7. HTMX patterns

### 7.1 Filter tabs and pills

Filter pills and page tab strips are HTMX-driven. They target a named container and replace its inner HTML.

```html

<button hx-get="{% url 'modules:specs' %}?state=in_workflow"
        hx-target="#spec-list"
        hx-swap="innerHTML"
        class="filter-pill {% if state == 'in_workflow' %}on{% endif %}">
    In workflow
</button>
```

The active class `on` is set server-side on the initial render and preserved via HTMX response headers.

### 7.2 AI field pre-fill arrival

If the LLM result is not yet ready when the form is first opened, fields render blank with a "✦ Preparing AI draft…"
indicator. A polling mechanism checks for the result:

```html

<div id="ai-status"
     hx-get="{% url 'llm:result_status' task_id=task.id %}"
     hx-trigger="every 2s [!aiReady]"
     hx-swap="outerHTML">
    <!-- replaced when result arrives -->
</div>
```

### 7.3 Agenda table row expansion

Committee agenda rows expand inline to show document preview:

```html

<tr hx-get="{% url 'committee:item_detail' pk=item.pk %}"
    hx-target="#detail-{{ item.pk }}"
    hx-swap="innerHTML">
```

---

## 8. Semantic colour rules

These rules govern which colour is used for a given data state. They are not suggestions — Claude Code must apply them
exactly.

| Data state                        | Colour |
|-----------------------------------|--------|
| Document published / approved     | Green  |
| Document in any workflow stage    | Amber  |
| Document blocked / action overdue | Red    |
| Document never published          | Gray   |
| No in-flight workflow (neutral)   | Gray   |
| AI-generated content, any type    | Purple |
| Primary action / navigation       | Blue   |
| Programme coordinator role        | Teal   |
| Committee / TLC chair role        | Purple |

---

## 9. What Claude Code must not do

- Do not introduce new hex colour values. Use tokens only.
- Do not use Bootstrap utility classes directly in feature templates — use the component includes.
- Do not write inline `style=""` attributes except for dynamic values computed in Python (e.g. progress bar width
  percentage).
- Do not create new component patterns. Flag the need and wait for a spec update.
- Do not use `<table>` layout for anything other than tabular data (assessment slots, agenda items). Use CSS grid or
  flex for layout.
- Do not add JavaScript for anything HTMX can handle.
- Do not apply the purple AI colour to non-AI UI elements under any circumstances.
- Do not use `--tealD` / teal accents for any role other than programme coordinator.
- Do not omit the AI disclaimer footer from any view that surfaces LLM output.

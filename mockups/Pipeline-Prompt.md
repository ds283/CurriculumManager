Read DESIGN.md before proceeding.

Create a workflow pipeline renderer as a self-contained JavaScript module 
at `static/js/workflow_pipeline.js` and a corresponding Django template 
fragment at `templates/workflows/pipeline.html`.

The renderer visualises a workflow FSM as a horizontal stage pipeline, 
matching the Bootstrap design system used throughout the application.

## Data structure

The renderer accepts a JSON object of this shape, delivered by a Django 
view to the template context (or fetched via HTMX from 
`/api/workflow/<id>/pipeline/`):

{
  "title": "Module Specification Approval",
  "type": "module_specification",
  "current_stage": "faculty_review",
  "stages": [
    { "id": "draft",          "label": "Draft",          "sublabel": "Author editing",    "status": "completed" },
    { "id": "submitted",      "label": "Submitted",      "sublabel": "Awaiting review",   "status": "completed" },
    { "id": "faculty_review", "label": "Faculty Review", "sublabel": "Panel assessment",  "status": "current"   },
    { "id": "approved",       "label": "Approved",       "sublabel": "Version published", "status": "pending"   }
  ],
  "transitions": [
    { "from": "draft",          "to": "submitted",      "label": "Submit", "active": true  },
    { "from": "submitted",      "to": "faculty_review", "label": "Assign", "active": true  },
    {
      "from": "faculty_review",
      "branches": [
        { "to": "approved",  "label": "Approve",  "type": "approve", "active": false },
        { "to": "submitted", "label": "Revise",   "type": "reject",  "active": false }
      ]
    }
  ]
}

## Visual requirements

- Horizontal left-to-right pipeline of stage nodes
- Three visual states for nodes: completed (green), current (blue, 
  with "Current" indicator badge), pending (grey)
- Simple labelled arrow connectors between stages
- Branch forks render as two vertically stacked paths with 
  approve/reject pill labels. Maximum one level of branching.
- Progress bar above the pipeline showing completed/total stages
- Legend below the pipeline
- Colour and typography must use Bootstrap CSS variables 
  (--bs-primary, --bs-success, --bs-body-color, --bs-font-sans-serif, 
  --bs-border-color, etc.) so it inherits the application theme
- Horizontally scrollable on small viewports

## Integration requirements

- The Django template fragment should accept a `pipeline_data` context 
  variable containing the JSON structure above
- The template should serialise it into a data attribute: 
  `data-pipeline='{{ pipeline_data|json_script }}'`
- The JavaScript module reads from that data attribute and renders 
  into a container element
- Must reinitialise correctly after an HTMX swap — listen for 
  `htmx:afterSwap` on the container

## Django view

Add a view at `workflows/views/pipeline.py` that:
- Takes a `workflow_instance_id` parameter
- Fetches the WorkflowInstance and its associated FSM definition
  using `get_fsm(workflow_instance.workflow_type)`
- Constructs the pipeline JSON by walking the FSM stages and 
  comparing against `workflow_instance.state`
- Returns JsonResponse for HTMX fetch, or adds to template context 
  for full page render

Follow the patterns established in the existing codebase for views, 
URL registration, and FSM access. Write tests in 
`tests/workflows/test_pipeline_view.py` following existing test patterns.
---
name: rocketlane
description: "Manage Rocketlane projects and tasks via REST API. Create, update, list, and delete tasks with phase and visibility (public/private) control. Use this skill whenever the user mentions Rocketlane, delivery tasks, project tracking, customer project management, task boards, creating follow-up tickets, or managing delivery project tasks. Also use when the user asks to create tasks from a status report or meeting notes."
---

# Rocketlane Project & Task Manager

Manage customer delivery projects in Rocketlane directly from Claude Code via REST API.

## Overview

Rocketlane is a customer onboarding and project delivery platform. This skill lets you:
- List projects and their tasks
- Create tasks (public or private, assigned to phases)
- Update task status (To do, In progress, Completed, Blocked)
- Delete tasks
- Read task details

**Public tasks** are visible to both Mattermost staff and the customer.
**Private tasks** are only visible to Mattermost staff (internal follow-ups, lessons learned, etc.).

## Configuration

### API Key

The skill requires a Rocketlane API key. Check for it in this order:

1. **Environment variable**: `ROCKETLANE_API_KEY`
2. **Ask the user** if not set

To obtain an API key: log into Rocketlane, navigate to your profile or settings, and generate an API key. The key from the browser session (visible in the `api-key` request header in DevTools) also works.

### Base URL

The default base URL for Mattermost's Rocketlane instance is:

```
https://services.api.mattermost.com/api/v1
```

If the user is on a different Rocketlane instance, they can specify the base URL. The general pattern is `https://<instance>.api.<domain>/api/v1`.

### Default Project

If the user frequently works with one project, they can specify a default project ID to avoid repeating it. Check the conversation context or ask which project they're working on. Use the "List Projects" endpoint to help them find it.

## API Reference

All requests use:
- **Header**: `api-key: <ROCKETLANE_API_KEY>`
- **Content-Type**: `application/json`
- **Base URL**: `https://services.api.mattermost.com/api/v1`

### List Projects

```bash
curl -s -H "api-key: $ROCKETLANE_API_KEY" \
  -H "Content-Type: application/json" \
  -X POST "$BASE_URL/projects/lightV1?sort=project*createdAt.desc&offset=0&limit=100" \
  -d '{"filter":{"nativeFields":[],"customFields":[],"match":"all","nestedFilter":[]}}'
```

Response contains an array of projects with `projectId`, `projectName`, `account` (customer), and status.

### Get Project Details

```bash
curl -s -H "api-key: $ROCKETLANE_API_KEY" \
  "$BASE_URL/projects/{projectId}"
```

### List Tasks in a Project

```bash
curl -s -H "api-key: $ROCKETLANE_API_KEY" \
  "$BASE_URL/projects/{projectId}/tasks?excludeMetaDataInFields=true"
```

Response is an array of task objects. Key fields per task:
- `taskId` — unique identifier
- `taskName` — task title
- `taskDescription` — HTML description
- `private` — boolean (true = staff-only, false = customer-visible)
- `projectPhase` — object with `projectPhaseId` and `projectPhaseName`
- `fields` — array containing Status and other custom fields
- `assignee` — users and teams assigned

To extract useful info from the response:
```python
import json, sys
data = json.load(sys.stdin)
for t in data:
    status = ''
    for f in t.get('fields', []):
        if f.get('fieldName') == 'Status':
            status = f.get('metaFieldValue', {}).get('label', '')
    priv = 'PRIVATE' if t.get('private') else 'PUBLIC'
    phase = t.get('projectPhase', {}).get('projectPhaseName', '-')
    print(f'[{status:12s}] [{priv:7s}] [{phase:15s}] {t["taskId"]} {t["taskName"]}')
```

### Create a Task

```bash
curl -s -X POST \
  -H "api-key: $ROCKETLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$BASE_URL/projects/{projectId}/tasks" \
  -d '{
    "taskName": "Task title",
    "taskDescription": "<p>HTML description of the task.</p>",
    "type": "TASK",
    "private": false,
    "projectPhase": {
      "projectPhaseId": 12345
    }
  }'
```

**Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `taskName` | string | Yes | Task title |
| `taskDescription` | string | No | HTML-formatted description |
| `type` | string | Yes | `"TASK"` or `"MILESTONE"` |
| `private` | boolean | Yes | `true` = staff-only, `false` = customer-visible |
| `projectPhase` | object | No | `{"projectPhaseId": <id>}` — assigns to a phase/column |
| `startDate` | string | No | `"YYYY-MM-DD"` format |
| `dueDate` | string | No | `"YYYY-MM-DD"` format |

**Phase assignment is important** — tasks without a phase won't appear in the Project Plan (kanban) view. Always discover available phases first (see "Discovering Phases" below).

### Update a Task

```bash
curl -s -X PUT \
  -H "api-key: $ROCKETLANE_API_KEY" \
  -H "Content-Type: application/json" \
  "$BASE_URL/tasks/{taskId}" \
  -d '{
    "taskName": "Updated title",
    "fields": [
      {
        "fieldName": "Status",
        "fieldValue": 2
      }
    ]
  }'
```

**Status values:**

| Value | Label |
|-------|-------|
| 1 | To do |
| 2 | In progress |
| 3 | Completed |
| 4 | Blocked |

### Delete a Task

```bash
curl -s -X DELETE \
  -H "api-key: $ROCKETLANE_API_KEY" \
  "$BASE_URL/tasks/{taskId}"
```

Returns empty body on success.

## Discovering Phases

Before creating tasks, discover the available project phases (kanban columns). Phases are embedded in existing tasks:

```bash
curl -s -H "api-key: $ROCKETLANE_API_KEY" \
  "$BASE_URL/projects/{projectId}/tasks?excludeMetaDataInFields=true" | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
phases = {}
for t in data:
    pp = t.get('projectPhase', {})
    pid = pp.get('projectPhaseId')
    pname = pp.get('projectPhaseName', 'None')
    if pid:
        phases[pid] = pname
for pid, pname in sorted(phases.items()):
    print(f'  {pid}: {pname}')
"
```

Common phase names: Intake, Prerequisites, Delivery, Wrapup.

## Common Workflows

### Create follow-up tasks from a status report

When the user has a status report or meeting notes and wants to create Rocketlane tasks:

1. Parse the report for actionable items
2. Determine which are customer-facing (public) vs internal (private)
3. Discover available phases in the target project
4. Create each task with the appropriate phase and visibility
5. Report back the created task IDs

### Bulk status update

To mark multiple tasks as completed:

```bash
for TASK_ID in 123 456 789; do
  curl -s -X PUT \
    -H "api-key: $ROCKETLANE_API_KEY" \
    -H "Content-Type: application/json" \
    "$BASE_URL/tasks/$TASK_ID" \
    -d '{"fields":[{"fieldName":"Status","fieldValue":3}]}'
  echo "Updated $TASK_ID"
done
```

### List all tasks grouped by phase

```bash
curl -s -H "api-key: $ROCKETLANE_API_KEY" \
  "$BASE_URL/projects/{projectId}/tasks?excludeMetaDataInFields=true" | \
  python3 -c "
import json, sys
from collections import defaultdict
data = json.load(sys.stdin)
by_phase = defaultdict(list)
for t in data:
    phase = t.get('projectPhase', {}).get('projectPhaseName', 'Unassigned')
    status = ''
    for f in t.get('fields', []):
        if f.get('fieldName') == 'Status':
            status = f.get('metaFieldValue', {}).get('label', '')
    priv = 'PRIV' if t.get('private') else 'PUB '
    by_phase[phase].append((status, priv, t['taskName'], t['taskId']))
for phase in ['Intake', 'Prerequisites', 'Delivery', 'Wrapup', 'Unassigned']:
    if phase in by_phase:
        print(f'\n=== {phase} ===')
        for status, priv, name, tid in by_phase[phase]:
            print(f'  [{status:12s}] [{priv}] {tid} {name}')
"
```

## Tips

- **Always use `python3 -c` for parsing JSON responses** — the responses are large and contain nested objects. Pipe curl output into python for clean formatting.
- **HTML in descriptions** — `taskDescription` accepts HTML. Use `<p>`, `<ul>`, `<li>`, `<code>` for formatting.
- **Shell quoting** — When task descriptions contain `$` or special characters, use single-quoted heredocs or escape carefully. Prefer single quotes around the JSON body.
- **Rate limiting** — No specific rate limits observed, but avoid tight loops with hundreds of requests.
- **Task creation returns the full task object** — You can extract the `taskId` from the response for confirmation.

# Project Brain MCP Tools — Complete Reference

All tools are called as MCP tool invocations. The default `response_mode` is `human` (readable text); pass `response_mode=json` when the output will be parsed programmatically.

---

## mcp__project-brain__context

Project context and discovery operations.

| Action | Description |
|---|---|
| `session` | Session start snapshot: in-progress tasks, todo backlog, recent decisions, team members |
| `summary` | High-level stats: task counts by status, milestone health |
| `search` | Full-text search across tasks, facts, decisions, skills |
| `changes` | Audit log of recent changes (use `since` to bound) |
| `shortlist` | Ranked subset of available tools for the current context |

### Parameters

| Param | Type | Notes |
|---|---|---|
| `action` | string | One of the above; defaults to `session` |
| `project_id` | UUID | Required for session/summary/changes/search |
| `q` | string | Search query (search action) |
| `since` | ISO-8601 | Lower bound timestamp (changes action) |
| `limit` | int | Max results; default 5 |
| `full_tool_mode` | bool | shortlist: return full catalog (default false) |

### Usage notes
- Call `session` at the start of every new conversation to orient before acting.
- Call `search` before implementing anything to avoid duplicating existing knowledge or contradicting past decisions.
- Call `changes` to catch up after time away from a project.

---

## mcp__project-brain__knowledge

Facts, decisions, and skills CRUD.

| Action | Description |
|---|---|
| `create` | Create a new knowledge entry |
| `list` | List entries with optional filters |
| `get` | Retrieve a single entry by ID |
| `update` | Update title, body, rationale, category |
| `delete` | Delete an entry |

### Parameters

| Param | Type | Used by | Notes |
|---|---|---|---|
| `entity` | string | all | Required: `fact`, `decision`, or `skill` |
| `action` | string | all | Required |
| `project_id` | UUID | create, list | Scopes entry to a project |
| `item_id` | UUID | get, update, delete | Entry UUID |
| `title` | string | create, update | Short, specific title |
| `body` | string | create, update | Main content (fact/skill) |
| `rationale` | string | create, update | Decision rationale (decision only) |
| `task_id` | UUID | create, update | Link entry to a specific task |
| `category` | string | create, update, list | Free-form label |
| `tags` | string[] | create, update, list | Tag array (skill only) |
| `q` | string | list | Text search |
| `cursor` / `limit` | string/int | list | Pagination |

### Entity type guidance

**fact** — Use `body` for the content. `title` should be a complete declarative statement.
```
title: "Alembic stores revision IDs in VARCHAR(32)"
body: "The alembic_version table uses VARCHAR(32) for the version_num column. Revision IDs longer than 32 characters cause StringDataRightTruncationError at migration time. Keep revision = '...' values under 32 chars."
```

**decision** — Use `rationale` for the explanation. `title` should name the choice made.
```
title: "Use cursor-based pagination over offset"
rationale: "Offset pagination becomes inconsistent under concurrent writes. Cursor-based is stable and consistent with our existing endpoints. Rejected: offset (unreliable), keyset (overkill for current scale)."
```

**skill** — Use `body` for numbered steps or structured procedure. `title` should be an action phrase.
```
title: "Run Alembic migrations safely in prod"
body: "1. pg_dump backup\n2. alembic upgrade head --sql (dry-run)\n3. Review SQL\n4. alembic upgrade head\n5. Verify alembic_version"
```

---

## mcp__project-brain__tasks

Task and milestone operations.

### Task Actions

| Action | Description |
|---|---|
| `list` | List tasks with filters |
| `create` | Create a single task |
| `update` | Update a task |
| `delete` | Delete a task |
| `context` | Get full context for a task (decisions, comments, dependencies) |
| `batch_create` | Create multiple tasks at once |
| `batch_update` | Update multiple tasks at once |
| `add_dependency` | Block task on another |
| `remove_dependency` | Remove a dependency |
| `add_comment` | Add a comment to a task |
| `list_comments` | List comments on a task |

### Milestone Actions

| Action | Description |
|---|---|
| `list_milestones` | List milestones for a project |
| `create_milestone` | Create a milestone |
| `update_milestone` | Update a milestone |
| `delete_milestone` | Delete a milestone |
| `reorder_milestones` | Reorder milestones |

### Key Task Parameters

| Param | Notes |
|---|---|
| `project_id` | Required for create and list |
| `task_id` | Required for update/delete/context/comments/dependencies |
| `title` | Task title |
| `description` | Markdown task description |
| `status` | Workflow status string (e.g. `todo`, `in_progress`, `done`) |
| `priority` | `low`, `medium`, `high`, `urgent` |
| `assignee_id` | User/agent UUID |
| `milestone_id` | Milestone UUID |
| `estimate` | Integer (story points or hours) |
| `sort_order` | Integer for manual ordering |

### Status values

Status values are **project-specific** — do not guess. Always call `projects action=get_workflow project_id=<id>` first to get the exact stage names. System defaults are `todo`, `in_progress`, `blocked`, `done`, but projects can customise these. Sending an invalid status causes the API to reject the request.

### batch_create payload

```json
items=[
  {"title": "Design schema", "status": "todo", "priority": "high"},
  {"title": "Write migration", "status": "todo", "priority": "medium"}
]
```

### batch_update payload

```json
updates=[
  {"id": "uuid-1", "status": "done"},
  {"id": "uuid-2", "status": "in_progress", "assignee_id": "agent-uuid"}
]
```

---

## mcp__project-brain__projects

Project and workflow management.

### Actions

| Action | Description |
|---|---|
| `list` | List all projects for the team |
| `get` | Get a single project |
| `create` | Create a project |
| `update` | Update name/description/settings |
| `get_workflow` | Get workflow stages and statuses |
| `add_workflow_stage` | Add a stage to the workflow |
| `update_workflow_stage` | Rename or update a stage |
| `delete_workflow_stage` | Remove a stage |
| `reorder_workflow_stages` | Reorder stages |

### Key Parameters

| Param | Notes |
|---|---|
| `project_id` | Required for get/update/workflow actions |
| `name` | Project name |
| `description` | Project description |
| `stage_id` | Workflow stage UUID |
| `stage_name` | Display name for stage |
| `stage_ids` | Ordered array for reorder action |
| `migrate_to_stage_id` | Where to move tasks when deleting a stage |

---

## mcp__project-brain__collaboration

Team, messaging, and identity operations.

### Actions

| Action | Description |
|---|---|
| `list_team_members` | List all humans and agents on the team |
| `discover_agents` | List AI agents specifically |
| `get_agent_activity` | Recent activity for a specific agent |
| `send_message` | Send async message to team member or broadcast |
| `get_messages` | Read inbox |
| `update_my_card` | Update your own agent profile card |
| `join_team` | Join a team using an invite code |

### Key Parameters

| Param | Notes |
|---|---|
| `recipient_id` | Target user/agent UUID (omit for broadcast) |
| `subject` | Message subject line |
| `body` | Message body (markdown supported) |
| `message_type` | `info`, `request`, `update`, `alert` |
| `mark_as_read` | Auto-mark messages read on fetch |
| `include_read` | Include already-read messages |
| `agent_id` | Agent UUID for get_agent_activity (defaults to self) |
| `since` | ISO-8601 timestamp to filter activity |
| `description` | Profile description for update_my_card |

### Message type guidance

- `info` — Status update, FYI, no response needed
- `request` — Asking another agent/human to take action
- `update` — Providing an update on something previously requested
- `alert` — Something requires immediate attention

---

## mcp__project-brain__files

Versioned file operations.

### Actions

| Action | Description |
|---|---|
| `list` | List files for a project |
| `get` | Get a file (latest version or specific version) |
| `create` | Create a new file |
| `add_version` | Add a new version to an existing file |
| `list_versions` | List all versions of a file |
| `delete` | Delete a file |

### Key Parameters

| Param | Notes |
|---|---|
| `project_id` | Required for list/create |
| `file_id` | Required for get/add_version/list_versions/delete |
| `title` | File title |
| `file_type` | `draft`, `spec`, `report`, `review`, `code` |
| `body` | File content (markdown) |
| `entity_type` | Linked entity type: `task`, `milestone`, etc. |
| `entity_id` | Linked entity UUID |
| `version` | Specific version number to retrieve |

### File type guidance

| Type | Use for |
|---|---|
| `spec` | Technical specifications, API designs, schema definitions |
| `draft` | Work-in-progress documents, proposals |
| `report` | Analysis results, status reports, post-mortems |
| `review` | Code review notes, design review feedback |
| `code` | Generated code snippets, scripts, configuration files |

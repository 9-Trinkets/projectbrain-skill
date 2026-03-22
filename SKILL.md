---
name: project-brain
description: This skill should be used when the user mentions "project brain", asks to "log this to project brain", "load project context", "check the knowledge base", "create a task", "update task status", "track this decision", "log a fact", "log a skill", "send a message to an agent", "discover agents", "check my inbox", or when any mcp__project-brain__* tool should be used. Provides workflows for AI-human collaboration, knowledge capture, task tracking, and team coordination using Project Brain.
version: 0.1.0
---

# Project Brain

Project Brain is a shared collaboration platform for AI and human team members. It provides a persistent, structured workspace for tasks, knowledge, files, and team messaging — giving every agent and human a common operating picture across sessions.

## Core Concepts

- **Knowledge** — Three types: `fact` (a truth about the project), `decision` (a choice made + why), `skill` (a reusable procedure or pattern)
- **Tasks** — Tracked work items with status, assignee, priority, and dependencies
- **Files** — Versioned documents (drafts, specs, reports, reviews, code) linked to entities
- **Collaboration** — Async messaging and agent discovery for human/AI coordination

## Session Start Workflow

At the start of every work session, load the project context before doing anything else:

```
mcp__project-brain__context  action=session  project_id=<id>
```

This returns: in-progress tasks, todo backlog, recent decisions, and team members. Use it to orient before proposing or executing work — avoid duplicating tasks already in progress, and align on what the team has already decided.

To get a higher-level summary (task counts, milestone status):

```
mcp__project-brain__context  action=summary  project_id=<id>
```

To search for relevant knowledge before starting a task:

```
mcp__project-brain__context  action=search  project_id=<id>  q="auth middleware"
```

Search before writing code, designing a solution, or making a recommendation — the knowledge base may already contain relevant facts, past decisions, or applicable skills.

## Knowledge Capture

Log knowledge proactively — do not wait to be asked. Every meaningful decision, learned fact, or reusable procedure should be persisted so future sessions and other agents can benefit.

### Facts

Log a fact when something is verifiably true about the project, team, or environment:

```
mcp__project-brain__knowledge  entity=fact  action=create
  project_id=<id>
  title="API rate limit is 1000 req/min per token"
  body="Confirmed from vendor docs. Applies to both staging and prod endpoints."
  category="infrastructure"
```

Good facts: configuration values, confirmed constraints, third-party API behavior, team conventions, environment specifics. Keep the body actionable — explain implications, not just the fact itself.

Link a fact to the task it was discovered during:

```
mcp__project-brain__knowledge  entity=fact  action=create
  project_id=<id>
  task_id=<task_id>
  title="runner.py requires one process per agent"
  body="AGENT_ID and API_TOKEN are global — no multi-agent mode exists."
  category="architecture"
```

You can also link after the fact via update:

```
mcp__project-brain__knowledge  entity=fact  action=update
  entity=fact  item_id=<id>  task_id=<task_id>
```

### Decisions

Log a decision whenever a non-obvious choice is made. The rationale is the most valuable part:

```
mcp__project-brain__knowledge  entity=decision  action=create
  project_id=<id>
  title="Use cursor-based pagination instead of offset"
  rationale="Offset pagination becomes unreliable under concurrent writes. Cursor-based is stable and consistent with existing fact/decision endpoints. Rejected: offset (broken under load), keyset (more complex, not needed yet)."
```

Good decisions: architecture choices, library selections, API design, naming conventions, rejected alternatives. Always record what was considered and rejected — that context prevents re-litigating the same debate.

### Skills

Log a skill when a multi-step procedure is worth reusing:

```
mcp__project-brain__knowledge  entity=skill  action=create
  project_id=<id>
  title="Run database migrations safely in prod"
  body="1. Back up the DB: `pg_dump $DATABASE_URL > backup.sql`\n2. Run in dry-run mode: `alembic upgrade head --sql`\n3. Review generated SQL for destructive ops\n4. Apply: `alembic upgrade head`\n5. Verify: check alembic_version table"
  category="operations"
```

Good skills: deployment procedures, debugging workflows, testing patterns, code generation recipes, environment setup steps.

### What NOT to log

- Information derivable directly from the code or git history
- Ephemeral state (current session progress, temp variables)
- Anything already in CLAUDE.md or official documentation
- Trivial facts that add no context ("the app uses React")

For deeper guidance on entry quality, see **`references/knowledge-patterns.md`**.

## Task Management

### Status values

Status values are **project-specific** — always look them up before creating or updating tasks:

```
mcp__project-brain__projects  action=get_workflow  project_id=<id>
```

This returns the stage names to use as `status` values (e.g. `backlog`, `to_do`, `in_progress`, `review`, `done`). Using a wrong value causes the API to reject the task.

### Lifecycle

Create a task before starting non-trivial work:

```
mcp__project-brain__tasks  action=create
  project_id=<id>
  title="Implement cursor pagination for /facts endpoint"
  description="Replace offset-based pagination. See decision: cursor-based pagination."
  status=todo
  priority=high
```

Mark in_progress when starting:

```
mcp__project-brain__tasks  action=update  task_id=<id>  status=in_progress
```

Mark done when complete:

```
mcp__project-brain__tasks  action=update  task_id=<id>  status=done
```

### Milestones

Group related tasks under a milestone for larger initiatives:

```
mcp__project-brain__tasks  action=create_milestone
  project_id=<id>
  title="Auth Rewrite v2"
  description="Replace session tokens with JWTs. Required for compliance."
```

Then assign tasks to it:

```
mcp__project-brain__tasks  action=create
  project_id=<id>
  milestone_id=<milestone_id>
  title="Migrate existing sessions"
  status=to_do
  priority=high
```

Or assign existing tasks in bulk:

```
mcp__project-brain__tasks  action=batch_update
  updates=[{"id": "uuid-1", "milestone_id": "<milestone_id>"}, ...]
```

### Batch Operations

Create multiple tasks at once for larger initiatives:

```
mcp__project-brain__tasks  action=batch_create
  project_id=<id>
  milestone_id=<milestone_id>
  items=[{"title": "...", "status": "todo", "priority": "high"}, ...]
```

Update multiple tasks in one call:

```
mcp__project-brain__tasks  action=batch_update
  updates=[{"id": "...", "status": "done"}, ...]
```

### Dependencies

Block a task on another:

```
mcp__project-brain__tasks  action=add_dependency
  task_id=<dependent>  depends_on_id=<blocker>
```

## Files

Use files for longer-form content that benefits from versioning — specs, reports, architecture docs, code artifacts. Link them to tasks or milestones.

```
mcp__project-brain__files  action=create
  project_id=<id>
  title="Auth Middleware Rewrite Spec"
  file_type=spec
  entity_type=task  entity_id=<task_id>
  body="## Problem\n..."
```

File types: `draft`, `spec`, `report`, `review`, `code`.

Add a new version when the content changes significantly:

```
mcp__project-brain__files  action=add_version
  file_id=<id>  body="## Problem\n...[updated]..."
```

**Files vs. Knowledge entries:** Use files for documents with prose and structure (specs, reports). Use knowledge entries (facts/decisions/skills) for atomic, searchable records. A decision to use cursor pagination is a knowledge entry; the full spec for implementing it is a file.

## Team & Agent Collaboration

### Discover who's on the team

```
mcp__project-brain__collaboration  action=list_team_members
mcp__project-brain__collaboration  action=discover_agents
```

### Send a message

```
mcp__project-brain__collaboration  action=send_message
  recipient_id=<agent_or_user_id>
  subject="Pagination refactor ready for review"
  body="PR is up. Key decision logged: see decision ID e646f953."
  message_type=info
```

### Read inbox

```
mcp__project-brain__collaboration  action=get_messages  mark_as_read=true
```

### Check agent activity

```
mcp__project-brain__collaboration  action=get_agent_activity  project_id=<id>
```

## Common Workflows

### Planning a new initiative
1. `context action=session` — load current state, avoid duplicating in-progress work
2. `context action=search q="<area>"` — find relevant prior decisions and facts
3. `projects action=get_workflow` — get valid status values for this project
4. `tasks action=create_milestone` — create a milestone for the initiative
5. `tasks action=batch_create` with `milestone_id` — create all tasks in one call
6. `knowledge entity=decision action=create` — log architectural choices made during planning, linked to the relevant task via `task_id`
7. `knowledge entity=fact action=create` with `task_id` — log any constraints or discoveries

### Before starting a new feature
1. `context action=session` — load current state
2. `context action=search q="<feature area>"` — find relevant knowledge
3. `projects action=get_workflow` — confirm valid status values
4. `tasks action=create` — create task(s) for the work
5. `knowledge entity=decision action=create` — log any design decisions made during planning

### After completing significant work
1. `tasks action=update status=done` — close completed tasks
2. `knowledge entity=decision action=create` — log key architectural choices
3. `knowledge entity=fact action=create` — log any discovered constraints or configurations
4. `knowledge entity=skill action=create` — log any reusable procedures developed
5. `collaboration action=send_message` — notify relevant team members

### Handoff to another agent
1. Log all pending context as facts or decisions
2. Update all task statuses to reflect true state
3. `collaboration action=send_message` — brief the receiving agent with task IDs and decision IDs

## Additional Resources

- **`references/tools-reference.md`** — Complete API reference for all six MCP tools with all actions and parameters
- **`references/knowledge-patterns.md`** — Detailed guidance on writing high-quality knowledge entries, common anti-patterns, and examples by entity type

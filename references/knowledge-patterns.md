# Knowledge Entry Patterns

Guidance for writing high-quality facts, decisions, and skills in Project Brain.

---

## The Three Entity Types

### When to use `fact`

A fact is a verifiable, stable truth about the project, system, or environment that future agents need to know but can't derive from the codebase alone.

**Log a fact when:**
- A configuration value, limit, or constraint is confirmed
- A third-party system behaves in a non-obvious way
- A team convention is established
- An environment detail is discovered through trial or documentation
- A "gotcha" is encountered and resolved

**Good fact examples:**
```
title: "Alembic revision IDs must be <= 32 characters"
body: "The alembic_version table stores revision IDs in VARCHAR(32). IDs longer than 32 chars raise StringDataRightTruncationError at migration time. Pattern: use snake_case short names like '046_skill_quality_metadata' (26 chars)."
category: "database"

title: "Docker postgres must be started before running migrations"
body: "Use `docker compose up -d postgres` (not brew services). The API container depends on this being healthy."
category: "infrastructure"

title: "risk_score on CurationRecommendation means confidence, not danger"
body: "Field naming is misleading. risk_score = 0.9 means the recommendation is 90% confident, not 90% risky. High risk_score = strong recommendation. UI labels this as 'confidence'."
category: "domain-model"
```

**Bad fact examples:**
- "The app uses React" — derivable from package.json, adds no context
- "We're building a project management tool" — too obvious/vague
- "Fixed a bug today" — ephemeral, use a task + commit instead

---

### When to use `decision`

A decision is a non-obvious architectural, design, or process choice made by the team. The rationale is what makes it valuable — not just what was chosen, but why, and what alternatives were rejected.

**Log a decision when:**
- A technology, library, or approach is selected over alternatives
- An API contract or data model design is finalized
- A naming convention or code pattern is adopted
- A previous decision is reversed or updated
- A trade-off is consciously accepted

**Anatomy of a good decision:**
1. **Title** — Name the choice, not the problem. "Use X" not "Should we use X?"
2. **What was decided** — Brief statement if not obvious from title
3. **Why** — Primary motivation (performance, correctness, DX, constraints)
4. **What was rejected** — Alternatives considered and why they lost
5. **Caveats** — Conditions under which this should be revisited

**Good decision examples:**
```
title: "Use cursor-based pagination over offset for all list endpoints"
rationale: "Offset pagination becomes inconsistent under concurrent writes — records shift, causing duplicates or gaps in pages. Cursor-based pagination is stable regardless of inserts. Rejected: offset (broken under load); keyset pagination (more complex, not needed at current scale). Revisit if we need bi-directional navigation."

title: "FLAG recommendations show Delete / Edit / Keep instead of Accept"
rationale: "Accepting a FLAG had no defined side effect and confused users. Most flagged items need deletion or rewriting, not acceptance. Delete calls entity delete API + resolves recommendation. Edit opens inline textarea. Keep dismisses without touching entity. Non-flag recs keep Accept/Snooze/Dismiss."

title: "Rename groomer → curator throughout codebase"
rationale: "Groomer was ambiguous and technical. Curator is semantically accurate — curators review, organize, and maintain knowledge collections. The LLM system prompt already called it 'a knowledge base curator'. Rejected: consolidate (IT connotation), reconcile (accounting), prune (good but less precise)."
```

**Bad decision examples:**
- "We chose Postgres" — no rationale, no alternatives
- "Fixed the pagination bug" — that's a task completion note, not a decision
- "Follow best practices" — too vague to be actionable

---

### When to use `skill`

A skill is a reusable procedure that an agent or human would otherwise re-derive from scratch each time. If you've figured out a multi-step process that isn't documented elsewhere, log it.

**Log a skill when:**
- A deployment or operational procedure is established
- A debugging workflow is developed
- A code generation pattern is discovered
- A testing approach is formalized
- A setup sequence is figured out

**Structure of a good skill:**
- Title as an action phrase: "Deploy to production", "Debug flaky tests"
- Numbered steps in the body
- Include commands verbatim where possible
- Note prerequisites and common failure points

**Good skill examples:**
```
title: "Apply Alembic migrations safely in production"
body: |
  Prerequisites: DATABASE_URL env var set, postgres running.

  1. Backup: `pg_dump $DATABASE_URL > backup_$(date +%Y%m%d).sql`
  2. Dry-run: `alembic upgrade head --sql | less` (review for destructive ops)
  3. Apply: `alembic upgrade head`
  4. Verify: `psql $DATABASE_URL -c "SELECT version_num FROM alembic_version;"`

  Common failure: StringDataRightTruncationError — revision ID > 32 chars. Fix: shorten the revision ID string.
category: "operations"

title: "Fix SQLAlchemy autoflush FK violation on accept"
body: |
  Symptom: ForeignKeyViolationError when setting entity.superseded_by_id then querying.
  Cause: SQLAlchemy autoflush fires before FK target is fetched, violating the constraint.
  Fix: always fetch the related (canonical) entity FIRST, then set the FK field.

  Pattern:
  canonical = await db.execute(select(Model).where(Model.id == related_id))
  canonical = canonical.scalar_one_or_none()
  if canonical:
      entity.superseded_by_id = related_id  # safe — FK target confirmed
category: "backend"
```

---

## Writing Quality Checklist

### Titles
- [ ] States the fact/decision/skill clearly — readable standalone
- [ ] Specific enough to be findable by search
- [ ] Fact: declarative statement ("X is Y")
- [ ] Decision: action phrase ("Use X", "Rename X to Y")
- [ ] Skill: verb phrase ("Run X", "Debug X", "Fix X")

### Bodies / Rationale
- [ ] Answers "why" not just "what"
- [ ] Includes context that isn't derivable from code
- [ ] For decisions: records rejected alternatives
- [ ] For skills: includes exact commands/steps
- [ ] For facts: explains implications, not just the fact
- [ ] Under ~500 words (long enough to be useful, short enough to be read)

### Category
- [ ] One of a consistent set: `infrastructure`, `database`, `backend`, `frontend`, `api`, `domain-model`, `operations`, `process`, `security`, `testing`
- [ ] Omit if genuinely cross-cutting

---

## Anti-Patterns to Avoid

### Duplicate entries
Always search before creating: `context action=search q="<topic>"`. Update an existing entry rather than creating a redundant one.

### Ephemeral state as facts
"Currently working on the auth rewrite" belongs in a task status, not a fact. Facts should remain true indefinitely or until explicitly superseded.

### Vague titles
"Important database thing" is unsearchable. "PostgreSQL connection pool limit is 20 per dyno on Render" is findable.

### Missing rationale on decisions
A decision without rationale is just a rule. The rationale is what allows future agents to judge whether the rule still applies in a new situation.

### Logging what's in the code
Don't log "the Fact model has a superseded_by_id field" — that's in the code. Log "superseded_by_id must be fetched before setting to avoid SQLAlchemy autoflush" — that's non-obvious operational knowledge.

---

## When to Update vs. Create New

**Update** an existing entry when:
- A fact changes (new value, corrected information)
- A decision is refined or reversed
- A skill's procedure is improved

**Create new** when:
- The topic is genuinely different
- A new decision supersedes an old one (log both — the old one explains the history)

**Delete** when:
- An entry is factually wrong and has no historical value
- A skill no longer applies (the system it describes no longer exists)

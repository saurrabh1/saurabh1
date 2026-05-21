# Jarvis — Neural Dot Memory System
## Database Schema Design (PostgreSQL + pgvector)

---

## Design Philosophy: Strict Bucket Isolation

The "Neural Dot" memory system enforces isolation at the **database schema level**, not application logic. Each bucket is a **separate table** — not a single table with a `bucket_type` enum column. This means:

- A query against `bucket_knowledge` can never accidentally return a `bucket_personal` row, even if application code has a bug.
- Separate indexes per bucket — no cross-contamination in vector similarity searches.
- Row-Level Security (RLS) policies applied independently per table.
- Future scaling: each bucket can be moved to its own partition or even a separate database without schema changes.

---

## Schema

### Enable Extensions

```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";    -- gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS "vector";       -- pgvector for embeddings
```

---

### Core: Users

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) UNIQUE NOT NULL,
    display_name    VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_active_at  TIMESTAMPTZ
);

CREATE INDEX idx_users_email ON users(email);
```

---

### Core: Conversation Sessions (Short-Term Context)

Tracks each voice interaction session. The Agent Manager uses this for multi-turn context within a single conversation, stored in Redis with TTL — this table is the persistent log.

```sql
CREATE TABLE conversation_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ended_at        TIMESTAMPTZ,
    turn_count      INTEGER DEFAULT 0,
    raw_context     JSONB DEFAULT '[]'   -- array of {role, content} turns
);

CREATE INDEX idx_sessions_user ON conversation_sessions(user_id, started_at DESC);
```

---

### BUCKET 1 — Personal Stuff

Personal logs, thoughts, diary-style entries, lifestyle observations.

```sql
CREATE TABLE bucket_personal (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    session_id      UUID REFERENCES conversation_sessions(id) ON DELETE SET NULL,

    -- Content
    raw_transcript  TEXT NOT NULL,          -- exact STT output
    summary         TEXT,                   -- Claude-generated summary
    tags            TEXT[] DEFAULT '{}',    -- e.g. ["mood", "health", "family"]

    -- Vector search
    embedding       vector(1536),           -- text-embedding-3-small

    -- Timestamps
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Metadata
    mood_signal     VARCHAR(20),            -- optional: "positive","negative","neutral"
    metadata        JSONB DEFAULT '{}'
);

CREATE INDEX idx_personal_user_time  ON bucket_personal(user_id, recorded_at DESC);
CREATE INDEX idx_personal_tags       ON bucket_personal USING GIN(tags);
CREATE INDEX idx_personal_embedding  ON bucket_personal USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

---

### BUCKET 2 — Reminders & Tasks

To-do items, follow-ups, time-based tasks. This bucket is tightly coupled to the Notification Daemon.

```sql
CREATE TABLE bucket_reminders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    session_id      UUID REFERENCES conversation_sessions(id) ON DELETE SET NULL,

    -- Content
    raw_transcript  TEXT NOT NULL,
    task_title      TEXT NOT NULL,          -- extracted task name
    task_detail     TEXT,                   -- fuller description

    -- Scheduling
    due_at          TIMESTAMPTZ,            -- nullable: no due date = "someday"
    is_completed    BOOLEAN NOT NULL DEFAULT FALSE,
    completed_at    TIMESTAMPTZ,
    priority        VARCHAR(10) NOT NULL DEFAULT 'medium'
                    CHECK (priority IN ('high', 'medium', 'low')),

    -- Vector search
    embedding       vector(1536),

    -- Timestamps
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    metadata        JSONB DEFAULT '{}'
);

CREATE INDEX idx_reminders_user_due      ON bucket_reminders(user_id, due_at ASC NULLS LAST);
CREATE INDEX idx_reminders_user_complete ON bucket_reminders(user_id, is_completed);
CREATE INDEX idx_reminders_embedding     ON bucket_reminders USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

---

### BUCKET 3 — Knowledge & Information

Researched facts, learned information, saved web results, reference material.

```sql
CREATE TABLE bucket_knowledge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    session_id      UUID REFERENCES conversation_sessions(id) ON DELETE SET NULL,

    -- Content
    raw_transcript  TEXT NOT NULL,
    summary         TEXT,
    source_url      TEXT,                   -- if sourced from web search
    domain          VARCHAR(100),           -- e.g. "AI", "Finance", "Health"
    tags            TEXT[] DEFAULT '{}',

    -- Vector search
    embedding       vector(1536),

    -- Timestamps
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    metadata        JSONB DEFAULT '{}'
);

CREATE INDEX idx_knowledge_user_time  ON bucket_knowledge(user_id, recorded_at DESC);
CREATE INDEX idx_knowledge_domain     ON bucket_knowledge(user_id, domain);
CREATE INDEX idx_knowledge_tags       ON bucket_knowledge USING GIN(tags);
CREATE INDEX idx_knowledge_embedding  ON bucket_knowledge USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

---

### BUCKET 4 — Projects & Ideas

Brainstorming, business ideas, coding architecture notes, product thoughts.

```sql
CREATE TABLE bucket_projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    session_id      UUID REFERENCES conversation_sessions(id) ON DELETE SET NULL,

    -- Content
    raw_transcript  TEXT NOT NULL,
    summary         TEXT,
    project_name    TEXT,                   -- explicit project label if spoken
    idea_type       VARCHAR(30),            -- "business", "code", "product", "creative"
    tags            TEXT[] DEFAULT '{}',

    -- Vector search
    embedding       vector(1536),

    -- Timestamps
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    metadata        JSONB DEFAULT '{}'
);

CREATE INDEX idx_projects_user_time    ON bucket_projects(user_id, recorded_at DESC);
CREATE INDEX idx_projects_name         ON bucket_projects(user_id, project_name);
CREATE INDEX idx_projects_tags         ON bucket_projects USING GIN(tags);
CREATE INDEX idx_projects_embedding    ON bucket_projects USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

---

### Notification Daemon Store

The scheduler table tracks push notifications independent of `bucket_reminders`. A reminder entry creates a notification entry — but they remain separate tables so a notification can be rescheduled or dismissed without altering the original reminder.

```sql
CREATE TABLE scheduled_notifications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- Content
    title               TEXT NOT NULL,
    tts_message         TEXT NOT NULL,      -- exact string sent to TTS engine

    -- Timing
    scheduled_for       TIMESTAMPTZ NOT NULL,
    notify_before_min   INTEGER NOT NULL DEFAULT 5,
    actual_fired_at     TIMESTAMPTZ,
    is_fired            BOOLEAN NOT NULL DEFAULT FALSE,
    is_dismissed        BOOLEAN NOT NULL DEFAULT FALSE,

    -- Source linkage (optional)
    source_reminder_id  UUID REFERENCES bucket_reminders(id) ON DELETE SET NULL,

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Daemon polls this index on every tick
CREATE INDEX idx_notifications_pending ON scheduled_notifications(
    user_id,
    scheduled_for ASC
) WHERE is_fired = FALSE AND is_dismissed = FALSE;
```

---

## Row-Level Security (RLS)

Supabase exposes tables over a REST API. RLS ensures a user can only ever read/write their own bucket — no cross-user data leak even if API keys are misused.

```sql
-- Enable RLS on every bucket table
ALTER TABLE bucket_personal     ENABLE ROW LEVEL SECURITY;
ALTER TABLE bucket_reminders    ENABLE ROW LEVEL SECURITY;
ALTER TABLE bucket_knowledge    ENABLE ROW LEVEL SECURITY;
ALTER TABLE bucket_projects     ENABLE ROW LEVEL SECURITY;
ALTER TABLE scheduled_notifications ENABLE ROW LEVEL SECURITY;

-- Policy template (repeat for each table)
CREATE POLICY "Users access own personal data"
ON bucket_personal
FOR ALL
USING (user_id = auth.uid());

CREATE POLICY "Users access own reminders"
ON bucket_reminders
FOR ALL
USING (user_id = auth.uid());

CREATE POLICY "Users access own knowledge"
ON bucket_knowledge
FOR ALL
USING (user_id = auth.uid());

CREATE POLICY "Users access own projects"
ON bucket_projects
FOR ALL
USING (user_id = auth.uid());

CREATE POLICY "Users access own notifications"
ON scheduled_notifications
FOR ALL
USING (user_id = auth.uid());
```

---

## Semantic Search Query Pattern

The Neural Dot retrieval for the dashboard UI (or agent memory lookup) uses vector cosine similarity:

```sql
-- "Find entries in my Projects bucket similar to this query"
SELECT
    id,
    summary,
    project_name,
    tags,
    recorded_at,
    1 - (embedding <=> $1::vector) AS similarity
FROM bucket_projects
WHERE user_id = $2
ORDER BY embedding <=> $1::vector   -- cosine distance ascending
LIMIT 10;
```

---

## Entity-Relationship Summary

```
users
  │
  ├── conversation_sessions (many)
  │
  ├── bucket_personal       (many) ◄─── ISOLATED
  ├── bucket_reminders      (many) ◄─── ISOLATED ──► scheduled_notifications
  ├── bucket_knowledge      (many) ◄─── ISOLATED
  └── bucket_projects       (many) ◄─── ISOLATED

No foreign keys cross bucket boundaries.
No shared indexes across bucket tables.
```

---

## Migration Order

```
1. users
2. conversation_sessions
3. bucket_personal
4. bucket_reminders
5. bucket_knowledge
6. bucket_projects
7. scheduled_notifications
8. (RLS policies last)
```

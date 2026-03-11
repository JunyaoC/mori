# Architecture

## The Hotel Model

mori uses a hotel metaphor for its architecture. Every component has a clear role and boundary.

### Roles

| Role | What it is | What it does |
|---|---|---|
| **Guest** | The user | Talks to any channel or GitHub directly |
| **Concierge** | Channel adapters (Discord, CLI, Slack) | Normalizes input, routes to manager or department |
| **Manager** | One persistent LLM agent | Coordinates, decides, dispatches. NOT a bottleneck — departments can act independently |
| **Office** | GitHub (Issues, Board, PRs, Labels) | Source of truth. Every mutation is reflected here. |
| **Scratchpad** | Database (SQLite + embeddings) | Working memory. Conversations, intermediate thinking, semantic search. |
| **Departments** | Spawned agents (claude-code, codex, research) | Execute tasks autonomously. Sync mutations back to office. |
| **Inbox** | #inbox / #attention channel | Surfaces things that need user action |

### The Two Layers

```
GITHUB (Office) = Mutations / Decisions / State Changes
  "anything that changes the world goes here"

  - new task → issue
  - status change → label/board move
  - code change → PR
  - decision → issue comment
  - knowledge update → PR to .agent/
  - research result → issue with findings

DATABASE (Scratchpad) = Conversation Continuity / Working Memory
  "the thinking, the back-and-forth, the scratch work"

  - full message history (every word, every channel)
  - embeddings for semantic search
  - intermediate thoughts and reasoning
  - department work-in-progress stream
  - session resume tokens
  - dispatch queue
```

### The Mutation Rule

The core invariant: **if it changes state, it goes to GitHub.**

```
MUTATION (→ GitHub):
  new task, status change, code change, decision made,
  knowledge learned, schedule created, assignment changed,
  research completed with findings

READ (→ DB only):
  "what's the status", "what's btc price", thinking out loud,
  intermediate reasoning, drafts, back-and-forth conversation
```

The bar for creating an issue: **did this produce something worth finding later?**

### Data Flow

#### Top-Down (User → Manager → Department)

```
User in #general: "change footer to XYZ"
    │
    DB: store message
    │
    Manager:
    ├── reads DB (conversation context, semantic search)
    ├── reads GitHub (board, active tasks)
    ├── DECIDES: this is a mutation
    │
    ├── GitHub: create Issue #72, label in-progress
    ├── DB: queue department dispatch
    │
    └── dispatches to coding department in #project-x
```

#### Bottom-Up (User → Department → Office)

Departments have autonomy. Users can talk to departments directly.

```
User in #project-x: "auth callback broken on safari"
    │
    DB: store message
    │
    Department (no manager needed):
    ├── reads DB (project-x conversation context)
    ├── reads GitHub (existing project-x issues)
    ├── DECIDES: this is a mutation
    │
    ├── GitHub: create Issue #73, label bug
    ├── DB: working scratchpad (investigating...)
    ├── GitHub: PR #74 when fix is ready
    ├── GitHub: Issue #73 label → needs-review
    │
    └── notifies #inbox: "safari auth fix ready"

    Manager (next read cycle):
    ├── sees Issue #73 in office
    ├── has full context
    └── can reference it from any channel
```

#### Lateral (Department → Department)

```
Coding department discovers: "this safari bug affects SSO too"
    │
    GitHub: link Issue #73 ↔ Issue #65
    GitHub: comment on #65: "related safari bug found"
    │
    Manager and all departments see the link
```

#### Read-Only (No mutation)

```
User: "what's btc at"
    │
    DB: store message + response
    No GitHub event.
    But searchable forever via DB embeddings.
```

### Department Autonomy

Departments don't need manager approval for everything:

```
DEPARTMENT CAN DO AUTONOMOUSLY:
├── Respond to user in its channel
├── Create/update issues in its domain
├── Open PRs
├── Stream working progress to DB
├── Link related issues
└── Notify #inbox when user attention needed

DEPARTMENT MUST ALWAYS DO:
├── Reflect every mutation to GitHub (office)
├── Log everything to DB (scratchpad)
└── Stay within its domain scope

DEPARTMENT NEEDS MANAGER FOR:
├── Cross-department coordination
├── Decisions affecting multiple domains
├── Promoting casual chat to tracked task
└── Priority conflicts
```

### State Model

| State | Where | Why there |
|---|---|---|
| Tasks | GitHub Issues | Durable, auditable, version-controlled |
| Task status | GitHub Labels + Board | Visual, queryable, native tooling |
| Code output | GitHub PRs | Native code review, CI/CD |
| Decisions | GitHub Issue comments | Permanent record with context |
| Large artifacts | GitHub Gists | API-uploadable, linkable |
| Cross-task links | GitHub Issue references | Native linking |
| Agent config | `.agent/` in repo | Version-controlled evolution |
| Tools | `.agent/tools/` | Agent-evolvable via PRs |
| Knowledge | `.agent/knowledge/` | Version-controlled, PR-evolvable |
| Scheduled work | GitHub Actions | Version-controlled cron |
| Conversation history | DB: messages table | Fast, every message, semantic searchable |
| Working context | DB: threads table | Current state of mind per thread |
| Session tokens | DB: sessions table | Agent resume capability |
| Dispatch queue | DB: queue table | Async department coordination |
| Embeddings | DB: vector index | Semantic search over all conversations |

### Context Loading

Every interaction, the agent (manager or department) loads from both layers:

```python
context = {
    # From DB (fast, conversational)
    "conversation": db.recent_messages(thread_id, limit=20),
    "semantic":     db.search(msg.content, limit=5),  # related past conversations
    "session":      db.get_active_session(thread_id),

    # From GitHub (authoritative)
    "active":       gh.issues(state="open", label="in-progress"),
    "board":        gh.get_project_board(),
    "related":      gh.search_issues(extract_keywords(msg)),
    "open_prs":     gh.pulls(state="open"),
}
```

### Continuity Model

**Conversational continuity**: DB stores every message with embeddings. "What did we discuss about X?" works via semantic search across all channels and time.

**Task continuity**: each issue has a living summary (first comment) capturing current state of mind. Any new session reads this and picks up immediately.

**Cross-task continuity**: the board gives a global view. Related issues are linked. Manager sees everything.

**Cross-channel continuity**: DB has all messages from all channels. GitHub has all mutations. Between the two, no context is ever lost.

### Multi-Model Support

The manager routes to different LLM providers via LiteLLM:

- Quick questions → fast/cheap model (haiku, gemini-flash)
- Coding tasks → spawn claude-code or codex as CLI subprocess
- Deep research → capable model (sonnet, o3)
- The manager itself → consistent capable model

Model preferences live in `.agent/prompt.md` and evolve over time.

### Artifact Pipeline

When departments produce output:

1. Stream progress to DB in real-time (every tool call, every intermediate result)
2. Sync summarized progress to GitHub Issue comments on milestones
3. Small artifacts (diffs, test results) → inline in issue comments
4. Large artifacts (full logs, screenshots) → GitHub Gists, linked from comments
5. Code changes → PRs linked to the issue
6. On completion → update living summary, move board card, notify #inbox

### Database Schema

```sql
CREATE TABLE messages (
    id TEXT PRIMARY KEY,
    thread_id TEXT,
    source TEXT,             -- "discord/#general", "cli", "github/issue/72"
    author TEXT,             -- "jun", "mori", "dept:coding"
    content TEXT,
    embedding VECTOR(1536),
    github_issue INTEGER,    -- linked issue, NULL for casual chat
    created_at TIMESTAMP
);

CREATE TABLE threads (
    id TEXT PRIMARY KEY,
    living_summary TEXT,
    github_issue INTEGER,    -- NULL until promoted to task
    department TEXT,
    status TEXT,             -- "active", "waiting", "done"
    last_active TIMESTAMP
);

CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    thread_id TEXT,
    agent_type TEXT,         -- "manager", "claude-code", "codex"
    session_token TEXT,
    status TEXT,
    created_at TIMESTAMP
);

CREATE TABLE queue (
    id TEXT PRIMARY KEY,
    thread_id TEXT,
    action TEXT,
    payload JSON,
    status TEXT,
    created_at TIMESTAMP
);
```

Start with SQLite + sqlite-vec for embeddings. Move to Postgres + pgvector if needed.

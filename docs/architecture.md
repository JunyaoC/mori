# Architecture

## The Hotel Model

mori uses a hotel metaphor. Every component has a clear role and boundary.

### Roles

| Role | What it is | What it does |
|---|---|---|
| **Guest** | The user | Talks to any channel or GitHub directly |
| **Concierge** | Channel adapters with channel awareness | Turns every interaction into a GitHub Issue, routes to the right place |
| **Manager** | One persistent LLM agent | Reads GitHub state, decides, acts, dispatches, self-evolves |
| **Office** | GitHub (Issues, Board, PRs, Labels, `.agent/`) | Source of truth. The bus. Every interaction recorded here. |
| **Departments** | Spawned agents (claude-code, codex, research) | Execute tasks. Report back to office. Can receive user input directly. |
| **DB** | SQLite + embeddings | Search index over GitHub content. Session tokens. Working scratch. Rebuildable from GitHub. |
| **Inbox** | #inbox channel | Surfaces things that need user action |

### Core Principle: GitHub Is the Bus

Every interaction — quick question, long task, casual chat — creates or continues a GitHub Issue. GitHub is the universal record and the single source of truth.

```
GITHUB = the bus
  every message in, every response out → recorded as issue comments
  every status change → labels, board
  every code output → PRs
  every self-evolution → PRs to .agent/

DB = search index + working cache
  embeddings over issue content (for semantic search)
  session tokens (for agent resume)
  thread-to-issue mappings (for channel routing)
  working scratch (intermediate agent state)

  KEY PROPERTY: rm -rf mori.db && rebuild_from_github() → zero data loss
```

### Concierge: Channel-Aware Router

The concierge knows the channel topology and turns every message into GitHub Issue activity. It is NOT an LLM — it's deterministic routing + one cheap classify call when needed.

```python
async def concierge(msg: NormalizedMessage):
    # 1. Channel awareness: what kind of channel is this?
    channel_config = channels.get(msg.source)

    # 2. Find existing issue for this conversation
    issue = (
        db.get_issue_for_thread(msg.source, msg.thread_id)  # known mapping
        or gh.find_related_open_issue(msg, channel_config)    # search
    )

    # 3. If no existing issue, create one
    if not issue:
        issue = gh.create_issue(
            title=summarize(msg.content),
            labels=classify_labels(msg, channel_config),
        )
        db.map_thread(msg.source, msg.thread_id, issue.number)

    # 4. Record the message on the issue
    gh.add_comment(issue.number,
        f"**[{msg.source}] @{msg.author}**: {msg.content}")

    # 5. Route to handler based on channel type
    if channel_config.type == "department":
        # Bottom-up: department handles directly
        await department_handler(issue, msg, channel_config)
    else:
        # Concierge/general: manager handles
        await manager_handler(issue, msg)
```

Channel configuration in `.agent/channels.json`:

```json
{
  "channels": {
    "discord/#general": {
      "type": "concierge",
      "description": "General intake, quick questions, cross-topic"
    },
    "discord/#project-x": {
      "type": "department",
      "department": "coding",
      "repo": "junyaoc/project-x",
      "default_labels": ["project-x"]
    },
    "discord/#shopping": {
      "type": "department",
      "department": "research",
      "default_labels": ["shopping"]
    },
    "discord/#crypto": {
      "type": "department",
      "department": "research",
      "default_labels": ["crypto"]
    },
    "discord/#inbox": {
      "type": "inbox",
      "description": "Agent posts here when user attention needed"
    },
    "discord/#journal": {
      "type": "journal",
      "description": "Agent self-reflection and pattern tracking"
    }
  }
}
```

### Manager: The Brain

The manager is triggered by GitHub Issue events (new issue, new comment). Its input is always a GitHub Issue. It never sees raw channel messages.

```python
async def manager_handler(issue, msg):
    # 1. Load context
    context = {
        # GitHub (authoritative state)
        "issue":       gh.get_issue_with_comments(issue.number, last=20),
        "board":       gh.get_project_board_summary(),
        "active":      gh.issues(state="open", label="in-progress"),
        "related":     gh.search_issues(extract_keywords(msg.content)),
        "open_prs":    gh.pulls(state="open"),
        "channels":    read(".agent/channels.json"),

        # DB (search aid)
        "semantic":    db.search(msg.content, limit=5),
    }

    # 2. Manager decides and acts (LLM call with tools)
    response = await llm(
        system=load_system_prompt(),
        context=context,
        tools=ALL_MANAGER_TOOLS,
    )

    # 3. All responses go to GitHub first
    gh.add_comment(issue.number, response.text)

    # 4. Execute any actions (labels, board, dispatch, close, self-evolve)
    for action in response.tool_calls:
        await execute(action)

    # 5. Reply in the source channel
    source = extract_source_channel(issue)
    await adapters[source].reply(response.text)
```

### Manager Tool Categories

#### Office Tools (GitHub state management)
- `create_issue(title, body, labels)` — new task
- `update_issue(number, title?, body?, state?)` — modify task
- `close_issue(number, summary)` — complete a task with final summary
- `add_comment(number, body)` — record on issue thread
- `add_label(number, label)` / `remove_label(number, label)` — state transitions
- `move_card(number, column)` — board management (Backlog/In Progress/Review/Done)
- `link_issues(from, to)` — cross-references
- `assign_issue(number, user)` — assign for attention
- `create_gist(filename, content)` — large artifact storage

#### Department Tools (dispatch)
- `spawn_coding_agent(issue, task, repo, agent="claude-code"|"codex")` — coding work
- `spawn_research(issue, query)` — web research

#### Communication Tools
- `post_to_channel(channel, content)` — send to specific channel
- `notify_inbox(content)` — surface to user attention

#### Self-Evolution Tools (system modification)
- `update_prompt(diff_description)` — propose change to `.agent/prompt.md` via PR
- `update_knowledge(file, content, reason)` — update `.agent/knowledge/*.md` via PR
- `update_channels(changes, reason)` — modify `.agent/channels.json` via PR
- `create_tool(name, code, description)` — create `.agent/tools/{name}.py` + update manifest via PR
- `modify_tool(name, code, reason)` — modify existing tool via PR
- `update_model_config(changes, reason)` — change model preferences via PR

#### Direct Response
- `reply(content)` — respond in the source channel

### Data Flow

#### Every Interaction (GitHub as bus)

```
User says anything (any channel)
    │
    Concierge:
    ├── find/create GitHub Issue           ~400ms
    ├── add message as comment             ~200ms
    │                                      (parallel with LLM thinking)
    Manager or Department:
    ├── read issue + context               ~300ms
    ├── LLM decides                        ~1-5s (dominates)
    ├── post response as comment           ~200ms
    ├── execute actions (labels, board)    ~200ms
    └── reply in source channel            ~100ms
    ─────────────────────────────────────
    TOTAL: ~2-6s (LLM dominates, GitHub overhead is noise)
```

#### Top-Down (Concierge → Manager → Department)

```
User in #general: "change footer to XYZ"
    │
    Concierge: create Issue #72, label incoming
    │
    Manager reads #72:
    ├── decides: coding task for project-x
    ├── add_label(#72, "in-progress", "project-x")
    ├── move_card(#72, "In Progress")
    ├── spawn_coding_agent(#72, "change footer ABC→XYZ", repo="project-x")
    ├── post_to_channel("#project-x", "starting footer change — #72")
    └── reply("on it, tracking in #project-x as #72")
```

#### Bottom-Up (User → Department → Office)

```
User in #project-x: "auth broken on safari"
    │
    Concierge: create Issue #73, labels ["incoming", "project-x"]
    │
    Department handles directly (channel type = department):
    ├── add_label(#73, "bug", "in-progress")
    ├── investigates, posts progress as comments on #73
    ├── opens PR #74 linked to #73
    ├── add_label(#73, "needs-review")
    └── notify_inbox("safari auth fix ready — PR #74")

    Manager sees #73 next time it reads office state.
    Full context available from any channel.
```

#### GitHub-Direct (User comments on Issue)

```
User comments on Issue #73 directly on GitHub
    │
    Webhook fires → manager_handler(issue=#73, msg=comment)
    │
    Manager reads #73 + new comment, acts accordingly
    Posts to discord/#project-x if relevant
```

### Issue Lifecycle

```
QUICK (lives seconds):
  create → user msg → agent reply → close
  Labels: quick
  Example: "what's btc price" → "$67k" → closed

TASK (lives hours/days):
  create → dispatch → department works → progress comments
  → PR → review → done → close
  Labels: in-progress → needs-review → done
  Example: "change footer to XYZ"

ONGOING (lives indefinitely):
  create → multiple conversations over time
  → living summary updated periodically → never closed
  Labels: ongoing
  Example: "ETH staking research" — revisited weekly
```

### DB Role (Search Index Only)

The DB is NOT a source of truth. It is a functional component that helps the agent think.

```sql
-- Search index: embeddings over issue content for semantic search
CREATE TABLE embeddings (
    github_issue INTEGER,
    comment_id TEXT,
    content TEXT,
    embedding VECTOR(1536),
    created_at TIMESTAMP
);

-- Thread mapping: which channel thread maps to which issue
CREATE TABLE thread_map (
    source TEXT,             -- "discord/#general"
    thread_id TEXT,          -- discord message/thread ID
    github_issue INTEGER,
    PRIMARY KEY (source, thread_id)
);

-- Session tokens: for resuming CLI agents
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    github_issue INTEGER,
    agent_type TEXT,         -- "claude-code", "codex"
    session_token TEXT,
    status TEXT,
    created_at TIMESTAMP
);

-- Working scratch: intermediate agent state during long operations
CREATE TABLE scratch (
    id TEXT PRIMARY KEY,
    github_issue INTEGER,
    content TEXT,
    created_at TIMESTAMP
);
```

`rm -rf mori.db` → rebuild embeddings from GitHub Issues, rebuild thread_map from issue metadata. Zero data loss.

### Self-Evolution

The manager can modify its own system through PRs to `.agent/`:

| What | How | Governance |
|---|---|---|
| Learn a preference | PR to `.agent/knowledge/preferences.md` | Auto-merge |
| Add knowledge | PR to `.agent/knowledge/{topic}.md` | Auto-merge |
| Update channel map | PR to `.agent/channels.json` | Auto-merge with notification |
| Create a tool | PR adding `.agent/tools/{name}.py` + manifest update | Auto-merge with notification |
| Modify a tool | PR changing `.agent/tools/{name}.py` | Auto-merge with notification |
| Change own prompt | PR to `.agent/prompt.md` | Requires human review |
| Change model config | PR to `.agent/models.json` | Requires human review |
| Delete a tool | PR removing from `.agent/tools/` | Requires human review |

All evolution is visible as git history. `git log .agent/` shows exactly how the system grew.

### Multi-Model Support

LiteLLM Router with model tiers defined in `.agent/models.json`:

```json
{
  "tiers": {
    "fast": {
      "models": ["anthropic/claude-haiku-4-5", "gemini/gemini-2.5-flash"],
      "fallback": ["gemini/gemini-2.5-flash"],
      "use_for": "quick questions, status checks, classification"
    },
    "smart": {
      "models": ["anthropic/claude-sonnet-4-6"],
      "fallback": ["openai/gpt-4o"],
      "use_for": "manager decisions, complex reasoning, research"
    },
    "coding": {
      "agents": ["claude-code", "codex"],
      "use_for": "spawned as CLI subprocess for coding tasks"
    }
  },
  "manager_default": "smart",
  "concierge_classify": "fast"
}
```

### Artifact Pipeline

When departments (coding agents) produce output:

1. Stream progress to DB scratch (working state)
2. Post milestone summaries as GitHub Issue comments
3. Small artifacts (diffs, test results) → inline in issue comments
4. Large artifacts (full logs, screenshots) → GitHub Gists, linked from comments
5. Code changes → PRs linked to the issue
6. On completion → close issue or update living summary, move board card, notify #inbox

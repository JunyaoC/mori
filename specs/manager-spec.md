# Spec: Manager Agent

## Role

The manager is the single persistent agent. It is the only LLM that always runs. Everything goes through it.

## Input

A normalized message from any channel adapter:

```python
Message(
    content="change footer text from ABC to XYZ",
    author="jun",
    source="discord/#general",
    thread=None,
)
```

## Context Loading

Before every decision, the manager loads current state from GitHub:

```python
state = {
    "active":     gh.issues(state="open", label="in-progress"),
    "blocked":    gh.issues(state="open", label="needs-attention"),
    "needs_input": gh.issues(state="open", label="needs-input"),
    "board":      gh.get_project_board(),
    "open_prs":   gh.pulls(state="open"),
    "related":    gh.search_issues(extract_keywords(msg)),
    "channels":   read(".agent/channels.json"),
}
```

## Decision Space

The manager has these tools and decides which to use:

### Office Tools (state management)
- `create_issue(title, body, labels)` — new task
- `update_issue(number, ...)` — modify existing task
- `close_issue(number)` — complete a task
- `add_comment(number, body)` — log to task thread
- `add_label(number, label)` / `remove_label(number, label)` — state transitions
- `move_card(number, column)` — board management
- `link_issues(from, to)` — cross-references
- `assign_issue(number, user)` — needs someone's attention
- `create_gist(filename, content)` — large artifact storage

### Department Tools (dispatch)
- `spawn_coding_agent(issue, task, agent="claude-code"|"codex")` — coding work
- `spawn_research(issue, query)` — web research

### Communication Tools
- `post_to_channel(channel, content)` — send to specific channel
- `notify_inbox(content)` — surface to user attention

### Self-Evolution Tools
- `update_knowledge(file, content)` — update `.agent/knowledge/`
- `create_tool(name, code, manifest_entry)` — new tool
- `modify_tool(name, code)` — improve existing tool

### Direct Response
- `reply(content)` — respond in the source channel

## Decision Heuristics

The manager's system prompt includes guidance for common patterns:

### Quick question (no task needed)
- Factual lookup, simple calculation, brief opinion
- Reply directly, no issue created

### New task
- User requests work that takes multiple steps
- Create issue → label with department → dispatch if appropriate → confirm in source channel

### Continue existing task
- Message relates to an open issue (keyword match or explicit reference)
- Add comment to issue → continue work → post in department channel

### Needs attention
- A department completes work, or something is blocked
- Post to inbox channel → assign issue to user

### Cross-reference
- Message in one context is relevant to another task
- Link issues → add comment noting the connection

## System Prompt Structure

```
.agent/prompt.md                   — personality, decision heuristics, model preferences
.agent/knowledge/preferences.md   — learned user preferences
.agent/knowledge/*.md              — any other accumulated knowledge
.agent/channels.json               — workspace topology
```

All loaded and concatenated into the system prompt for every manager call.

## Living Summary Protocol

When the manager creates or updates a task, the first comment on the issue is a living summary:

```markdown
## Status
[current status]

## Context
[why this task exists, key decisions]

## Progress
[what's been done]

## Open
[what's still needed]

## Related
[linked issues]
```

The manager updates this after every significant interaction on the issue.

## Example Flows

### "what happened to btc price"

1. Manager loads state — no related open issues
2. Decides: quick question, no task needed
3. Uses web_search tool
4. Replies directly in source channel

### "change footer text from ABC to XYZ"

1. Manager loads state — sees project-x in channels.json
2. Decides: new task, coding department
3. Creates Issue #65, label: `in-progress`, board: "In Progress"
4. Posts to #project-x: "starting footer change — #65"
5. Replies in #general: "on it, tracking in #project-x #65"
6. Spawns claude-code on the project repo
7. Claude-code streams progress → issue comments
8. On completion: manager updates issue, posts to #inbox

### "i need a new wallet"

1. Manager loads state — sees shopping in channels.json
2. Decides: new task, research department
3. Creates Issue #66, label: `shopping`
4. Posts to #shopping: "starting wallet research — #66"
5. Replies in #general: "looking into it, check #shopping"
6. Spawns research agent
7. Research posts findings as issue comments + #shopping messages
8. When done: manager closes issue, posts summary to #inbox

### User comments directly on GitHub Issue #65

1. Webhook fires → manager receives message with source "github/issue/65"
2. Manager loads issue context
3. Decides: continue existing task
4. Acts accordingly
5. Posts update to #project-x

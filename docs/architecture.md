# Architecture

## The Hotel Model

mori uses a hotel metaphor for its architecture. Every component has a clear role and boundary.

### Roles

| Role | What it is | What it does | What it doesn't do |
|---|---|---|---|
| **Guest** | The user | Talks to any channel or GitHub directly | — |
| **Concierge** | Channel adapters (Discord, CLI, Slack) | Normalizes input, hands to manager | Doesn't decide, doesn't execute |
| **Manager** | One LLM agent | Reads office state, decides next action, dispatches work, follows up, reports back | Doesn't do heavy execution itself |
| **Office** | GitHub (Issues, Board, PRs, Labels) | Stores all state. Single source of truth. | Doesn't think, doesn't decide |
| **Departments** | Spawned agents (claude-code, codex, research) | Execute tasks, report results back through office | Don't decide priorities, don't talk to guest without manager knowing |
| **Inbox** | #inbox / #attention channel | Surfaces things that need user action | — |

### The Rules

1. **Every mutation goes through the office.** Channels are views. GitHub is the source of truth.
2. **The manager is the only agent that always runs.** Departments are spawned on demand.
3. **Departments always sync back to the office.** Every action is reflected as an issue update, comment, label change, or PR.
4. **The guest can talk to anyone** — concierge, office, or department directly. State is always consistent because the office is always the authority.

### Flow

```
Guest says something (any channel)
      │
 CONCIERGE normalizes the message
      │
 MANAGER reads office state:
      ├── active issues
      ├── board status
      ├── open PRs
      ├── related context
      │
 MANAGER decides:
      ├── is this a quick reply? → respond directly
      ├── is this a new task? → create issue, dispatch to department
      ├── is this about an existing task? → update issue, continue work
      ├── does something need user attention? → post to inbox
      │
 MANAGER executes:
      ├── mutates office (GitHub) FIRST
      ├── dispatches to department if needed
      └── posts to appropriate channels
```

### State Model

There is no custom database. All state maps to GitHub primitives:

| State | GitHub primitive |
|---|---|
| Task | Issue |
| Conversation log | Issue comments |
| Task status | Labels + Project Board column |
| Code output | PR linked to issue |
| Large artifacts | Gist linked from comment |
| Cross-task context | Issue references (#42) |
| What needs attention | Label: `needs-attention` + assigned to user |
| Agent config | Files in `.agent/` |
| Tools | Scripts in `.agent/tools/` |
| Learned knowledge | Markdown in `.agent/knowledge/` |
| Agent evolution history | Git log of `.agent/` |
| Scheduled work | GitHub Actions |
| Governance | Branch protection rules |

### Context Loading

Every interaction, the manager loads state from GitHub:

```python
state = {
    "active":   gh.issues(state="open", label="in-progress"),
    "blocked":  gh.issues(state="open", label="needs-attention"),
    "board":    gh.get_project_board(),
    "open_prs": gh.pulls(state="open"),
    "related":  find_related_issues(msg),
}
```

This is the agent's "working memory" — always live, always correct, no sync needed.

### Continuity Model

**Per-task continuity**: each issue has a living summary (pinned/first comment) that captures the current "state of mind" — not just what happened, but what was decided, what's open, what's next. Any new session reads this and picks up immediately.

**Cross-task continuity**: the board gives a global view. Related issues are linked. The manager sees everything.

**Cross-channel continuity**: every channel interaction is logged as an issue comment with `[source]` tag. The issue thread is the canonical conversation, channels are just windows into it.

### Multi-Model Support

The manager routes to different LLM providers via LiteLLM:

- Quick questions → fast/cheap model (haiku, gemini-flash)
- Coding tasks → spawn claude-code or codex as subprocess
- Deep research → capable model (sonnet, o3)
- The manager itself → consistent capable model

Model preferences live in `.agent/prompt.md` and evolve over time.

### Artifact Pipeline

When departments (coding agents) produce output:

1. Stream progress as issue comments (periodic updates)
2. Small artifacts (diffs, test results) → inline in comments
3. Large artifacts (full logs, screenshots) → GitHub Gists, linked from comments
4. Code changes → PRs linked to the issue
5. On completion → manager updates living summary, notifies inbox if needed

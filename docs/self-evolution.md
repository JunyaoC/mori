# Self-Evolution

mori is not a static system. It grows its own tools, accumulates knowledge, and refines its own behavior — all through the same git workflow as any other code change.

## The Three Loops

### Loop 1: Per-Interaction (every message)

Standard operation. Manager reads state, acts, updates issues. No evolution.

### Loop 2: Per-Task Reflection (when a task closes)

When an issue is closed, the manager reflects:

- Did I have the right tools? What was missing?
- Did I struggle with something I shouldn't have?
- Did the user correct me on preferences or facts?
- Should any knowledge files be updated?

If changes are needed, the manager opens a PR to `.agent/`.

### Loop 3: Periodic Meta-Reflection (cron — weekly)

The manager reviews all recent activity and considers:

- Are there recurring tasks that need a dedicated tool?
- Are any tools underperforming or unused?
- Are there patterns in user corrections worth codifying?
- Should the base prompt be updated?

This runs as a GitHub Actions cron job.

## What Evolves

### Tools

Tools are scripts in `.agent/tools/`. The agent writes them:

```
1. Agent encounters a need
   "I keep searching crypto prices via web_search — slow and expensive"

2. Agent creates a tool
   → writes .agent/tools/price_check.py
   → updates manifest.json
   → opens PR with rationale

3. Agent uses the tool in future tasks

4. Agent improves the tool when it breaks
   → PR with diff + reasoning
```

### Knowledge

`.agent/knowledge/` accumulates learned facts:

- `preferences.md` — user preferences learned over time
- `sources.md` — which data sources are reliable
- Any other knowledge files the agent creates

### Prompt

`.agent/prompt.md` is the agent's personality and instructions. It can propose changes to itself — model selection heuristics, communication style, decision-making rules.

### Workspace Map

`.agent/channels.json` maps the channel topology. When the user adds a new channel or changes how channels are used, the agent updates this.

## Governance

Not everything auto-merges:

| Risk level | What | Action |
|---|---|---|
| Low | Knowledge updates (learned preferences) | Auto-merge |
| Medium | New tool creation, prompt refinements | Auto-merge with notification |
| High | Base prompt changes, tool deletion, model changes | Requires human review |

This is enforced via GitHub branch protection rules and PR labels.

## Evolution Record

The git history of `.agent/` IS the evolution record. You can:

- `git log .agent/tools/` — see when each tool was created and why
- `git log .agent/knowledge/` — see what the agent has learned over time
- `git log .agent/prompt.md` — see how the agent's personality has evolved
- `git diff` any two points — see exactly what changed and why

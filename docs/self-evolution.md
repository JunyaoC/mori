# Self-Evolution

mori's manager can modify its own system. All changes go through PRs to `.agent/`, giving you version control, diffs, review, and rollback.

## What The Manager Can Evolve

| File | What | Example |
|---|---|---|
| `.agent/knowledge/*.md` | Learned facts and preferences | "user prefers non-custodial solutions" |
| `.agent/channels.json` | Channel topology | Add #crypto as a department channel |
| `.agent/tools/*.py` | Custom tools | Create a `crypto_price` tool to avoid repeated web searches |
| `.agent/tools/manifest.json` | Tool registry | Register new tool with parameters |
| `.agent/prompt.md` | Own personality and instructions | "be more concise" |
| `.agent/models.json` | Model tier configuration | Switch fast tier from haiku to gemini-flash |

## How It Works

Every self-evolution tool creates a PR:

```python
# Manager notices user always asks for non-custodial options
update_knowledge(
    file="preferences.md",
    content="- prefers non-custodial crypto solutions\n",
    reason="user stated preference in issue #42",
)
# → creates branch: agent/knowledge-update-{timestamp}
# → commits change to .agent/knowledge/preferences.md
# → opens PR with reason in description
# → governance rules determine merge behavior
```

## Governance

| Risk | What | Merge Policy |
|---|---|---|
| Low | Knowledge updates (preferences, facts) | Auto-merge |
| Medium | New tools, tool changes, channel map updates | Auto-merge + notification to #inbox |
| High | Prompt changes, model config changes, tool deletion | Requires human review, notification to #inbox |

Governance is enforced via GitHub branch protection rules and PR labels:
- Label `auto-merge` → CI merges after checks pass
- Label `needs-review` → waits for human approval
- All PRs notify #inbox regardless

## The Three Reflection Loops

### Loop 1: Per-Interaction

During normal operation. If the manager notices something worth remembering (user correction, stated preference, repeated pattern), it calls the appropriate evolution tool immediately.

### Loop 2: Per-Task (on issue close)

When an issue is closed, the manager reflects:

- Did I have the right tools for this? → `create_tool` if not
- Did the user correct me? → `update_knowledge`
- Was I slow or awkward at anything? → consider `update_prompt`
- Is there a related channel that should exist? → `update_channels`

### Loop 3: Periodic Meta-Reflection (cron)

Weekly GitHub Actions trigger. Manager reviews recent closed issues and considers:

- Recurring task patterns that need dedicated tools
- Underperforming or unused tools to remove
- Prompt refinements based on accumulated feedback
- Model tier adjustments based on performance patterns

Results posted to #journal and standing reflection issue.

## Evolution Record

`git log .agent/` is the full evolution history:

```
git log --oneline .agent/
a1b2c3d update knowledge: user prefers non-custodial (from #42)
d4e5f6g create tool: crypto_price (avoid repeated web searches)
h7i8j9k update channels: add #crypto department
l0m1n2o update prompt: add "be concise" principle (weekly reflection)
```

Every change has a reason. Every change is reversible. The system's growth is fully auditable.

# Channel Architecture

## Principle: GitHub Is the Bus, Channels Are Windows

Every message from every channel becomes a GitHub Issue comment. The channel is just how the message arrived. GitHub has the complete record.

## Channel Types

Defined in `.agent/channels.json`:

| Type | Role | Example | How messages are handled |
|---|---|---|---|
| **concierge** | General intake | #general, CLI | Concierge finds/creates issue → manager handles |
| **department** | Specialized work | #project-x, #crypto | Concierge finds/creates issue → department handles directly |
| **inbox** | User notifications | #inbox | Agent posts here, user reads. Not an input channel. |
| **journal** | Agent reflections | #journal | Agent posts here for self-reflection. Not an input channel. |

## Adapter Contract

Every channel adapter implements:

```python
class Adapter:
    async def listen() -> AsyncIterator[NormalizedMessage]
    async def post(channel: str, content: str)
    async def reply(msg: NormalizedMessage, content: str)
```

```python
@dataclass
class NormalizedMessage:
    content: str          # what was said
    author: str           # who said it
    source: str           # "discord/#general", "cli", "github/issue/72"
    thread_id: str | None # platform-specific thread/message ID
    attachments: list     # files, images
```

## Concierge Routing

The concierge is channel-aware but NOT an LLM. It's deterministic routing:

```python
async def concierge(msg: NormalizedMessage):
    channel_config = channels.get(msg.source)

    # Find existing issue or create new one
    issue = (
        db.get_issue_for_thread(msg.source, msg.thread_id)
        or gh.find_related_open_issue(msg, channel_config)
        or gh.create_issue(
            title=summarize(msg.content),
            labels=channel_config.default_labels + ["incoming"],
        )
    )

    # Record the message
    gh.add_comment(issue.number,
        f"**[{msg.source}] @{msg.author}**: {msg.content}")

    # Route based on channel type
    if channel_config.type == "department":
        await department_handler(issue, msg, channel_config)
    else:
        await manager_handler(issue, msg)
```

## Bidirectional Flow

### Channel → GitHub

```
User says something in Discord
  → concierge creates/finds issue
  → records as issue comment: [discord/#general] @jun: ...
  → manager or department handles it
  → response recorded as issue comment
  → response also posted back to Discord channel
```

### GitHub → Channel

```
User comments directly on GitHub Issue
  → webhook fires
  → manager handles it
  → response recorded as issue comment
  → if issue has a mapped department channel, post there too
```

### Agent → Channel

```
Agent needs user attention
  → post to #inbox
Agent completes background work
  → post to department channel + #inbox
Agent self-reflects
  → post to #journal
```

## Adding New Channels

### New adapter (e.g., Slack)

1. Write adapter implementing the contract (~80-100 lines)
2. Register in channel registry
3. Update `.agent/channels.json` with Slack channels
4. Done. Everything else (issue creation, routing, manager) works unchanged.

### New department channel

The agent can propose this itself:

```
Agent notices repeated crypto questions in #general
  → update_channels({"discord/#crypto": {"type": "department", ...}})
  → PR to .agent/channels.json
  → auto-merges with notification to #inbox
```

Or the user can request it directly.

## Channel Configuration Example

```json
{
  "channels": {
    "discord/#general": {
      "type": "concierge",
      "description": "General intake, quick questions"
    },
    "discord/#project-x": {
      "type": "department",
      "department": "coding",
      "repo": "junyaoc/project-x",
      "default_labels": ["project-x"]
    },
    "discord/#crypto": {
      "type": "department",
      "department": "research",
      "default_labels": ["crypto"]
    },
    "discord/#shopping": {
      "type": "department",
      "department": "research",
      "default_labels": ["shopping"]
    },
    "discord/#inbox": {
      "type": "inbox",
      "description": "Completions, blockers, digests"
    },
    "discord/#journal": {
      "type": "journal",
      "description": "Agent self-reflection"
    }
  }
}
```

# Channel Architecture

## Principle: Channel Agnosticism

mori doesn't care where a message comes from. Every channel adapter produces the same normalized message format. The manager sees no difference between Discord, CLI, or Slack.

## Normalized Message

```python
@dataclass
class Message:
    content: str          # what was said
    author: str           # who said it
    source: str           # "discord/#general", "cli", "slack/#dev"
    thread: str | None    # issue number if continuing a thread
    attachments: list     # files, images, etc.
```

## Adapter Contract

Every channel adapter implements:

```python
class Adapter:
    async def listen() -> AsyncIterator[Message]    # receive messages
    async def post(channel: str, content: str)      # send to specific channel
    async def reply(msg: Message, content: str)     # reply to a message
```

## Channel Roles

Channels are mapped in `.agent/channels.json`:

```json
{
  "concierge": ["discord/#general", "cli", "slack/#random"],
  "departments": {
    "project-x": "discord/#project-x",
    "shopping": "discord/#shopping",
    "crypto": "discord/#crypto"
  },
  "inbox": "discord/#inbox",
  "journal": "discord/#journal"
}
```

This map is maintained by the agent and evolves over time.

### Concierge Channels

General intake. User says anything here, manager figures out where it belongs.

### Department Channels

Where work happens visibly. Each maps to a label/topic in GitHub. Work on "project-x" issues shows up in the project-x channel.

### Inbox

Where the agent surfaces things that need user attention:
- Completed long-running tasks
- Blocked items needing decisions
- Daily/weekly digests
- PR review requests

### Journal

Where the agent posts self-reflections and pattern observations. The user can read it or ignore it.

## Bidirectional Sync

Every channel interaction syncs to GitHub:

```
User says something in #project-x
  → comment added to relevant issue: "[discord/#project-x] @user: ..."

Agent updates an issue on GitHub
  → posts to mapped department channel: "updated #42: ..."

User comments directly on a GitHub issue
  → webhook fires → manager picks up → posts to relevant channel if needed
```

The issue thread is always the complete record. Channel messages may be partial views.

## Adding New Channels

To add a new channel (e.g., Telegram):

1. Write an adapter implementing the contract (~80-100 lines)
2. Update `.agent/channels.json`
3. Done. No other changes needed.

The agent can also propose new channel mappings. If the user starts discussing a new topic frequently, the agent might suggest: "Should I create a #research channel for these kinds of tasks?"

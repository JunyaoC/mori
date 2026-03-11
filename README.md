# mori 森

**The forest manages itself. You just walk through it.**

mori is a self-evolving personal AI agent system rooted in GitHub. You talk to it from anywhere — Discord, CLI, Slack — and it handles everything: creates tasks, dispatches work, grows its own tools, and keeps perfect context across every session and channel.

無為無所不為 — *do nothing, yet nothing is left undone.*

## Philosophy

A forest is self-evolving, interconnected, and alive. Many trees, one root system. mori works the same way:

- **You just talk.** Say something in any channel. mori figures out what to do, where to do it, and who to tell.
- **GitHub is the office.** All state lives in Issues, PRs, and Project Boards. No shadow databases, no sync problems.
- **The agent is the manager.** One brain reads the state, makes decisions, dispatches work, follows up.
- **Channels are concierges.** Discord, CLI, Slack — they're just intake windows. The work happens in the office.
- **Tools and knowledge evolve.** The agent writes its own tools, accumulates knowledge, and improves itself — all through PRs.

## Architecture

```
GUEST (you)
  │
  ├── CONCIERGE (channels — intake)
  │     Discord, CLI, Slack, webhooks
  │
  ├── MANAGER (agent — the brain)
  │     reads office, decides, dispatches, follows up
  │
  ├── OFFICE (GitHub — state)
  │     Issues, Board, PRs, Labels
  │
  └── DEPARTMENTS (workers — execution)
        claude-code, codex, research agents
```

## Status

Early design phase. See [docs/](./docs/) for architecture and [specs/](./specs/) for design specs.

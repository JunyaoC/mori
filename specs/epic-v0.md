# Epic: mori v0 — Proof of Concept

## Goal

A working system where the user talks in Discord, the manager agent processes it, and all state lives in GitHub. End-to-end flow for both quick replies and task dispatch.

## Success Criteria

The following interactions work:

1. **Quick question** — "what happened to btc price" → agent answers in-place, no issue created
2. **Task dispatch** — "change footer text from ABC to XYZ" → agent creates issue, spawns coding agent, reports progress in project channel, notifies inbox when done
3. **Shopping thread** — "i need a new wallet" → agent creates issue in shopping, starts research in #shopping channel
4. **Cross-channel continuity** — start a topic in #general, continue it in #tasks or directly on the GitHub issue — context is preserved
5. **Cron reflection** — hourly journal entry posted to #journal

## Milestones

### M1: Office Setup
- [ ] GitHub repo with Project Board (columns: Backlog, In Progress, Review, Done)
- [ ] Label taxonomy: `in-progress`, `needs-attention`, `needs-input`
- [ ] `.agent/prompt.md` — initial manager personality
- [ ] `.agent/channels.json` — initial workspace map
- [ ] `.agent/tools/manifest.json` — starter tools

### M2: Manager Core
- [ ] Context loader — queries GitHub Issues/Board/PRs to build state
- [ ] Manager agent — single LLM call with tools that decides what to do
- [ ] Office tools — create issue, update issue, close issue, add label, move board card, add comment, link issues
- [ ] Decision logic — quick reply vs. create task vs. continue existing task

### M3: Discord Adapter
- [ ] discord.py bot that listens to configured channels
- [ ] Normalizes messages into standard format
- [ ] Routes to manager
- [ ] Posts manager responses to correct channels
- [ ] Handles GitHub webhook events (issue updates → channel notifications)

### M4: Department — Coding Agent
- [ ] Spawn claude-code as subprocess with `--output-format stream-json`
- [ ] Artifact pipeline — stream progress as issue comments
- [ ] Small artifacts inline, large artifacts to gists
- [ ] PR creation linked to issue
- [ ] Completion notification to inbox

### M5: Department — Research
- [ ] Web search + summarization for ephemeral tasks
- [ ] Results posted as issue comments
- [ ] Auto-close when research is complete

### M6: Cron + Self-Reflection
- [ ] GitHub Actions workflow for hourly/daily triggers
- [ ] Reflection loop — review recent activity, post journal
- [ ] Knowledge accumulation — update `.agent/knowledge/` via PRs

### M7: Multi-Model
- [ ] LiteLLM integration
- [ ] Model selection per task type
- [ ] Fallback chains

## Non-Goals for v0

- Slack/CLI adapters (Discord only for v0)
- Full self-evolution (tool creation by agent)
- Voice channels
- Browser automation
- Multiple concurrent department workers

## Tech Stack

- **Language**: Python (asyncio)
- **Discord**: discord.py
- **GitHub API**: PyGitHub or gh CLI
- **LLM**: Claude API (direct for v0, LiteLLM for M7)
- **Coding agent**: Claude Code CLI (subprocess)
- **Cron**: GitHub Actions

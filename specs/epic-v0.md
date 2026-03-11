# Epic: mori v0 — Proof of Concept

## Goal

A working system where the user talks from any channel (Discord, CLI, or directly on GitHub), the manager agent or department handles it, mutations are reflected in GitHub (office), and conversation continuity is maintained via a local database (scratchpad).

## Success Criteria

The following interactions work end-to-end:

1. **Quick question (read-only, DB only)** — "what happened to btc price" → agent answers in-place, conversation stored in DB, no issue created
2. **Task dispatch (top-down)** — "change footer text from ABC to XYZ" → manager creates issue, spawns coding agent, streams progress to DB, syncs milestones to GitHub issue, reports completion to #inbox
3. **Direct department interaction (bottom-up)** — user goes to #project-x and says "auth is broken on safari" → department handles it directly, creates issue, opens PR, syncs to office without needing manager
4. **Cross-channel continuity** — discuss staking in #general, then ask about it in #crypto → DB semantic search finds the prior conversation, GitHub issue links the context
5. **Conversational memory** — "what did we decide about staking last week?" → DB embedding search retrieves it, even if it was casual chat that never became an issue
6. **Cron reflection** — hourly journal entry, weekly self-reflection with knowledge PRs

## Architecture Overview

```
GitHub (Office)     = source of truth, all mutations
DB (Scratchpad)     = working memory, conversation history, embeddings
Channels            = concierge (intake) + departments (work)
Manager             = coordinator (not bottleneck)
Departments         = autonomous workers, sync back to office
```

See [docs/architecture.md](../docs/architecture.md) for full details.

## Milestones

### M1: Foundation
- [ ] GitHub Project Board (columns: Backlog, In Progress, Review, Done)
- [ ] Label taxonomy: `in-progress`, `needs-attention`, `needs-input`, `bug`, domain labels
- [ ] `.agent/prompt.md` — manager personality
- [ ] `.agent/channels.json` — workspace map
- [ ] `.agent/tools/manifest.json` — tool registry
- [ ] SQLite database schema (messages, threads, sessions, queue)
- [ ] Embedding setup (sqlite-vec or similar)

### M2: Manager Core
- [ ] Context loader — reads BOTH DB (conversation, semantic search) and GitHub (issues, board, PRs)
- [ ] Manager agent — LLM with tools, decides what to do
- [ ] Mutation rule enforcement — mutations → GitHub, reads → DB only
- [ ] Office tools: create_issue, update_issue, close_issue, add_label, move_card, add_comment, link_issues, create_gist
- [ ] DB tools: store_message, search_conversations, get_thread_context
- [ ] Decision logic: quick reply vs. create task vs. continue existing task vs. dispatch to department

### M3: Discord Adapter
- [ ] discord.py bot listening to configured channels
- [ ] Channel registry pattern (borrowed from NanoClaw)
- [ ] Message normalization to standard format
- [ ] Route to manager (concierge channels) or department (department channels)
- [ ] Post responses to correct channels, chunked at 2000 chars
- [ ] GitHub webhook listener (issue updates → channel notifications)
- [ ] Typing indicators during LLM processing

### M4: Department — Coding Agent
- [ ] Spawn claude-code as subprocess (borrowed from PicoClaw's `claude_cli_provider`)
- [ ] `--output-format stream-json` for structured output capture
- [ ] Stream all progress to DB in real-time
- [ ] Sync milestone summaries to GitHub Issue comments
- [ ] PR creation linked to issue
- [ ] Codex CLI support as alternative (`codex --json`)
- [ ] Session tracking (session IDs stored in DB for potential resume)
- [ ] Completion/blocker notification to #inbox
- [ ] **Bottom-up**: department can handle user messages directly without manager

### M5: Department — Research
- [ ] Web search + summarization
- [ ] Results streamed to DB, summary posted to GitHub Issue
- [ ] Auto-close issue when research is complete
- [ ] **Bottom-up**: user can ask directly in a department channel

### M6: Cron + Self-Reflection
- [ ] GitHub Actions workflow for hourly/daily triggers
- [ ] Reflection loop — review DB (recent conversations) + GitHub (recent issues)
- [ ] Journal entry posted to #journal channel + standing GitHub Issue
- [ ] Knowledge accumulation — update `.agent/knowledge/` via PRs
- [ ] Self-improvement: identify missing tools, refine prompts

### M7: Multi-Model
- [ ] LiteLLM Router integration (borrowed from Nanobot pattern)
- [ ] Model tiers: fast (haiku/flash), smart (sonnet), cheap (gemini-flash)
- [ ] Fallback chains with error classification (borrowed from PicoClaw)
- [ ] Cooldown tracking for failing providers (borrowed from IronClaw)
- [ ] Manager can choose model tier per-task
- [ ] Cost tracking per interaction

### M8: Semantic Memory
- [ ] Embedding generation for all messages (on write)
- [ ] Semantic search tool for manager and departments
- [ ] "What did we discuss about X?" works across all channels and time
- [ ] Memory decay — recent conversations weighted higher
- [ ] Consider Mem0 integration as alternative to DIY embeddings

## Non-Goals for v0

- Slack/CLI adapters (Discord + GitHub only for v0)
- Full self-evolution (agent creating its own tools)
- Voice channels
- Browser automation
- WASM sandboxing
- RAG over large document corpora
- Multiple concurrent department instances

## Tech Stack

- **Language**: Python 3.11+ (asyncio)
- **Discord**: discord.py
- **GitHub API**: PyGitHub or `gh` CLI
- **Database**: SQLite + sqlite-vec (embeddings)
- **LLM**: Claude API direct (v0), LiteLLM Router (M7)
- **Coding agent**: Claude Code CLI, Codex CLI (subprocess)
- **Embedding**: OpenAI text-embedding-3-small or local alternative
- **Cron**: GitHub Actions

## Borrowed Patterns

| Pattern | Source | Used In |
|---|---|---|
| Channel registry (factory-based) | NanoClaw | M3 |
| CLI agent spawning (claude-code, codex subprocess) | PicoClaw | M4 |
| Asymmetric error recovery | NanoClaw | M3, M4 |
| LiteLLM provider registry + retry wrapper | Nanobot | M7 |
| Fallback chain with error classification | PicoClaw | M7 |
| Cooldown-aware failover | IronClaw | M7 |
| Cost tracking + budget guard | IronClaw | M7 |

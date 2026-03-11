# Scouting Report: OpenClaw Ecosystem

Research conducted March 2026. This documents what exists, what to borrow, and what to avoid.

## Projects Evaluated

| Project | Language | LOC | Providers | Channels | Key Strength |
|---|---|---|---|---|---|
| OpenClaw | TypeScript | ~100K+ | ~8 | 22+ | Most features, largest ecosystem |
| NanoClaw | TypeScript | ~3,900 | 1 (Claude) | 5 | Minimal, self-modifying, container isolation |
| ZeroClaw | Rust | ~15-25K | 41+ | 27+ | Best provider abstraction, hybrid memory |
| PicoClaw | Go | ~8-12K | ~8+ | 14+ | CLI agent spawning, cleanest fallback chains |
| IronClaw | Rust | ~20-30K | ~9+ | 6+ | Production-grade failover, WASM sandbox, cost guard |
| Nanobot | Python | ~4,000 | 14+ | 10 | LiteLLM integration, closest to mori's stack |

## Feature Comparison (Relevant to mori)

### CLI Agent Spawning
- **PicoClaw**: `claude_cli_provider.go` + `codex_cli_provider.go` — spawns Claude Code and Codex as subprocesses. Best reference for mori's department pattern.
- **NanoClaw**: Uses Claude Code as its agent runtime inside containers.
- Others: Don't spawn CLI agents as subprocesses.

### Multi-Model
- **ZeroClaw**: 41 providers via Rust trait abstraction. `RouterProvider` with hint-based routing. `capabilities()` struct per provider.
- **PicoClaw**: Factory pattern + `FallbackChain` with cooldown tracking and error classification (retriable vs non-retriable).
- **IronClaw**: `FailoverProvider` with lock-free atomic health tracking, cooldown config, budget enforcement via `cost_guard.rs`.
- **Nanobot**: LiteLLM-based (same as mori plans). Provider registry with `find_by_model()` auto-detection, `chat_with_retry()`.

### Memory / Embeddings
- **ZeroClaw**: 9 memory backends. Hybrid search (FTS5 + vector via Qdrant). Memory decay. RAG module. Most comprehensive.
- **IronClaw**: pgvector + Reciprocal Rank Fusion hybrid search. Context compaction.
- **OpenClaw**: 96-file memory system with batch embedding pipelines. Over-engineered.
- **NanoClaw**: Simple per-group CLAUDE.md files. Minimal but effective.

### Channel Architecture
- **NanoClaw**: Factory-based `registerChannel()` pattern. Cleanest abstraction.
- **OpenClaw**: 73-file Discord module. Message chunking at 2000 chars, typing indicators.
- **ZeroClaw**: 27+ channels with per-channel formatting.

### Cron / Automation
- **NanoClaw**: Drift-free interval scheduling (anchor to previous time, not now()).
- **ZeroClaw**: SOP system (Standard Operating Procedures with approval gates).
- **IronClaw**: Routines engine with cron + event triggers.

### Self-Evolution
- **NanoClaw**: Claude Code rewrites NanoClaw's own source. Most aggressive self-modification.
- **IronClaw**: Dynamic WASM tool building (describe needs → system builds it).
- **ZeroClaw**: SkillForge module + estimation/learner.rs.

### Security
- **IronClaw**: Prompt injection defense, leak detection, WASM sandbox with capability-based permissions.
- **ZeroClaw**: Natural-language approval gating for dangerous operations.
- **NanoClaw**: OS-level container isolation.

## What mori Borrows

### Priority 1 (v0)
1. **Channel registry** from NanoClaw — factory-based adapter registration
2. **CLI agent spawning** from PicoClaw — claude-code/codex as subprocess
3. **Asymmetric error recovery** from NanoClaw — don't retry if user saw output
4. **Message normalization** — universal pattern across all projects

### Priority 2 (M7: Multi-Model)
5. **LiteLLM + retry** from Nanobot — same stack, directly applicable
6. **Error classification** from PicoClaw — retriable vs non-retriable before failover
7. **Cooldown tracking** from IronClaw — failing providers go on ice
8. **Cost tracking + budget guard** from IronClaw

### Priority 3 (Later)
9. **Hybrid search** from ZeroClaw — when .agent/knowledge/ grows large
10. **Memory decay** from ZeroClaw — recent knowledge weighted higher
11. **Context compaction** from IronClaw — summarize when hitting context limits
12. **SOP/Workflows** from ZeroClaw — structured multi-step procedures
13. **Prompt injection defense** from IronClaw — when mori faces untrusted input

## What mori Does Differently

| Dimension | Others | mori |
|---|---|---|
| State store | SQLite/Postgres/files | GitHub (source of truth) + SQLite (scratchpad) |
| Session model | Per-channel/user isolated sessions | One manager + autonomous departments, GitHub is the session |
| Memory | Custom embedding pipelines | DB embeddings for conversation, GitHub for decisions |
| Agent model | Single agent or container-isolated | Manager-department hierarchy with bottom-up autonomy |
| Self-evolution | Code self-modification (NanoClaw) | PRs to .agent/ (version-controlled, reviewable) |
| Cross-session | Hard problem (all projects struggle) | Not a problem — GitHub + DB have everything |

## What mori Does NOT Borrow
- OpenClaw's 242-file gateway (mori's gateway = manager + GitHub API)
- OpenClaw's 96-file memory system (GitHub + SQLite is mori's memory)
- Any custom database as primary state store (GitHub is primary)
- WASM sandboxing (subprocess isolation is enough for v0)
- Hardware/IoT features
- 22+ channel adapters (start with Discord)

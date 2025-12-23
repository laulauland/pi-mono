# jj-first Coding Agent

## Goal

Replace "chat with code changes as side effects" with **version control graph as primary artifact**. Each chat session is keyed to a jj change id. The prompt becomes the change description, agent work accumulates as file edits tracked by jj, and checkpointing is just `jj new`.

```
@ qmwxnhgl  Add Redis backend for rate limiting
│           ↳ session: rlvkptmz (root)
│           ↳ description: active summary
│           ↳ transcript: queryable on demand
│
◯ rlvkptmz  Add rate limiting to API
│
◯ main
```

## Why jj

- **Change IDs are stable** — survive rebase, amend, everything
- **Mutable history by default** — agent doesn't need commit ceremony
- **Working copy is just another change** — no staging area confusion
- **Conflicts surface in the graph** — not hidden until merge time

---

## Workflow

### 1. Start New Chat

```bash
jj-agent start "Add rate limiting to API"
```

This:
- Runs `jj new -m "Add rate limiting to API"` to create a new change
- Records the change id as the session root
- Starts the agent loop

### 2. Agent Works

Standard coding loop:
- Agent reads files, runs commands, makes edits
- All changes accumulate in the jj working copy
- No explicit commits needed — jj tracks everything

### 3. Description Stays Current

After each agent turn, the jj description is updated with a living summary:

```
Add rate limiting to API

Done:
- Token bucket implementation in src/ratelimit.ts
- Redis backend with connection pooling

Key decisions:
- Token bucket over sliding window (simpler)
- 100 req/min default, configurable per-tenant

Left to do:
- Per-endpoint limit overrides
- Dashboard metrics

Open questions:
- Should limits apply to WebSocket connections?
```

This happens automatically via hook on `agent_end`.

### 4. Context Overflow → New Session, Same Summary

When the 200k context fills up:
- Human or agent triggers `jj new` (checkpoint)
- New session starts
- Parent description loads as handoff context
- Full transcript queryable on demand via `query_ancestor`

Compression: 180k conversation → ~2k summary in parent description.

### 5. Human Reviews with jj diff

At any point:

```bash
jj diff           # see what agent changed in this session
jj log            # see the change graph
jj show @-        # see parent change
```

No special tooling needed. Standard jj workflow.

---

## Architecture

### pi-mono Integration

Build on `@mariozechner/pi-coding-agent` with custom components:

```
┌─────────────────────────────────────────────────────────────┐
│                      createAgentSession                      │
│                                                              │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────────┐  │
│  │ JjSessionMgr  │  │  jj-tools.ts  │  │ jj-describe.ts  │  │
│  │ (custom)      │  │ (custom tool) │  │ (hook)          │  │
│  └───────────────┘  └───────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### What pi-mono Provides

| Component | Purpose |
|-----------|---------|
| `pi-ai` | Multi-provider LLM (Anthropic, OpenAI, Google, Mistral) |
| `pi-agent-core` | Agent loop, streaming, tool execution |
| Built-in tools | read, bash, edit, write, grep, find, ls |
| Hook system | Intercept turn/agent events |
| Custom tools | Add jj-specific tools |
| Compaction | Context summarization (backup to jj description) |

### What We Build

| Component | Purpose | LOC |
|-----------|---------|-----|
| `JjSessionManager` | Store transcript by change id | ~150 |
| `jj-tools.ts` | `jj_new`, `jj_describe`, `query_ancestor` | ~100 |
| `jj-describe-hook.ts` | Update description on turn end | ~50 |
| CLI wrapper | `jj-agent start/resume` | ~100 |

---

## Session Storage

```
~/.agent/sessions/
└── {root_change_id}/
    └── transcript.jsonl
```

- Keyed by root change id of the session lineage
- Checkpoints (`jj new`) continue appending to same transcript
- Checkpoint changes store `[session: {root_id}]` in description to link back
- Preserves LLM cache (message array stays stable, just appends)

---

## Context Model

| Source | When Loaded | Purpose |
|--------|-------------|---------|
| Current description | Always | Active summary: done, left to do, decisions, blockers |
| Parent description | Always | Handoff context from previous session |
| Current diff | Always | What's changed in this session |
| Transcript | On-demand | Full conversation history, agent queries when needed |
| Parent diff/transcript | On-demand | Agent reaches back when needed |

---

## Agent Tools

Standard coding tools (read/write/bash) plus:

### `jj_new`

```typescript
jj_new({ description: string })
```

Checkpoint: creates new jj change, continues appending to same transcript file.

### `jj_describe`

```typescript
jj_describe({ description: string })
```

Update current change description. Called automatically by hook, but agent can also call explicitly.

### `query_ancestor`

```typescript
query_ancestor({
  change_id: string,
  include: ('description' | 'diff' | 'transcript')[],
  transcript_filter?: { search?: string, range?: [number, number] }
})
```

Query parent change for context. Returns requested data.

---

## Description Update Hook

Fires on `agent_end`. Generates summary from conversation:

```typescript
// hooks/jj-describe.ts
export default function(pi: HookAPI) {
  pi.on("agent_end", async (event, ctx) => {
    const summary = await generateSummary(event.messages);
    await ctx.exec("jj", ["describe", "-m", summary]);
  });
}
```

Summary format:
```
{original task}

Done:
- {completed items}

Key decisions:
- {decisions with rationale}

Left to do:
- {remaining items}

Open questions:
- {unresolved questions}
```

---

## Edge Cases

| Scenario | Handling |
|----------|----------|
| Change squashed into parent | Session remains valid if root change id survives |
| Change abandoned | Mark session non-resumable in metadata |
| Change rebased to unrelated parent | `query_ancestor` fails, agent surfaces discontinuity |
| Multiple agents on same change | Use jj workspaces (future) |

---

## Implementation Plan

### Phase 1: Minimal Viable

1. **JjSessionManager** — implement SessionManager interface, key by change id
2. **jj-describe hook** — update description on agent_end
3. **CLI wrapper** — `jj-agent start "task"` creates change and starts session

### Phase 2: Full Tools

4. **jj_new tool** — agent can checkpoint
5. **query_ancestor tool** — agent can reach back to parent context
6. **Resume support** — `jj-agent resume` loads from change id

### Phase 3: Polish

7. **Summary generation** — better prompts for description updates
8. **Context loading** — smart loading of parent description + diff
9. **TUI integration** — show jj status in agent UI

---

## Usage Examples

### Start new task

```bash
cd my-project
jj-agent start "Add rate limiting to API"
# Creates jj change, starts agent
```

### Continue after context overflow

```bash
jj-agent continue
# Loads parent description, starts fresh context
```

### Resume existing session

```bash
jj-agent resume qmwxnhgl
# Loads transcript for change id, continues
```

### Review changes

```bash
jj diff              # what changed
jj log -r @::main    # change history
jj show @            # current change with description
```

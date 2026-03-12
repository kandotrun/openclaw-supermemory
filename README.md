# openclaw-supermemory

OpenClaw plugin for full [supermemory](https://supermemory.ai)-driven memory management.

Your AI assistant remembers everything — every session, every conversation. No more context loss.

## What it does

- **Auto-Recall**: Before every response, searches supermemory for relevant context and injects it into the prompt (via `before_prompt_build` hook)
- **Auto-Capture**: After every conversation, sends ALL messages (user + assistant) to supermemory — supermemory decides what to remember
- **Compaction Hook**: Automatically saves conversation content before session compaction
- **Agent Tools**: Registers `memory_save`, `memory_recall`, `memory_forget` as native agent tools (no manual mcporter calls needed)

## Prerequisites

- [OpenClaw](https://github.com/openclaw/openclaw) (2026.3+ recommended)
- [mcporter](https://github.com/nicholasgriffintn/mcporter) CLI installed and configured with supermemory
- A [supermemory](https://supermemory.ai) account and API key

### mcporter setup

```bash
# Install mcporter
npm install -g mcporter

# Configure supermemory MCP server (~/.mcporter/mcporter.json)
cat > ~/.mcporter/mcporter.json << 'EOF'
{
  "mcpServers": {
    "supermemory": {
      "baseUrl": "https://mcp.supermemory.ai/mcp",
      "headers": {
        "Authorization": "Bearer ${SUPERMEMORY_API_KEY}"
      }
    }
  }
}
EOF

# Set your API key
export SUPERMEMORY_API_KEY="your-key-here"

# Verify it works
mcporter call supermemory.recall query="test"
```

## Install

### Option A: Copy to extensions directory

```bash
git clone https://github.com/kandotrun/openclaw-supermemory.git
cp -r openclaw-supermemory ~/.openclaw/extensions/supermemory-auto
```

### Option B: Load from custom path

Add to your `openclaw.json`:

```json
{
  "plugins": {
    "load": {
      "paths": ["/path/to/openclaw-supermemory"]
    }
  }
}
```

### Enable the plugin

```json
{
  "plugins": {
    "entries": {
      "supermemory-auto": {
        "enabled": true,
        "config": {
          "autoRecall": true,
          "autoCapture": true
        }
      }
    },
    "slots": {
      "memory": "supermemory-auto"
    }
  }
}
```

Then restart the gateway:

```bash
openclaw gateway restart
```

## Configuration

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `autoRecall` | boolean | `true` | Search supermemory and inject context before each response |
| `autoCapture` | boolean | `true` | Send ALL conversation messages (user + assistant) to supermemory |
| `recallTimeoutMs` | number | `10000` | Timeout for supermemory recall calls (ms) |
| `captureTimeoutMs` | number | `20000` | Timeout for supermemory save calls (ms) |
| `maxQueryLength` | number | `300` | Max characters from user message used as search query |
| `maxContextChars` | number | `15000` | Max characters of recalled context injected into prompt |
| `maxCaptureChars` | number | `5000` | Max characters per capture payload sent to supermemory |
| `containerTag` | string | `""` | Optional supermemory project/container tag for scoping memories |

## How it works

```
User sends message
        │
        ▼
┌─────────────────────────┐
│ before_prompt_build      │  ← Recall phase
│                         │
│ 1. Extract query        │
│    (strip metadata)     │
│ 2. mcporter call        │
│    supermemory.recall   │
│ 3. Inject context       │
│    into prompt          │
└─────────────────────────┘
        │
        ▼
   Agent responds
   (with full context)
        │
        ▼
┌─────────────────────────┐
│ agent_end                │  ← Capture phase
│                         │
│ 1. Collect ALL messages │
│    (user + assistant)   │
│ 2. Label each:          │
│    [ユーザー] / [アシスタント] │
│ 3. Strip noise          │
│    (metadata, context)  │
│ 4. Send to supermemory  │
│    (it decides what     │
│     to remember)        │
└─────────────────────────┘

Session compaction triggered
        │
        ▼
┌─────────────────────────┐
│ before_compaction        │  ← Pre-compaction save
│                         │
│ Save full conversation  │
│ to supermemory before   │
│ messages are pruned     │
└─────────────────────────┘
```

## Agent Tools

The plugin registers three tools that the agent can use directly:

| Tool | Description |
|------|-------------|
| `memory_save` | Save information to supermemory (with optional containerTag) |
| `memory_recall` | Search supermemory for relevant memories |
| `memory_forget` | Remove specific information from supermemory |

These tools use mcporter internally — the agent doesn't need to call mcporter manually.

## Capture Format

Messages sent to supermemory are clearly labeled:

```
[ユーザー] 明日福岡に行くんだけど、充電スポット調べて

[アシスタント] 福岡エリアのTesla Supercharger一覧:
- 福岡天神SC (4基)
- LECT広島SC (8基)
...
```

Supermemory receives the full exchange and decides what's worth remembering. No client-side filtering — supermemory's AI handles the curation.

## Skipped messages

The plugin skips:
- Heartbeat/system messages
- Messages shorter than 5 characters
- `NO_REPLY` / `HEARTBEAT_OK` responses
- Already-injected supermemory-context blocks
- Metadata blocks (JSON, conversation info)

## Notes

- **Latency**: Each recall adds ~2-3s (mcporter process spawn + MCP call). This is the trade-off for always having context.
- **No filtering by design**: v2 sends everything to supermemory. Supermemory's AI decides what to remember. This avoids losing important context that pattern-based filters would miss.
- **Memory slot**: Setting `plugins.slots.memory` to `supermemory-auto` replaces the default `memory-core` plugin. If you want both, don't set the memory slot.
- **containerTag**: Use this to scope memories to a specific project (e.g., `sm_project_default`).

## Changelog

### v2.0.0 (2026-03-13)

- **Full rewrite** — supermemory-driven architecture
- Migrated from `before_agent_start` (legacy) to `before_prompt_build`
- Full conversation capture: user + assistant messages, labeled with `[ユーザー]` / `[アシスタント]`
- No client-side filtering — supermemory decides what to remember
- Added `before_compaction` hook for pre-compaction saves
- Registered agent tools: `memory_save`, `memory_recall`, `memory_forget`
- Added `containerTag` support for project-scoped memories
- Expanded `maxBuffer` to 1MB for large recall results
- Better query extraction (strips context blocks + metadata noise)

### v1.0.0 (2026-03-12)

- Initial release
- Auto-recall via `before_agent_start`
- Optional auto-capture (user messages only, pattern-filtered)

## License

MIT

Copyright (c) 2026 Kan Ninomiya & 白川 玲 (AI)

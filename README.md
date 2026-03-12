# openclaw-supermemory

OpenClaw plugin that automatically recalls context from [supermemory](https://supermemory.ai) before each agent response.

No more "wait, who are you again?" — your AI assistant remembers everything, every session.

## What it does

- **Auto-Recall**: Before every agent response, searches supermemory for context relevant to the current message and injects it into the prompt
- **Auto-Capture** (opt-in): After conversations, saves important user messages to supermemory for future recall

## Prerequisites

- [OpenClaw](https://github.com/openclaw/openclaw) (2026.1+ recommended)
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
          "autoCapture": false
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
| `autoCapture` | boolean | `false` | Save important user messages to supermemory after conversations |
| `recallTimeoutMs` | number | `10000` | Timeout for supermemory recall calls (ms) |
| `maxQueryLength` | number | `300` | Max characters from user message used as search query |
| `maxContextChars` | number | `3000` | Max characters of recalled context injected into prompt |

## How it works

```
User sends message
        │
        ▼
┌─────────────────────┐
│ before_agent_start   │ ← OpenClaw lifecycle hook
│                     │
│ 1. Extract query    │
│    from user message │
│ 2. mcporter call    │
│    supermemory.recall│
│ 3. Inject context   │
│    into prompt      │
└─────────────────────┘
        │
        ▼
  Agent responds
  (with full context)
        │
        ▼
┌─────────────────────┐
│ agent_end            │ ← (if autoCapture enabled)
│                     │
│ Save important      │
│ messages to         │
│ supermemory          │
└─────────────────────┘
```

## Skipped messages

The plugin skips recall for:
- Heartbeat/system messages
- Messages shorter than 5 characters
- When supermemory returns less than 20 characters

## Notes

- **Latency**: Each recall adds ~2-3s (mcporter process spawn + MCP call). This is the trade-off for always having context.
- **Auto-Capture off by default**: If your agent already saves to supermemory manually (via mcporter tool calls in prompts), keep `autoCapture: false` to avoid duplicates.
- **Memory slot**: Setting `plugins.slots.memory` to `supermemory-auto` replaces the default `memory-core` plugin. If you want both, don't set the memory slot.

## License

MIT

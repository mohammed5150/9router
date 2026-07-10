# OpenClaw Integration

Integrate 9Router with [OpenClaw](https://docs.openclaw.ai) (formerly Clawdbot), the open-source personal AI assistant, to route its model calls through 9Router's smart routing and fallback system.

9Router works with OpenClaw over two protocols:

- **Anthropic Messages** (`api: "anthropic-messages"`) — recommended for Claude models (`cc/*`). 9Router natively serves `/v1/messages` and `/v1/messages/count_tokens`, so thinking blocks and tool calls round-trip without lossy conversion.
- **OpenAI Completions** (`api: "openai-completions"`) — for any other model (`cx/*`, `glm/*`, etc.) via `/v1/chat/completions`.

## Prerequisites

- OpenClaw installed (locally or on a server)
- 9Router running locally or a cloud endpoint configured
- API key from 9Router dashboard

## Setup

### 1. Get Your 9Router API Key

Open the 9Router dashboard, connect your provider accounts, and create an API key from the Keys page. Check available model IDs:

```bash
curl http://localhost:20128/v1/models \
  -H "Authorization: Bearer your-api-key-from-dashboard"
```

### 2. Add 9Router as a Custom Provider

Edit OpenClaw's config file (`~/.openclaw/openclaw.json`) and register 9Router under `models.providers`:

```json5
{
  "agents": {
    "defaults": {
      "model": { "primary": "9router/cc/claude-sonnet-4-5-20250929" },
      "models": {
        "9router/cc/claude-sonnet-4-5-20250929": { "alias": "Sonnet" },
        "9router/cc/claude-opus-4-5-20251101": { "alias": "Opus" }
      }
    }
  },
  "models": {
    "providers": {
      "9router": {
        // Bare host for anthropic-messages — the client appends /v1/messages
        "baseUrl": "http://localhost:20128",
        "apiKey": "your-api-key-from-dashboard",
        "api": "anthropic-messages",
        "models": [
          {
            "id": "cc/claude-sonnet-4-5-20250929",
            "name": "Claude Sonnet via 9Router",
            "reasoning": true,
            "input": ["text", "image"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 200000,
            "maxTokens": 8192
          },
          {
            "id": "cc/claude-opus-4-5-20251101",
            "name": "Claude Opus via 9Router",
            "reasoning": true,
            "input": ["text", "image"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}
```

Key points:

- **Model `id` is the full 9Router model ID** including the provider-alias prefix (`cc/...`, `cx/...`), exactly as listed by `/v1/models`. The fully qualified OpenClaw name becomes `9router/cc/...`.
- **Register models in both places.** `models.providers` defines the runtime model; `agents.defaults.models` allowlists it. Missing the second causes "model not allowed" errors.
- For non-Claude models, use a second provider entry with `"api": "openai-completions"` and `"baseUrl": "http://localhost:20128/v1"` (the `/v1` suffix is required for the OpenAI protocol).

### 3. Restart and Verify

Restart the OpenClaw gateway, then:

```bash
openclaw models set 9router/cc/claude-sonnet-4-5-20250929
```

Send a test message from your connected chat channel (Telegram, Discord, etc.).

## Running Both on Railway

You can host OpenClaw and 9Router together in one Railway project:

1. **9Router service** — deploy with the repo Dockerfile. Attach a volume at `/app/data` (matches `DATA_DIR`) so accounts and keys survive redeploys, and set `REQUIRE_API_KEY=true`, `JWT_SECRET`, `INITIAL_PASSWORD`, `API_KEY_SECRET`, and `MACHINE_ID_SALT`. Never expose an unauthenticated `/v1` publicly.
2. **OpenClaw service** — deploy the [official OpenClaw Railway template](https://docs.openclaw.ai/install/railway). Attach a volume at `/data` and set `SETUP_PASSWORD`, `OPENCLAW_STATE_DIR=/data/.openclaw`, `OPENCLAW_WORKSPACE_DIR=/data/workspace`, and `OPENCLAW_GATEWAY_TOKEN`.
3. In OpenClaw's config, set `baseUrl` to your 9Router public URL (`https://your-9router.up.railway.app`).

**Private networking (optional):** to keep traffic inside the project, set `HOSTNAME=::` on the 9Router service (Railway's private mesh is IPv6-only) and use `http://<9router-service>.railway.internal:20128` as the base URL. Keep `REQUIRE_API_KEY=true` either way — the public domain still exposes `/v1`.

## Troubleshooting

### Model Not Allowed

1. Verify the model appears in both `models.providers["9router"].models` and `agents.defaults.models`
2. Use the fully qualified name: `9router/cc/claude-sonnet-4-5-20250929`

### Authentication Errors (401)

1. Verify the API key from the 9Router dashboard
2. 9Router accepts both `Authorization: Bearer` and Anthropic-style `x-api-key` headers, so either protocol authenticates
3. If `REQUIRE_API_KEY=false`, no key is needed for local setups

### Connection Issues

1. Verify 9Router is running: `curl http://localhost:20128/health`
2. For `anthropic-messages`, use the bare host as `baseUrl` (no `/v1`); for `openai-completions`, include `/v1`
3. On Railway private networking, confirm `HOSTNAME=::` is set on the 9Router service

## Next Steps

- [Configure Claude Code](claude-code.md) with the same Anthropic-compatible endpoint
- [Other OpenAI-compatible tools](other-tools.md)
- [Cloud deployment guide](../deployment/cloud.md)

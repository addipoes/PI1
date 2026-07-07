# Multimodal Router for Claude Code CLI

## Problem
DeepSeek V4 Pro (used in Claude Code CLI) cannot process images/video/audio. When a user pastes an image into Claude Code with DeepSeek as backend, the API call fails because DeepSeek lacks vision support.

## Solution: HTTP Proxy (`mm-router.js`)
A Node.js HTTP proxy that sits between Claude Code and OpenRouter. It intercepts every API request, detects images, routes them through a vision-capable model (MiniMax M3.6), replaces images with text descriptions, and forwards the text-only request to DeepSeek.

**Location:** `C:\AI\multimodal-router\`

### Files
- `mm-router.js` — The proxy server (Node.js, zero dependencies, ~12KB)
- `start.bat` — Windows launcher
- `SETUP.md` — Full setup instructions

### Architecture
```
Claude Code CLI → localhost:9999 → OpenRouter
                    │                ├─ MiniMax M3.6 (when images detected)
                    │                └─ DeepSeek V4 Pro (text only)
                    │
                    Has images? 
                    ├─ YES → Call MiniMax for description
                    │        Replace images with text
                    │        Forward to DeepSeek
                    └─ NO → Forward directly to DeepSeek
```

### Key Design Decisions
- Uses **Anthropic Messages API format** (what Claude Code sends natively)
- Also supports OpenAI Chat Completions format
- Vision call uses OpenAI format to OpenRouter (simpler, universally supported)
- Forwards streaming (SSE) transparently
- Passes through `/v1/models` endpoint for Claude Code init
- Zero dependencies — uses only Node.js built-in `http` module and global `fetch`
- Health check at `GET /health`

### Environment Variables
| Variable | Default | Purpose |
|----------|---------|---------|
| `OPENROUTER_API_KEY` | (required) | OpenRouter API key |
| `MM_ROUTER_PORT` | `9999` | Proxy listen port |
| `MM_VISION_MODEL` | `minimax/minimax-m3` | Vision model for image description |
| `MM_MAIN_MODEL` | `deepseek/deepseek-v4-pro` | Main coding model |
| `MM_DEBUG` | `0` | Set to `1` for verbose logging |

### Claude Code Configuration
```bash
# Windows (permanent)
setx ANTHROPIC_BASE_URL http://localhost:9999/v1
setx ANTHROPIC_API_KEY any-value

# Linux/macOS
export ANTHROPIC_BASE_URL=http://localhost:9999/v1
export ANTHROPIC_API_KEY=any-value
```

### Limitations (vs pi-multimodal-proxy)
- ❌ No image description caching (re-calls vision model each time)
- ❌ No `analyze_image` tool with crop/bounding-box support
- ❌ No video/audio analysis (images only)
- ❌ No consent dialog
- ❌ Single-user/proxy (not session-aware)
- ✅ Core functionality: automatic image→text conversion is working

### Comparison: pi vs Claude Code + mm-router
| Feature | pi Agent | Claude Code + mm-router |
|---------|----------|------------------------|
| Approach | Extension (`before_agent_start` hook) | HTTP proxy (intercepts API requests) |
| Automatic | ✅ | ✅ |
| Image caching | ✅ SHA-256 hash | ❌ |
| analyze_image tool | ✅ Crop, BBox, grounding | ❌ |
| Video/Audio | ✅ Grok/MiniMax native | ❌ |
| Consent | ✅ GUI dialog | ❌ |
| Model selection for video | ✅ slash command | ❌ |

### Cost Model (via OpenRouter, as of June 2026)
- MiniMax M3.6: $0.30/M input, $1.20/M output (50% launch discount until June 7)
- DeepSeek V4 Pro: $0.435/M input, $0.87/M output
- MiniMax also offers $20/month flatrate (1.7B tokens) — only via direct MiniMax API, not OpenRouter

### Notes from the pi setup
- The `pi-multimodal-proxy` package (npm:pi-multimodal-proxy, GitHub: rfxlamia/eye-pi) is a mature v1.5.0 extension for pi agent
- pi-multimodal-proxy config expects **separate** `provider` and `modelId` fields in JSON, NOT a combined `model` string
- Common pitfall: `imagescript` and `imghash` dependencies may fail to install (npm crash), requiring manual `npm install` in the extension directory
- Both MiniMax M3 and DeepSeek V4 Pro available via OpenRouter with single API key

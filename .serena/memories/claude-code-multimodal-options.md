# Claude Code Multimodal Proxy — Optionen für später

## Status: Idee, noch nicht umgesetzt

Vier Architektur-Optionen, um Claude Code CLI (mit DeepSeek V4 Pro als Hauptmodell) multimodal zu machen:

### Option A: pi als Hintergrund-Agent
Claude Code spawnet `pi --rpc` und kommuniziert via JSONL. pi stellt pi-multimodal-proxy + pi-ocr als Tools bereit.
- ✅ Volles Caching, Video, Audio, analyze_image, OCR
- ❌ Hohe Komplexität, pi-Start-Latenz 3-5s, Headless ungetestet

### Option B: CLI-Tool-Wrapper
Schlankes Node-Script, das bei Bedarf OpenRouter (MiniMax M3) aufruft. Claude Code ruft es via bash auf.
- ✅ Extrem einfach, kein Dauerprozess
- ❌ Kein Caching, kein Video/Audio, explizit statt automatisch

### Option C: MCP-Server (empfohlen für nächsten Schritt)
Claude Code nativ unterstützt MCP. Ein MCP-Server stellt `describe_image`, `ocr_document`, `analyze_video` als Tools bereit.
- ✅ Claude Code nativ, saubere Trennung, Caching möglich
- ❌ Dauerprozess nötig, etwas Boilerplate

### Option D: mm-router.js (fertig gebaut)
HTTP-Proxy bei `C:\AI\multimodal-router\mm-router.js`, transparent zwischen Claude Code und OpenRouter.
- ✅ Fertig, funktioniert, zero-touch
- ❌ Nur Bilder, kein Caching, kein Video/Audio

### Empfohlene Evolution
1. mm-router (D) für Basis-Bildsupport → läuft
2. MCP-Server (C) für describe_image + ocr_document
3. pi im Hintergrund (A) wenn Caching & Video wichtig werden

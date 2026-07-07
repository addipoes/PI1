# pi-Konfiguration — Transfer-Guide

> Exportiert von `C:/AI/PI1` · pi v0.80.3 · 2026-07-02

---

## 1. Environment Variables

```bash
export OPENROUTER_API_KEY="sk-or-..."    # Dein Key
export PI_CODING_AGENT="1"
```

---

## 2. Global Config (`~/.pi/agent/`)

### `settings.json`

```json
{
  "lastChangelogVersion": "0.80.3",
  "defaultProvider": "deepseek",
  "defaultModel": "deepseek-v4-pro",
  "defaultThinkingLevel": "xhigh",
  "packages": [
    "npm:pi-webaio",
    "npm:@juicesharp/rpiv-btw",
    "npm:pi-ocr",
    "npm:pi-multimodal-proxy"
  ],
  "hideThinkingBlock": true,
  "theme": "light",
  "skills": [
    "~/.claude/skills",
    "~/.pi/agent/skills"
  ]
}
```

### `models.json`

```json
{
  "providers": {
    "openrouter": {
      "baseUrl": "https://openrouter.ai/api/v1",
      "api": "openai-completions",
      "apiKey": "$OPENROUTER_API_KEY",
      "models": [
        {
          "id": "deepseek/deepseek-v4-pro",
          "name": "DeepSeek V4 Pro (OR)",
          "reasoning": true,
          "input": ["text"],
          "cost": { "input": 0.435, "output": 0.87, "cacheRead": 0.0036, "cacheWrite": 0.435 },
          "contextWindow": 1000000,
          "maxTokens": 16384,
          "compat": { "thinkingFormat": "deepseek" }
        },
        {
          "id": "minimax/minimax-m3",
          "name": "MiniMax M3 (Vision)",
          "reasoning": true,
          "input": ["text", "image"],
          "cost": { "input": 0.30, "output": 1.20, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 1000000,
          "maxTokens": 16384
        },
        {
          "id": "x-ai/grok-4.3",
          "name": "Grok 4.3 (Video/Audio)",
          "reasoning": true,
          "input": ["text", "image"],
          "cost": { "input": 1.50, "output": 6.00, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 1000000,
          "maxTokens": 16384
        },
        {
          "id": "openrouter/fusion",
          "name": "Fusion (Multi-Model: Opus+GPT+Gemini)",
          "reasoning": true,
          "input": ["text"],
          "cost": { "input": 5.0, "output": 20.0, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 1000000,
          "maxTokens": 16384
        }
      ]
    }
  }
}
```

### `AGENTS.md`

```markdown
# Global Agent Instructions

## Dependency Deep-Insight: serena + opensrc

Du hast Zugriff auf **serena** (semantische Code-Analyse) und **opensrc** (Dependency-Source-Caching). Kombiniere sie, um nicht raten zu müssen.

### opensrc – Source von Abhängigkeiten holen

# Pfad zur gecachten Source erhalten (fetch bei Cache-Miss):
opensrc path <dep>              # z.B. zod, pypi:requests, crates:serde
opensrc path <owner>/<repo>     # GitHub-Repos

# Cache verwalten:
opensrc list                    # Was ist gecached?
opensrc fetch <dep>             # Nur cachen, kein Pfad

Source liegt global unter `~/.opensrc/repos/<host>/<owner>/<repo>/<version>/`.

### serena – Semantisch verstehen

Nutze serena-Tools für symbol-basierte Navigation. Setup pro Projekt:
serena init
python C:/ai/setup-serena-deps.py --deps <wichtige-deps>

### ⚠️ PFLICHT: Source analysieren VOR Implementierung
Bevor du Code schreibst, der eine externe API, ein Protokoll oder eine Library integriert → **erst die Source-Types lesen.** Keine Ausnahme.
```

---

## 3. Globale Extensions (`~/.pi/agent/extensions/`)

Diese Dateien müssen 1:1 rüberkopiert werden:

| Datei | Zweck |
|-------|-------|
| `fusion-switch.ts` | Registriert `fusion_analyze` + `deliberate` Tools. ~33KB. Kern-Feature. |
| `auto-consent.ts` | Auto-Bestätigung für Tool-Calls |
| `browser-tool.ts` | Playwright-Browser-Automation |
| `debug-vision.ts` | Vision-Debugging |
| `sprint.ts` | Sprint-Loop Tool (Lead/Worker) |
| `serena-mcp/` | Serena semantische Code-Analyse (MCP-Integration) |
| `node_modules/` + `package.json` | Dependencies für Browser-Tool (playwright) |

**Aktion:** Gesamten Ordner `~/.pi/agent/extensions/` kopieren.

---

## 4. Projekt-Extensions (MoA)

Von `C:/AI/PI1/.pi/extensions/moa-enhanced/` kopieren:

| Datei | Zweck |
|-------|-------|
| `index.ts` | MoA-Tool: GLM 5.2 + DSv4 Pro → Opus Aggregator |
| `package.json` | Package-Metadaten |
| `MOA.md` | Architektur-Doku |
| `PHASE2.md` | Phase-2-Plan (Two-Pass) |

---

## 5. Skills

| Pfad | Skill |
|------|-------|
| `~/.pi/agent/skills/dep-insight/SKILL.md` | Dependency-Analyse mit opensrc + serena |
| `~/.pi/agent/skills/sprint-lead.md` | Sprint-Loop Lead-Verhalten |
| `~/.pi/agent/skills/sprint-loop.skill.md` | Sprint-Loop Aktivierung |
| `~/.pi/agent/skills/sprint-worker.md` | Sprint-Loop Worker |
| `~/.agents/skills/handoff/SKILL.md` | Session-Handoff |
| `C:/AI/PI1/.pi/skills/moa-skill/SKILL.md` | **MoA Tool-Selection-Guide** (wann moa vs fusion vs deliberate) |

**Aktion:** Alle kopieren. `moa-skill` ins jeweilige Projekt-`.pi/skills/`.

---

## 6. npm Global Packages

```bash
npm install -g \
  @earendil-works/pi-coding-agent@0.80.3 \
  opensrc@0.7.2 \
  pi-playwright@0.1.1 \
  typescript@5.9.3 \
  pyright@1.1.410
```

Optional:
```bash
npm install -g \
  @anthropic-ai/claude-code \
  perplexity-user-mcp \
  pnpm
```

---

## 7. Quick-Setup-Checkliste

```bash
# 1. pi installieren
npm install -g @earendil-works/pi-coding-agent@0.80.3

# 2. Verzeichnisse anlegen
mkdir -p ~/.pi/agent/extensions
mkdir -p ~/.pi/agent/skills

# 3. Config-Dateien kopieren
#    → ~/.pi/agent/settings.json
#    → ~/.pi/agent/models.json
#    → ~/.pi/agent/AGENTS.md

# 4. Extensions kopieren
#    → ~/.pi/agent/extensions/* (alle .ts-Dateien + serena-mcp/)

# 5. Skills kopieren
#    → ~/.pi/agent/skills/*
#    → ~/.agents/skills/handoff/

# 6. MoA ins Projekt
mkdir -p <projekt>/.pi/extensions/moa-enhanced
mkdir -p <projekt>/.pi/skills/moa-skill
# → Dateien aus C:/AI/PI1/.pi/ rüberkopieren

# 7. API-Key setzen
export OPENROUTER_API_KEY="sk-or-..."

# 8. Testen
pi --version
pi --list-models
cd <projekt> && pi -e .pi/extensions/moa-enhanced/index.ts -p "test"
```

---

## 8. Was NICHT kopiert werden muss

- `~/.pi/agent/auth.json` — enthält API-Keys, lieber neu einrichten
- `~/.pi/agent/sessions/` — Session-Historie, projektspezifisch
- `node_modules/` — via `npm install` neu generieren
- `~/.serena/` — Serena-Cache, wird bei `serena init` neu gebaut
- `~/.opensrc/` — Dependency-Cache, bei Bedarf via `opensrc fetch` neu befüllen

---

## 9. Serena Memories

Serena-Memories sind globales Wissen, das zwischen Sessions erhalten bleibt.
Auf dem Ziel-Laptop via `serena_write_memory` mit diesen Namen + Inhalten anlegen:

### `global/serena-rules`
```markdown
# Serena — PFLICHTWERKZEUG (gilt projektübergreifend)

## Eiserne Regel

**Serena ist das PFLICHTWERKZEUG für Code-Analyse.** `bash`/`grep`/`find` sind nur
Fallback, wenn Serena nachweislich nicht verfügbar ist.

## Ablauf bei jedem Analyse-Schritt

1. **Erst Serena versuchen** — `find_symbol`, `find_referencing_symbols`, `get_symbols_overview`
2. **Bei Timeout/Fehler:** STOPP und melden — "Serena nicht verfügbar wegen [Grund]. Soll ich mit bash/grep weitermachen?"
3. **NIEMALS stillschweigend auf bash/grep ausweichen** und so tun, als sei nichts passiert.

## Grep/bash zählen NIE als Verifikation

`bash`/`grep`/`find`/`read` liefern TEXT, keine Symbol-Semantik. Sie können
niemals einen Serena-Beleg ersetzen. Wenn Serena nicht verfügbar ist, bleibt
der Analyse-Schritt **unverifiziert** — das muss explizit so benannt werden.

## Editieren

- **Code- und Test-Änderungen erfolgen über Serenas Symbol-Editing**
  (`replace_symbol_body`, `insert_after_symbol`, `insert_before_symbol`), NICHT über
  sed/python-Textersetzung.

## Tests und Verifikation

- **Tests und Verifikation IMMER gegen den committeten Stand**
- **„Verifiziert" erst schreiben, wenn Serena den Beleg geliefert hat.**
```

### `global/browser-tool`
```markdown
# Pi Browser Tool — Native Playwright Automation

## Installation
mkdir -p ~/.pi/agent/extensions && cd ~/.pi/agent/extensions
npm init -y && npm install playwright && npx playwright install chromium

Extension liegt in `~/.pi/agent/extensions/browser-tool.ts`.
Aktivierung: `/reload` in pi, dann Tool `browser` verfügbar.

## Verwendung
Tool `browser` mit Parameter `script` (Playwright-Code, erhält `page`-Objekt) und optional `url`.
```

### `global/multimodal-consent`
```markdown
At the start of every new session, remind the user to run:
/multimodal-proxy consent yes

This enables image analysis (analyze_image, pi_ocr via remote backends).
Without consent, image analysis tools return errors. The user wants this enabled always.
```

### `global/qa-self-check-workflow`
```markdown
# QA Self-Check Workflow (Playwright + Screenshot + Vision)

## Prinzip
1. playwright → Aktion im Browser
2. Screenshot → Datei
3. analyze_image → Frage an Vision-Model
4. Auswertung → grün/rot

## Wichtige Fallstricke
- Refs verfallen nach page reload → immer frischen Snapshot nehmen
- eval mit async/await scheitert → keine async functions, keine Pfeilfunktionen, kein template literal
- eval mit function() statt ()=> funktioniert zuverlässiger
- fill triggert nicht immer Vue watch() → ggf. press für einzelne Buchstaben
- --raw bei eval für cleanen Output

## Muster für häufige Checks
| Check | Eval |
|-------|------|
| Element sichtbar? | document.querySelector('[aria-label=X]') !== null |
| Text im DOM? | document.body.innerText.includes('...') |
```

### `claude-code-multimodal-options`
```markdown
# Claude Code Multimodal Proxy — Optionen für später

Vier Architektur-Optionen, um Claude Code CLI multimodal zu machen:

A: pi als Hintergrund-Agent (spawn pi --rpc)
B: CLI-Tool-Wrapper (Node-Script für OpenRouter)
C: MCP-Server (empfohlen für nächsten Schritt)
D: mm-router.js (fertig gebaut, läuft)

Empfohlene Evolution: mm-router → MCP-Server → pi im Hintergrund
```

### `multimodal-router-claude-code`
```markdown
# Multimodal Router for Claude Code CLI

HTTP Proxy (mm-router.js) zwischen Claude Code und OpenRouter.
Erkennt Bilder, routet sie durch MiniMax M3.6, ersetzt mit Textbeschreibung.

Location: C:\AI\multimodal-router\
Port: 9999
Env: OPENROUTER_API_KEY, MM_ROUTER_PORT, MM_VISION_MODEL, MM_MAIN_MODEL, MM_DEBUG

Claude Code Config:
setx ANTHROPIC_BASE_URL http://localhost:9999/v1
setx ANTHROPIC_API_KEY any-value
```

---

## 10. Bekannte Fallstricke

- **Serena auf Windows:** `PI_SERENA_PROJECT=<pfad>` setzen, sonst versucht serena `C:\` zu scannen → `WinError 5`
- **OpenRouter-Key:** Muss als `OPENROUTER_API_KEY` (nicht `OR_KEY` o.ä.) gesetzt sein
- **MoA-Extension:** Braucht `OPENROUTER_API_KEY` und Node.js ≥18 (fetch + AbortSignal.any)
- **Fusion-Switch:** `DEFAULT_MODEL = "z-ai/glm-5.2"` — sicherstellen dass GLM via OpenRouter verfügbar ist

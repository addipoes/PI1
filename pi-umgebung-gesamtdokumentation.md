# pi-Entwicklungsumgebung — Gesamtdokumentation

> **Instanz:** Windows · pi v0.80.3 · Stand 2026-07-07  
> **Erstellt für:** Architektur-Review & Optimierung (via Fable 5)  
> **Scope:** Alle selbst entwickelten Extensions, Skills, Konfigurationen. Built-in-Tools (read, bash, edit, write) sind ausgeklammert.  
> **Update 2026-07-07 (3 Wellen):** ① Gemeinsamer OpenRouter-Client + Bug-Fixes. ② fusion_analyze in moa verschmolzen, Strategien als JSON-Daten, Thinking-Level medium, Preise korrigiert. ③ Serena-Denylist (16 statt 27 Tools), GLM 5.2 als Default-Modell. Details in Abschnitt 13.

---

## 1. Architektur-Übersicht

```
┌─────────────────────────────────────────────────────────────────┐
│                        PI RUNTIME                               │
│                                                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │   Settings   │  │   Models     │  │     AGENTS.md         │  │
│  │  (json)      │  │   (json)     │  │  (Global Instructions)│  │
│  └──────┬───────┘  └──────┬───────┘  └───────────┬───────────┘  │
│         │                 │                      │              │
│         ▼                 ▼                      ▼              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   EXTENSIONS (7)                         │   │
│  │                                                          │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │   │
│  │  │fusion-switch │  │  moa-enhanced│  │sprint (CLI)   │  │   │
│  │  │23KB · Kern   │  │ 13KB · glob. │  │12KB · Subpr.  │  │   │
│  │  │deliberate    │  │  moa (synth/ │  │sprint         │  │   │
│  │  │+ Routing     │  │   analyze)   │  │               │  │   │
│  │  └──────┬───────┘  └──────┬───────┘  └───────┬───────┘  │   │
│  │         │                 │                  │           │   │
│  │  ┌──────┴───────┐  ┌──────┴───────┐  ┌───────┴───────┐  │   │
│  │  │browser-tool  │  │serena-mcp    │  │  Support      │  │   │
│  │  │Playwright    │  │16 Code-Tools │  │auto-consent   │  │   │
│  │  │browser       │  │via MCP       │  │debug-vision   │  │   │
│  │  └──────────────┘  └──────────────┘  └───────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   SKILLS (7)                              │   │
│  │  dep-insight · sprint-lead/worker/loop · handoff · moa   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │               EXTERNAL PACKAGES                           │   │
│  │  pi-webaio · pi-ocr · pi-multimodal-proxy · rpiv-btw     │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Extensions (selbst entwickelt)

### 2.1 fusion-switch.ts — Der Kern

**Pfad:** `~/.pi/agent/extensions/fusion-switch.ts`  
**Größe:** ~23 KB / ~505 Zeilen *(vor den Updates: 33 KB / 850)*  
**Registriert:** 1 Tool, 3 Commands, Input-Handler  
**API-Zugriff:** über gemeinsamen Client `lib/openrouter-client.ts` (→ 2.8) — Retry, Hard-Timeout (300s/Phase), Error-Hygiene

#### Tools

| Tool | Beschreibung | Modelle | Kosten |
|------|-------------|---------|--------|
| `deliberate` | Single-Model Multi-Role: Architekt→Kritiker→Synthesizer. Für Architektur, Reviews, Planung. | 1 Modell, 3-4 sequenzielle Rollen | ~3× |

> **Entfernt 2026-07-07:** `fusion_analyze` ist in `moa` aufgegangen (→ 2.2). `moa mode="analyze"` liefert dieselbe Konsens-/Widersprüche-/Lücken-Analyse; ein Premium-Panel ist via `models`-Parameter möglich. Das OpenRouter-Preset `openrouter/fusion` wurde aus models.json entfernt. Zuvor war bereits der tote Parameter `analysis_models` gestrichen worden (wurde nie an die API durchgereicht).

#### Commands

| Command | Funktion |
|---------|----------|
| `/deepthink on\|off\|status` | Auto-Erkennung für komplexe Prompts |
| `/deliberate [strategy]` | Erzwingt deliberate für nächsten Prompt (Strategien = JSON-Dateien, siehe unten) |
| `/moa [analyze\|synthesize]` | Erzwingt moa für nächsten Prompt *(ersetzt das frühere `/fusion`)* |

#### Auto-Erkennung (Keyword-basiert)

Der `input`-Handler prüft jeden Prompt auf Trigger-Keywords:
- **→ deliberate:** architektur, entwurf, design, code review, refactoring, migration, technische schuld
- **→ moa (mode=analyze):** vergleich, bewertung, evaluierung, trade-off/tradeoff, vor- und nachteil, strategie, entscheidung

> Fix 2026-07-07: Die Keywords `trade.off` und `vor.-.nachteil` waren als Regex-Syntax notiert, wurden aber mit `String.includes` (Literal-Vergleich) geprüft und matchten dadurch **nie**. Ersetzt durch echte Literal-Varianten (`trade-off`, `tradeoff`, `trade off`, `vor- und nachteil`, `vor-/nachteil`, `vor und nachteil`).

#### Strategien (für deliberate) — seit 2026-07-07 als JSON-Daten

**Pfad:** `~/.pi/agent/extensions/strategies/*.json` — Dateiname = Strategie-Key. Neue Strategie = neue Datei ablegen, pi neu starten, **kein Code-Change**. Format: `{ label, description, phases: [{ name, temperature, maxTokens, reasoningEffort, systemPrompt (String oder Zeilen-Array) }] }`. Defekte Dateien werden mit Log-Meldung übersprungen.

| Strategie | Rollen (Phasen) |
|-----------|-----------------|
| `architecture` | Architekt (0.6) → Kritiker (0.7) → Gegenentwurf (0.8) → Synthesizer (0.5) |
| `code_review` | Reviewer (0.5) → Security (0.4) → Performance (0.5) → Synthesizer (0.4) |
| `debate` | Pro (0.7) → Contra (0.7) → Judge (0.3) |
| `planning` | Anforderungsanalyse (0.5) → Risiko-Check (0.6) → Umsetzungsplan (0.4) |
| `creative` | Brainstormer (0.9) → Filter (0.5) → Verfeinerer (0.6) |

#### Default-Modell

**`z-ai/glm-5.2`** via OpenRouter. Begründung im Code:
> „GLM 5.2 — exzellente Reasoning-/Code-Qualität mit xhigh Thinking (SWE-bench Pro 62.1, Terminal-Bench 81.0). Vorher deepseek-v4-pro — hatte in Praxis wiederholt falsche/abbruchende Antworten."

#### Reasoning-Effort

- DeepSeek-Modelle: `thinkingFormat: "deepseek"` mit `thinking: { type: "enabled" }`
- GLM 5.2: `reasoning_effort: "xhigh"` (Z.AI API)
- Alle anderen: `reasoning_effort` im Body (OpenRouter-Konvention)

---

### 2.2 moa-enhanced/index.ts — Mixture-of-Agents

**Pfad:** `~/.pi/agent/extensions/moa-enhanced/index.ts` *(seit 2026-07-07 global; vorher projektlokal `C:/AI/PI1/.pi/extensions/` — Umzug nötig, damit das globale Keyword-Routing auf `moa` zeigen kann; löst nebenbei den fragilen Pfad-Import)*  
**Größe:** ~13 KB / ~321 Zeilen  
**Registriert:** 1 Tool  
**API-Zugriff:** über gemeinsamen Client `../lib/openrouter-client.ts` (→ 2.8)

#### Tool

| Tool | Beschreibung | Referenzen (Default) | Aggregator | Kosten |
|------|-------------|-----------|-----------|--------|
| `moa` | Referenzen parallel → Opus als Aggregator. `mode=synthesize`: beste Antwort. `mode=analyze`: Meta-Analyse (Konsens/Widersprüche/Lücken — ersetzt fusion_analyze). | `z-ai/glm-5.2`, `deepseek/deepseek-v4-pro` (via `models` austauschbar) | `anthropic/claude-opus-4.8` | ~2-3× |

#### Parameter

| Parameter | Typ | Default | Beschreibung |
|-----------|-----|---------|-------------|
| `prompt` | String (max 10k) | Pflicht | Die Frage |
| `refine` | Boolean | `false` *(seit 2026-07-07, vorher `true`)* | `true` = Paper-Modus (2 Runden Peer-Refinement, ~50% teurer), `false` = Fast-Modus (1 Runde) |
| `models` | String[] (max 4) | GLM 5.2 + DSv4 Pro | *(neu 2026-07-07)* Referenz-Modelle überschreiben — z.B. Frontier-Panel Opus+GPT+Gemini (früheres „Fusion"-Setup, deutlich teurer) |
| `mode` | `"synthesize"` \| `"analyze"` | `synthesize` | *(neu 2026-07-07)* synthesize = Opus schreibt beste Antwort; analyze = strukturierte Meta-Analyse |

**Referenz-Calls:** maxTokens 2500 *(vorher 1500 — abgeschnittene Referenzen verschlechterten die Synthese)*, reasoningEffort `high` (wirkt nur bei GLM/DeepSeek).

#### Sicherheits-Features

| Schutz | Implementierung |
|--------|----------------|
| Prompt Injection | Referenz-Outputs in `<reference_outputs>` XML-Tags + `untrusted data`-Warnung |
| Cost-DoS | Prompt ≤ 10.000 Zeichen |
| API-Retry | im gemeinsamen Client (→ 2.8): 3 Versuche, exponentielles Backoff (1s/2s/4s), bei 429/502/503/504 |
| Timeout | im gemeinsamen Client: 120s Hard-Timeout pro Versuch via `AbortSignal.timeout()` |
| Error Leakage | im gemeinsamen Client: Details via `console.error`, nur generische Meldung nach außen |
| API-Key | via pi Model-Registry (`resolveApiKey`), Fallback `OPENROUTER_API_KEY` *(vorher nur env-Variable)* |

#### Pipeline

```
User Prompt
    │
    ├──► GLM 5.2 ($2.86/1M out)  ──parallel──  DeepSeek V4 Pro ($0.87/1M out)
    │         │                     (oder models-Override, max 4)
    │         ▼ Referenz-Antworten                 │
    │         │                                    │
    │    ┌────┴────┐                               │
    │    │ refine? │──true──► Runde 2 (Peer-Refinement)
    │    └────┬────┘                               │
    │         ▼                                    │
    │    Claude Opus 4.8 ($25/1M out) ◄───────────┘
    │    (Aggregator — mode=synthesize: schreibt beste Antwort;
    │     mode=analyze: Konsens/Widersprüche/Lücken)
    │         │
    ▼    Finale Antwort
```
*Preise: OpenRouter, verifiziert 2026-07-07. Die früheren Angaben ($5.80 GLM, $30 Opus) waren falsch.*

---

### 2.3 sprint.ts — Iterativer Sprint-Loop

**Pfad:** `~/.pi/agent/extensions/sprint.ts`  
**Größe:** ~12 KB / ~300 Zeilen  
**Registriert:** 1 Tool  
**Abhängigkeit:** Externer CLI-Subprocess (`pi-sprint` aus `C:/ai/sprint-loop/`)

#### Tool

| Tool | Beschreibung |
|------|-------------|
| `sprint` | Lead-LLM plant Tasks, Worker-LLM führt aus, Lead reviewt. Iterativ bis Done oder max Iterationen. |

#### Parameter

| Parameter | Typ | Default |
|-----------|-----|---------|
| `goal` | String | Pflicht. Mit `Deliverable: ...` Klausel |
| `tools` | String[] | `["read"]` |
| `max_iterations` | Number | `2` |
| `worker_skill` | String | `sprint-worker` |
| `lead_skill` | String | `sprint-lead` |

#### Architektur (Subprocess-Modell)

```
pi-Extension (sprint.ts)
    │
    │ spawn
    ▼
pi-sprint CLI (Node.js Subprocess)
    │
    │ JSON-Lines stdout
    ▼
Extension parst Events → Live-Updates via onUpdate → Chat
```

**CLI-Auflösung (4-stufig):**
1. `PI_SPRINT_CLI` env var
2. `pi-sprint` binary in PATH
3. Relative Pfade zur Extension
4. Hardcoded Fallback: `C:/ai/sprint-loop/src/cli/index.ts`

#### Abhängiges Paket

**Pfad:** `C:/ai/sprint-loop/`  
**Package:** `@user/sprint-loop`  
**Peer-Dependency:** `@user/sprint-loop-contracts`  
**Adapter:** Core (zustandslos), Chat, CLI, PIcowork

---

### 2.4 browser-tool.ts — Playwright-Automation

**Pfad:** `~/.pi/agent/extensions/browser-tool.ts`  
**Größe:** ~5 KB / ~160 Zeilen  
**Registriert:** 1 Tool  
**Abhängigkeit:** `playwright` (npm, global installiert via `pi-playwright`)

#### Tool

| Tool | Beschreibung |
|------|-------------|
| `browser` | Führt Playwright-Skripte im Chromium-Browser aus. Erhält `page`-Objekt. |

#### Parameter

| Parameter | Typ | Beschreibung |
|-----------|-----|-------------|
| `script` | String | Playwright-JavaScript (async, CommonJS) |
| `url` | String? | Start-URL vor Skript-Ausführung |

---

### 2.5 serena-mcp/ — Semantische Code-Analyse

**Pfad:** `~/.pi/agent/extensions/serena-mcp/`  
**Typ:** MCP-Integration (Model Context Protocol)  
**Registriert:** 16 von 27 Tools *(seit 2026-07-07 — Denylist filtert Duplikate von pi-Built-ins und Selten-Tools)*

#### Architektur

```
pi Extension (index.ts)
    │
    │ spawn
    ▼
uvx serena-mcp-server (Python)
    │
    │ JSON-RPC 2.0 via stdio
    ▼
16 registrierte Tools (Serenas Alleinstellungsmerkmal):
  find_symbol, find_referencing_symbols, get_symbols_overview,
  find_declaration, find_implementations, rename_symbol,
  replace_symbol_body, insert_after_symbol, insert_before_symbol,
  search_for_pattern, get_diagnostics_for_file,
  read_memory, write_memory, list_memories, activate_project,
  (+ initial_instructions)

11 per Denylist übersprungen:
  read_file, list_dir, find_file, execute_shell_command,   ← duplizieren Built-ins
  replace_content,                                          ← Symbol-Edits sind der Mehrwert
  delete_memory, rename_memory, edit_memory,                ← selten interaktiv
  get_current_config, onboarding, safe_delete_symbol        ← Einmal-/Selten-Tools
```

> Weniger Tool-Definitionen im Kontext = sauberere Tool-Wahl des Agenten (weniger Griffe zum falschen Werkzeug) + weniger Input-Token pro Turn. Die „Eiserne Regel Serena-Pflicht" (Memory `global/serena-rules`) betrifft die semantischen Tools — die bleiben vollständig.

#### Konfiguration

- **Server:** `serena-mcp` via `uvx` (Python-Paket)
- **Projekt:** Auto-detection via `PI_SERENA_PROJECT` env oder `--project` Flag
- **Fallback bei Windows:** `--project` statt `--project-from-cwd` + Root-Guard gegen `C:\`-Scan
- **`PI_SERENA_ALL=1`:** Denylist deaktivieren, alle 27 Tools registrieren (z.B. für Onboarding/Memory-Aufräumen)
- **`PI_SERENA_DISABLE=1`:** serena komplett stilllegen

---

### 2.6 auto-consent.ts — Multimodal-Consent

**Pfad:** `~/.pi/agent/extensions/auto-consent.ts`  
**Größe:** ~1.4 KB / ~50 Zeilen  
**Registriert:** Kein Tool — Event-Handler

#### Funktion

Liest `~/.pi/agent/multimodal-proxy.json` aus und schreibt bei `session_start` automatisch einen `vision-proxy-consent`-Eintrag in die Session. Verhindert den manuellen `/multimodal-proxy consent yes`-Befehl.

---

### 2.7 debug-vision.ts — Debug-Logging

**Pfad:** `~/.pi/agent/extensions/debug-vision.ts`  
**Größe:** ~1.6 KB / ~50 Zeilen  
**Registriert:** Kein Tool — Event-Handler

#### Funktion

Loggt `before_agent_start`- und `input`-Events nach `~/.pi/agent/debug-vision.log`. Erfasst: Modell, Bilder, Prompt-Länge, Multimodal-Proxy-Konfiguration.

---

### 2.8 lib/openrouter-client.ts — Gemeinsamer API-Client *(neu 2026-07-07)*

**Pfad:** `~/.pi/agent/extensions/lib/openrouter-client.ts`  
**Größe:** ~5.6 KB / ~154 Zeilen  
**Registriert:** Kein Tool — reine Bibliothek (liegt bewusst in `lib/` ohne `index.ts`, damit pi's Extension-Auto-Discovery sie nicht als Extension lädt)

#### Funktion

Vereint die vormals **duplizierten** `callModel`-Implementierungen aus fusion-switch und moa-enhanced (~140 Zeilen Duplikat entfernt). Konsumenten:

- `fusion-switch.ts` — Import `./lib/openrouter-client` (Timeout 300s/Phase wegen langer Reasoning-Läufe)
- `moa-enhanced/index.ts` — Import `../lib/openrouter-client` *(seit dem Umzug nach global; der frühere laufwerksübergreifende Pfad-Import ist Geschichte)*

#### API

| Export | Signatur | Verhalten |
|--------|----------|-----------|
| `callModel` | `(apiKey, model, messages, options?) → Promise<CallModelResult>` | Wirft nie; Fehler in `result.error` |
| `resolveApiKey` | `(ctx?) → Promise<string \| undefined>` | pi Model-Registry, Fallback `OPENROUTER_API_KEY` |

`CallModelOptions`: `temperature`, `maxTokens`, `reasoningEffort` (nur deepseek/glm/z-ai), `signal`, `timeoutMs` (Default 120s, **pro Versuch** — *am 2026-07-07 von 60s erhöht: Funktionstest zeigte GLM-5.2-Latenzen >60s bei ~50% der Calls*), `maxRetries` (Default 3).

#### Eigenschaften

- Retry mit exponentiellem Backoff (1s/2s/4s) bei HTTP 429/502/503/504 und Netzwerkfehlern
- Hard-Timeout pro Versuch via `AbortSignal.timeout` (nicht über alle Versuche hinweg)
- Kein Retry bei Timeout oder externem Abort
- Error-Hygiene: rohe API-Antworten nur ins Log (`[openrouter]`-Prefix), generische Meldung nach außen

---

## 3. Konfiguration

### 3.1 settings.json

**Pfad:** `~/.pi/agent/settings.json`

```json
{
  "lastChangelogVersion": "0.80.3",
  "defaultProvider": "openrouter",
  "defaultModel": "z-ai/glm-5.2",
  "defaultThinkingLevel": "medium",
  "packages": [
    "npm:pi-webaio",
    "npm:@juicesharp/rpiv-btw",
    "npm:pi-ocr",
    "npm:pi-multimodal-proxy"
  ],
  "hideThinkingBlock": true,
  "theme": "light",
  "skills": ["~/.claude/skills", "~/.pi/agent/skills"]
}
```

**Hinweise:**
- Default-Modell: **GLM 5.2** *(seit 2026-07-07, vorher DeepSeek V4 Pro — Begründung: pi ist ein Coding-Agent; DeepSeeks dokumentierte Ausfälle (falsche/abbruchende Antworten) sind im Agent-Loop teurer als die ~3× Preisdifferenz beim Output. Absolut: wenige $/Monat.)*
- **Kompromiss-Regel:** Für lange Lese-/Analyse-Sessions **ohne Schreibzugriffe** bewusst auf DeepSeek V4 Pro (OR) wechseln — dessen Cache-Read ($0.0036/1M vs. ~$0.18 bei GLM) ist bei input-lastigen Sessions mit großem, wiederholt gesendetem Kontext konkurrenzlos.
- Thinking: `medium` *(seit 2026-07-07, vorher `xhigh` — Kosten-Fix: xhigh traf jeden trivialen Prompt. Hohe Reasoning-Tiefe holen sich die Spezial-Tools selbst: deliberate-Phasen high/xhigh, moa-Referenzen high. Für einzelne schwere Prompts manuell hochschalten.)*
- 4 externe pi-Packages aktiv
- Skills aus zwei Quellen (Claude-Code-kompatibel + pi-nativ)

### 3.2 models.json

**Pfad:** `~/.pi/agent/models.json`

Vier Custom-Modelle via OpenRouter registriert *(2026-07-07: `openrouter/fusion` entfernt, `z-ai/glm-5.2` neu — jetzt Default-Modell)*:

| ID | Name | Reasoning | Input | Kosten/1M Input | Kosten/1M Output |
|----|------|:---:|------|----:|---:|
| `z-ai/glm-5.2` | GLM 5.2 (OR) — **Default** | ✅ | Text | $0.91 | $2.86 |
| `deepseek/deepseek-v4-pro` | DeepSeek V4 Pro (OR) | ✅ | Text | $0.435 | $0.87 |
| `minimax/minimax-m3` | MiniMax M3 (Vision) | ✅ | Text, Image | $0.30 | $1.20 |
| `x-ai/grok-4.3` | Grok 4.3 (Video/Audio) | ✅ | Text, Image | $1.50 | $6.00 |

### 3.3 multimodal-proxy.json

**Pfad:** `~/.pi/agent/multimodal-proxy.json`

```json
{
  "mode": "fallback",
  "provider": "openrouter",
  "modelId": "minimax/minimax-m3",
  "videoProvider": "openrouter",
  "videoModelId": "minimax/minimax-m3",
  "includeContext": true,
  "tool": "on",
  "maxImagesPerCall": 10,
  "maxBatch": 4,
  "cacheSize": 50,
  "maxImageBytes": 10485760
}
```

### 3.4 AGENTS.md

**Pfad:** `~/.pi/agent/AGENTS.md`  
**Inhalt:** Serena/opensrc-Nutzungsregeln, „Source analysieren VOR Implementierung"-Pflicht, WSL2-Workflow

### 3.5 Environment

| Variable | Wert |
|----------|------|
| `OPENROUTER_API_KEY` | sk-or-... (73 Zeichen) |
| `PI_CODING_AGENT` | 1 |

---

## 4. Skills

| Name | Pfad | Zweck |
|------|------|-------|
| `dep-insight` | `~/.pi/agent/skills/dep-insight/SKILL.md` | Dependency-Analyse mit opensrc + serena |
| `sprint-lead` | `~/.pi/agent/skills/sprint-lead.md` | Lead-Verhalten im Sprint-Loop (JSON-Output) |
| `sprint-worker` | `~/.pi/agent/skills/sprint-worker.md` | Worker-Verhalten im Sprint-Loop |
| `sprint-loop` | `~/.pi/agent/skills/sprint-loop.skill.md` | Sprint-Loop-Aktivierung |
| `handoff` | `~/.agents/skills/handoff/SKILL.md` | Session-Handoff zwischen Fenstern/Agents |
| `moa-skill` | `C:/AI/PI1/.pi/skills/moa-skill/SKILL.md` | Entscheidungslogik: moa (synthesize/analyze) vs deliberate *(2026-07-07 auf Zwei-Tool-Welt aktualisiert)* |

---

## 5. Memories (Serena)

| Memory | Scope | Inhalt |
|--------|-------|--------|
| `global/serena-rules` | Global | Eiserne Regel: Serena ist Pflichtwerkzeug, bash/grep nur Fallback |
| `global/browser-tool` | Global | Playwright-Installation & Verwendung |
| `global/multimodal-consent` | Global | Auto-Consent für Image-Analyse |
| `global/qa-self-check-workflow` | Global | QA-Workflow (Playwright → Screenshot → Vision) |
| `claude-code-multimodal-options` | Projekt | Architektur-Optionen für Claude Code multimodal |
| `multimodal-router-claude-code` | Projekt | mm-router.js — HTTP-Proxy-Dokumentation |

---

## 6. Tool-Entscheidungsmatrix (Agent-Sicht)

```
Komplexität der Aufgabe?
│
├── Trivial (2+2, einfache Frage)
│   └── Direkt antworten (kein Tool)
│
├── Mittel (braucht Struktur, Planung, Review)
│   ├── Single-Model reicht → deliberate
│   └── Iterativ mit Review → sprint
│
└── Hoch (braucht mehrere Perspektiven) → moa
    ├── Synthese (beste Antwort bauen) → mode=synthesize (Default)
    │   ├── Schnell → refine=false (~2×, Default)
    │   └── Max-Qualität → refine=true (~3×)
    │
    └── Analyse (Konsens/Widersprüche/Lücken) → mode=analyze (~2×)
        (Premium: models-Override mit Frontier-Panel → maximale Diversität)
```

---

## 7. Modell-Hierarchie

### Nach Zweck

*Preise verifiziert 2026-07-07 (OpenRouter-Listenpreise; Marketplace-Routen schwanken):*

| Zweck | Modell | Provider | Kosten/1M (In / Out) |
|-------|--------|----------|----------|
| **Default (Agent, Coding)** | GLM 5.2 *(seit 2026-07-07, vorher DeepSeek)* | OpenRouter | $0.91 / $2.86 |
| **Lange Lese-/Analyse-Sessions (kein Schreiben)** | DeepSeek V4 Pro — manuell wählen (Cache-Read $0.0036/1M) | OpenRouter | $0.435 / $0.87 |
| **Deliberation (Architektur, Review)** | GLM 5.2 | OpenRouter | $0.91 / $2.86 |
| **MoA-Referenz 1 (Coding)** | GLM 5.2 | OpenRouter | $0.91 / $2.86 |
| **MoA-Referenz 2 (Reasoning)** | DeepSeek V4 Pro | OpenRouter | $0.435 / $0.87 |
| **MoA-Aggregator (Schreibstil)** | Claude Opus 4.8 | OpenRouter | $5.00 / $25.00 |
| **MoA Premium-Panel (optional)** | via `models`-Param (z.B. Opus+GPT+Gemini) | OpenRouter | modellabhängig |
| **Vision (Bilder)** | MiniMax M3 | OpenRouter | $0.30 / $1.20 |
| **Vision (Video/Audio)** | Grok 4.3 | OpenRouter | $1.50 / $6.00 |
| **fusion_analyze (deprecated)** | — | — | Entfernt 2026-07-07, in moa aufgegangen |
| **Disagreement-Check (deprecated)** | — | — | Entfernt in MoA v2 |

### Nach Kosten (Output, pro 1M Tokens)

```
$0.87  DeepSeek V4 Pro        ← Günstigster Frontier-Code
$1.20  MiniMax M3 (Vision)
$2.86  GLM 5.2                 ← Sweet Spot: Leistung/Preis
$6.00  Grok 4.3 (Video)
$25.00 Claude Opus 4.8         ← Bester Schreibstil
```

---

## 8. Externe Abhängigkeiten

### 8.1 npm Global

| Paket | Version | Zweck |
|-------|---------|-------|
| `@earendil-works/pi-coding-agent` | 0.80.3 | pi selbst |
| `opensrc` | 0.7.2 | Dependency-Source-Caching |
| `pi-playwright` | 0.1.1 | Playwright-Tool für pi |
| `typescript` | 5.9.3 | TS-Kompilierung |
| `pyright` | 1.1.410 | Python-Typ-Prüfung |
| `@anthropic-ai/claude-code` | 2.1.183 | Claude Code CLI |
| `perplexity-user-mcp` | 0.8.39 | Perplexity MCP-Server |
| `pnpm` | 11.5.2 | Paketmanager |

### 8.2 pi-Packages (via settings.json)

| Package | Zweck |
|---------|-------|
| `npm:pi-webaio` | Web-Fetching (aio-webfetch, aio-websearch, aio-webpull) |
| `npm:@juicesharp/rpiv-btw` | RPi-BTW Integration |
| `npm:pi-ocr` | OCR (MinerU, Tesseract, etc.) |
| `npm:pi-multimodal-proxy` | Automatische Bild→Text-Konvertierung |

---

## 9. Prozess-Flows

### 9.1 Session-Start

```
pi startet
    │
    ├── Extensions laden (auto-discovery)
    │   ├── ~/.pi/agent/extensions/*.ts (global)
    │   └── .pi/extensions/*.ts (projekt)
    │
    ├── session_start-Event
    │   ├── auto-consent: vision-proxy-consent schreiben
    │   ├── serena-mcp: MCP-Server spawnen, 16 Tools registrieren (11 per Denylist gefiltert)
    │   ├── debug-vision: Log-Eintrag
    │   ├── fusion-switch: Deep-Think-State initialisieren
    │   └── moa-enhanced: "MoA geladen"-Notification
    │
    ├── resources_discover
    │   └── Skills + AGENTS.md laden
    │
    └── Bereit für User-Input
```

### 9.2 Prompt-Verarbeitung

```
User-Input
    │
    ├── input-Event
    │   ├── fusion-switch prüft:
    │   │   ├── /command? → Bypass
    │   │   ├── nextTool gesetzt? → "MANDATORY: call moa/deliberate"
    │   │   └── Keyword-Match? → Auto-Routing
    │   │
    │   └── debug-vision: Log-Entry
    │
    ├── Skill-Expansion (/skill:name → SKILL.md laden)
    │
    ├── before_agent_start
    │   └── debug-vision: Modell/Bilder/Prompt loggen
    │
    └── Agent-Loop
        ├── LLM-Call (mit Tools)
        │   ├── Entscheidet: welches Tool?
        │   │   ├── moa → MoA-Pipeline (Referenzen → Opus; synthesize/analyze)
        │   │   ├── deliberate → Rollen-Loop (single model, JSON-Strategien)
        │   │   ├── sprint → Subprocess (Lead→Worker→Review→Iterate)
        │   │   ├── browser → Playwright
        │   │   └── serena_* → MCP-Server
        │   │
        │   └── Tool-Ergebnis → nächster Turn oder Antwort
        │
        └── Finale Antwort
```

---

## 10. Optimierungspotenzial

> **Status-Update 2026-07-07:** Welle 1: gemeinsamer OpenRouter-Client (→ 2.8), `refine`-Default `false`, tote Keywords und toter `analysis_models`-Parameter gefixt. Welle 2: fusion_analyze→moa, Strategien als JSON, Thinking medium, Preise verifiziert. Details in Abschnitt 13.

| Bereich | Beobachtung | Vorschlag | Status |
|--------|------------|-----------|--------|
| **callModel-Duplikat** | OpenRouter-Client 2× implementiert, ungleich robust | Gemeinsame Lib extrahieren | ✅ 2026-07-07 (→ 2.8) |
| **MoA refine-Default** | `refine=true` verdoppelte Referenz-Calls bei jedem Aufruf | Default `false` | ✅ 2026-07-07 |
| **fusion-switch** | 33 KB Monolith. Tools, Commands, Routing in einer Datei | Strategien-Prompts als JSON auslagern | ✅ 2026-07-07 (jetzt 23 KB) |
| **deliberate-Strategien** | 5 Strategien hartkodiert | Strategien als JSON-Dateien → user-erweiterbar | ✅ 2026-07-07 (`strategies/*.json`) |
| **MoA-Modelle** | DeepSeek V4 Pro + GLM 5.2 fest verdrahtet | Austauschbar ohne Code-Change | ✅ 2026-07-07 (`models`-Parameter) |
| **sprint-Subprocess** | CLI-Auflösung 4-stufig, fehleranfällig | In-Process-Library-Mode als Alternative (für schnelle Tasks) | offen |
| **serena-mcp** | 27 Tools registriert, viele selten genutzt | Statische Denylist statt Lazy-Loading (pi kann nicht dynamisch nachladen): 16 Kern-Tools aktiv, `PI_SERENA_ALL=1` als Escape-Hatch | ✅ 2026-07-07 |
| **auto-consent** | Konsumiert `session_start`, kein Command | `/consent off` für Debug-Sessions | offen |
| **debug-vision** | Immer aktiv, schreibt Logs | `/debug-vision on\|off` Command, nur bei Bedarf | offen |
| **Models** | 4 Custom-Modelle, aber `openrouter/fusion` kaum genutzt | `fusion_analyze` in `moa` aufgehen lassen, Preset entfernen | ✅ 2026-07-07 |
| **Thinking-Level** | Global `xhigh` — hohe Kosten bei einfachen Tasks | Global `medium`; deliberate/moa setzen high/xhigh pro Call selbst | ✅ 2026-07-07 (statisch; echtes Auto-Scaling pro Prompt bräuchte pi-Runtime-API) |
| **Skills-Duplikation** | `~/.claude/skills` + `~/.pi/agent/skills` beide geladen | Zusammenführen, Claude-Skills als pi-Skills symlinken | offen |

---

## 11. Bekannte Probleme & Grenzen

1. **Serena auf Windows:** `C:\`-Scan führt zu `WinError 5`. Workaround: `PI_SERENA_PROJECT` setzen oder aus Projektverzeichnis starten.
2. **DeepSeek V4 Pro:** Hatte in Praxis falsche/abbruchende Antworten in Kritiker-Rollen. Wurde durch GLM 5.2 ersetzt.
3. **MoA Cascade:** Haiku-Disagreement-Check wurde entfernt. Jetzt deterministisch (`refine`-Parameter).
4. **fusion-switch Keywords:** Nur deutsche Keywords. Englische Prompts triggern nicht. *(Update 2026-07-07: Die nie matchenden Regex-Literale `trade.off`/`vor.-.nachteil` sind gefixt — Deutsch-only bleibt.)*
5. **sprint-Subprocess:** JSON-Lines-Parsing fragil — serena-mcp-Lärm auf stderr kann JSON-Parser stören (wird gefiltert, aber nicht robust).
6. **Kein Modell-Fallback:** Schlägt ein Modell fehl, gibt's keinen automatischen Ersatz. MoA fängt es pro Referenz. *(Update 2026-07-07: fusion_analyze und deliberate haben jetzt via gemeinsamem Client Retry bei transienten Fehlern (429/5xx) und Hard-Timeout; deliberate macht bei Phasen-Fehlern ab Phase 2 mit Teilergebnissen weiter. Ein Ersatz-**Modell** bei hartem Ausfall fehlt weiterhin.)*
7. ~~**Lib-Import per relativem Pfad**~~ — *behoben 2026-07-07:* moa-enhanced liegt jetzt global neben der Lib (`~/.pi/agent/extensions/moa-enhanced/`), Import ist ein normales `../lib/openrouter-client`.
8. **Kein echtes Thinking-Auto-Scaling:** `defaultThinkingLevel` ist statisch `medium`; die Spezial-Tools setzen high/xhigh pro API-Call. Dynamisches Hochschalten pro Prompt (z.B. bei Trigger-Keywords) bräuchte eine pi-Runtime-API zum Setzen des Thinking-Levels aus Extensions — derzeit nicht verfügbar/bekannt.
9. **GLM-5.2-Latenz auf OpenRouter schwankt stark:** Funktionstest 2026-07-07 maß >60s bei ~50% der Calls (Marketplace-Routing). Client-Timeout deshalb auf 120s erhöht. Falls das Default-Modell im Alltag spürbar hängt: DeepSeek als Default zurück, GLM nur für moa/deliberate.

---

## 12. npm Global Packages (vollständig)

```
@anthropic-ai/claude-code@2.1.183
@continuedev/cli@1.5.43
@earendil-works/pi-coding-agent@0.80.3
@shivarajbakale/claudeport@1.0.8
npm@11.13.0
opensrc@0.7.2
perplexity-user-mcp@0.8.39
pi-playwright@0.1.1
pnpm@11.5.2
pyright@1.1.410
typescript@5.9.3
```

---

## 13. Changelog

### 2026-07-07 (Nachtrag) — Funktionstest + Timeout 60s→120s

Funktionstest in laufender pi-Session (via `pi -p`, Print-Mode):

| Test | Ergebnis |
|------|----------|
| Session-Start: 16 Serena-Tools (11 Denylist), MoA geladen, 5 Strategien | ✅ |
| moa mode=synthesize (GLM+DSv4 → Opus) | ✅ |
| moa mode=analyze (Meta-Analyse mit Konsens/Widersprüche/Lücken) | ✅ |
| moa `models`-Override | ✅ (Verdacht „Override ignoriert" widerlegt: der Lauf mit Override lief ohne GLM-Call durch; die GLM-Timeouts stammten aus Läufen ohne Override) |
| serena_find_symbol | ✅ |
| deliberate | ⚠️ nicht bewertbar: 3-4 sequenzielle Reasoning-Calls sprengen das Print-Mode-/Bash-Timeout — **in interaktiver TUI-Session nachtesten** |
| /moa-Command | ⏭️ nur interaktiv testbar |

**Konsequenz:** `DEFAULT_TIMEOUT_MS` im gemeinsamen Client von 60s auf **120s** erhöht — GLM 5.2 brauchte via OpenRouter-Marketplace bei ~50% der Calls >60s; der knappe Timeout kostete moa regelmäßig die GLM-Perspektive. (deliberate nutzt weiterhin 300s/Phase.)

**Bekanntes Risiko dokumentiert (→ Problem 9):** GLM-5.2-Latenz auf OpenRouter schwankt stark; wenn das Default-Modell selbst hängt, hilft der Client-Timeout der Extensions nicht (pi's eigener Call-Pfad). Beobachten — falls Alltag leidet, DeepSeek als Default zurück und GLM nur als moa-/deliberate-Modell.

---

### 2026-07-07 (Welle 3) — Serena-Denylist + GLM 5.2 als Default

| Datei | Änderung |
|-------|----------|
| `~/.pi/agent/extensions/serena-mcp/index.ts` | Tool-Denylist (11 Tools gefiltert) + `PI_SERENA_ALL=1` Escape-Hatch |
| `~/.pi/agent/models.json` | `z-ai/glm-5.2` als Custom-Model registriert ($0.91/$2.86, cacheRead $0.18) |
| `~/.pi/agent/settings.json` | `defaultProvider: openrouter`, `defaultModel: z-ai/glm-5.2` (vorher deepseek/deepseek-v4-pro) |
| `~/.pi/agent/extensions/fusion-switch.ts` | Kommentar zum Default-Modell aktualisiert |

**1. Serena-Denylist (statt Lazy-Loading):**

- Echtes Lazy-Loading ist mit pi nicht sinnvoll machbar (der Agent kann nicht aufrufen, was nicht registriert ist; dynamisches Nachladen mid-session gibt es nicht). Stattdessen statische Denylist in der Registrierungsschleife.
- **Gefiltert (11):** `read_file`, `list_dir`, `find_file`, `execute_shell_command`, `replace_content` (duplizieren pi-Built-ins bzw. edit), `delete_memory`, `rename_memory`, `edit_memory` (selten interaktiv), `get_current_config`, `onboarding` (Einmal-Tools), `safe_delete_symbol` (selten; Löschen via edit).
- **Registriert bleiben (16):** alle semantischen Symbol-Tools, `search_for_pattern`, `get_diagnostics_for_file`, Memory-Kern (`read/write/list_memories`), `activate_project`, `initial_instructions`.
- Motivation: weniger Kontext-Ballast pro Turn und v.a. sauberere Tool-Wahl (Serena-Duplikate konkurrierten mit Built-ins). Die „Eiserne Regel Serena-Pflicht" bleibt unberührt — sie betrifft die semantischen Tools.
- Escape-Hatch: `PI_SERENA_ALL=1` registriert alle 27 (z.B. für Onboarding oder Memory-Aufräumen).

**2. GLM 5.2 als Default-Modell:**

- Begründung: pi ist ein Coding-Agent — das Default-Modell macht Tool-Calls und Edits. DeepSeeks dokumentierte Zuverlässigkeitsprobleme (falsche/abbruchende Antworten; deshalb schon aus deliberate und Sprint-Lead verbannt) kosten im Agent-Loop menschliche Zeit; die ~3× Output-Preisdifferenz ist absolut klein (wenige $/Monat).
- **Kompromiss-Regel (bewusste Entscheidung):** Für lange Lese-/Analyse-Sessions **ohne Schreibzugriffe** manuell auf DeepSeek V4 Pro (OR) wechseln — Cache-Read $0.0036/1M (GLM ~$0.18) macht DeepSeek bei input-lastigen Sessions mit großem, wiederholt gesendetem Kontext konkurrenzlos billig.
- GLM 5.2 musste dafür neu in models.json registriert werden (war bisher nur direkt aus den Extensions aufgerufen). DeepSeek bleibt registriert (MoA-Referenz + manueller Lese-Modus).

**Verifikation:** `tsc --noEmit` — keine neuen Fehler (2 vorbestehende `process`-Meldungen in serena-mcp stammen aus der Umgebung ohne @types/node, Zeilen existierten vor der Änderung); models.json/settings.json JSON-validiert, Default-Auflösung geprüft. Funktionaler Test (Session-Start: „Registered 16 tools", GLM antwortet als Default) steht aus.

---

### 2026-07-07 (Welle 2) — fusion_analyze→moa, Strategien als Daten, Thinking medium, Preise

Umgesetzt nach Freigabe der offenen Architekturentscheidungen. Betroffene Dateien:

| Datei | Änderung |
|-------|----------|
| `~/.pi/agent/extensions/moa-enhanced/` | **Umzug** von `C:/AI/PI1/.pi/extensions/` nach global + neue Parameter `models`, `mode` |
| `~/.pi/agent/extensions/fusion-switch.ts` | fusion_analyze entfernt, Strategien→JSON-Loader, `/fusion`→`/moa`; 793 → 505 Zeilen |
| `~/.pi/agent/extensions/strategies/*.json` | **NEU** — 5 Strategien als editierbare Daten |
| `~/.pi/agent/models.json` | `openrouter/fusion` entfernt (3 statt 4 Custom-Modelle) |
| `~/.pi/agent/settings.json` | `defaultThinkingLevel`: xhigh → medium |
| `C:/AI/PI1/.pi/skills/moa-skill/SKILL.md` | Auf Zwei-Tool-Welt umgeschrieben |

**1. fusion_analyze in moa verschmolzen:**

- `moa` hat jetzt `mode="synthesize"` (Default, bisheriges Verhalten) und `mode="analyze"` (strukturierte Meta-Analyse: Konsens / Widersprüche / Lücken & blinde Flecken / Einordnung — die Rolle des alten fusion_analyze, aber mit kontrolliertem eigenem Aggregator-Prompt statt serverseitigem Blackbox-Judging).
- Neuer `models`-Parameter (max 4 OpenRouter-Slugs) ersetzt das feste Fusion-Panel: Premium-Analysen laufen über z.B. Opus+GPT+Gemini als Referenzen, Standard bleibt GLM 5.2 + DeepSeek V4 Pro.
- **moa-enhanced wurde nach global verschoben** (`~/.pi/agent/extensions/moa-enhanced/`) — nötig, damit das globale Keyword-Routing in fusion-switch auf `moa` zeigen darf (das Tool muss in jedem Projekt existieren). Nebeneffekt: der fragile laufwerksübergreifende Lib-Import (bekanntes Problem 7) ist behoben.
- fusion-switch: Command `/fusion` → `/moa [analyze|synthesize]`; Auto-Routing bei Vergleichs-/Bewertungs-Keywords instruiert jetzt `moa mode="analyze"`. Tool-Registrierung fusion_analyze komplett entfernt.
- `openrouter/fusion` aus models.json gestrichen.
- Referenz-maxTokens 1500 → 2500 (abgeschnittene Referenzen verschlechterten die Synthese), Referenzen laufen mit reasoningEffort high.

**2. Strategien als Daten:**

- Die 5 deliberate-Strategien (~350 Zeilen Prompt-Text) liegen jetzt als JSON in `~/.pi/agent/extensions/strategies/` (Dateiname = Strategie-Key, `systemPrompt` als Zeilen-Array). Neue Strategie = Datei ablegen + pi-Neustart, kein Code-Change. Defekte Dateien werden geloggt und übersprungen; leeres Verzeichnis wird beim Laden gemeldet.

**3. Thinking-Level:**

- `defaultThinkingLevel` global von `xhigh` auf `medium` — xhigh traf jeden trivialen Prompt (größter laufender Kostenposten). Die Qualitäts-Pfade holen sich Reasoning-Tiefe selbst: deliberate-Phasen high/xhigh (per Strategie-JSON), moa-Referenzen high. Echtes Auto-Scaling pro Prompt bräuchte eine pi-Runtime-API (bekanntes Problem 8); für einzelne schwere Prompts manuell hochschalten.

**4. Preise verifiziert (OpenRouter-Listenpreise, 2026-07-07):**

| Modell | In/1M | Out/1M | Vorher in Doku |
|--------|------:|-------:|----------------|
| GLM 5.2 | $0.91 | $2.86 | „$5.80" (moa-Kommentar) bzw. „$3.00" (Sektion 7) — beide ungenau |
| Claude Opus 4.8 | $5.00 | $25.00 | „$30" (Pipeline-Diagramm) falsch, „$25" korrekt |
| DeepSeek V4 Pro | $0.435 | $0.87 | korrekt (nach 75%-Preissenkung ggü. $1.74/$3.48) |

Hinweis: OpenRouter ist ein Marketplace — Routen schwanken (GLM-Input z.B. $0.93–3.00 je Provider). Angaben sind Listenpreise.

**Verhaltensänderungen für den Agenten:**

- Tool `fusion_analyze` existiert nicht mehr → `moa` mit `mode="analyze"` verwenden.
- Command `/fusion` existiert nicht mehr → `/moa [analyze|synthesize]`.
- Standard-Chat denkt mit `medium` statt `xhigh`.

**Verifikation:** `tsc --noEmit` sauber (nur erwartete Umgebungsfehler); alle 7 JSON-Dateien validiert; Strategie-Loader-Logik per Node-Smoke-Test geprüft (5 Strategien, 3-4 Phasen). Funktionaler Test in laufender pi-Session steht aus.

---

### 2026-07-07 (Welle 1) — Gemeinsamer OpenRouter-Client + Bug-Fixes

Umgesetzt nach Architektur-Review (Fable 5). Betroffene Dateien:

| Datei | Änderung |
|-------|----------|
| `~/.pi/agent/extensions/lib/openrouter-client.ts` | **NEU** — gemeinsamer Client (→ 2.8) |
| `~/.pi/agent/extensions/fusion-switch.ts` | Client-Duplikat entfernt, 2 Bug-Fixes, ~850 → 793 Zeilen |
| `C:/AI/PI1/.pi/extensions/moa-enhanced/index.ts` | Client-Duplikat entfernt, refine-Default, ~378 → 279 Zeilen |

**Bug-Fixes:**

1. **Tote Trigger-Keywords** (fusion-switch): `"trade.off"` und `"vor.-.nachteil"` waren Regex-Syntax, wurden aber per `String.includes` als Literale verglichen → matchten nie. Betroffen waren sowohl die Auto-Erkennung (Fusion-Routing) als auch die `debate`-Strategie-Wahl. Ersetzt durch Literal-Varianten.
2. **Toter Parameter `analysis_models`** (`fusion_analyze`): wurde entgegengenommen, aber nie an die API durchgereicht — der Call ging immer an das Preset `openrouter/fusion`. Parameter entfernt (ehrlicher als still ignorieren); die ungültigen Default-Slugs (`openai/gpt-latest` etc.) entfielen damit ebenfalls.

**Refactoring (gemeinsamer Client):**

- `callModel` existierte doppelt: moa-Version (Retry, Timeout, Error-Hygiene) vs. fusion-switch-Version (nichts davon, rohe API-Fehlertexte im Chat-Output). Jetzt eine Implementierung in `lib/openrouter-client.ts`, beste Eigenschaften beider: Retry/Backoff + per-Versuch-Timeout (aus moa), Reasoning-Effort + Registry-basiertes `resolveApiKey` (aus fusion-switch).
- Einheitliche Signatur: `callModel(apiKey, model, messages[], options) → { content, tokens, error? }` — wirft nie.
- `deliberate` und `fusion_analyze` profitieren jetzt von Retry + Timeout (vorher: ein 429 in Phase 3/4 warf die Tokens der Vorphasen weg). Phasen-Timeout dort 300s (Reasoning-Läufe), moa nutzt das 60s-Default.
- moa nutzt jetzt pi's Model-Registry für den API-Key (vorher nur `process.env`).

**Verhaltensänderungen:**

- `moa`: `refine`-Default **`true` → `false`** (1 statt 2 Referenz-Runden, ~33% günstiger pro Standard-Call; `refine=true` weiterhin explizit wählbar).
- `fusion_analyze`: Parameter `analysis_models` existiert nicht mehr (war wirkungslos).
- Fehlermeldungen von `deliberate`/`fusion_analyze` enthalten keine rohen API-Antworten mehr — Details stehen im Log (`[openrouter]`-Prefix auf stderr/Konsole).
- Timeout gilt jetzt **pro Versuch**, nicht über alle Retries hinweg (moa-Altverhalten hätte bei Retries das Budget aufgezehrt).

**Doku-Korrekturen:**

- Falscher Code-Kommentar in fusion-switch entfernt (behauptete, GLM 5.2 sei settings.json-defaultModel — tatsächlich `deepseek-v4-pro`).
- Klargestellt, dass das „Judging" bei `fusion_analyze` serverseitig im OpenRouter-Preset passiert, nicht im eigenen Code.

**Verifikation:** `tsc --noEmit --skipLibCheck` auf allen drei Dateien — nur erwartete Umgebungsfehler (`@earendil-works/pi-coding-agent`/`typebox` stellt pi zur Laufzeit bereit, `process` ohne @types/node). Funktionaler Test der Tools steht aus → beim nächsten pi-Start `/deliberate` + `moa` einmal aufrufen.

**Damals offen gelassen — inzwischen umgesetzt:** Welle 2 (fusion_analyze→moa, Thinking medium, Strategien als JSON, Preis-Sync) und Welle 3 (Serena-Denylist, GLM-Default).

---

*Dokumentation erstellt am 2026-07-06, zuletzt aktualisiert 2026-07-07 · 13 Abschnitte · pi v0.80.3 · Windows*

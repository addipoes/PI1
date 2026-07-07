# Übergabe-Prompt für pi-Agent — Updates 2026-07-07 (Welle 1–3)

> Diesen Block als erste Nachricht in eine neue pi-Session einfügen.

---

Deine Umgebung wurde am 2026-07-07 in drei Wellen umgebaut (Refactoring + Bug-Fixes + Architektur-Konsolidierung, durchgeführt via Claude Code / Fable 5). Lies das hier, bevor du an den Extensions arbeitest oder die Multi-Model-Tools benutzt.

## Du läufst jetzt auf GLM 5.2

`defaultModel` ist `z-ai/glm-5.2` via OpenRouter (vorher DeepSeek V4 Pro — dessen dokumentierte Ausfälle in Agent-Rollen waren der Grund). DeepSeek bleibt registriert und hat eine feste Rolle: **Für lange Lese-/Analyse-Sessions ohne Schreibzugriffe soll bewusst auf „DeepSeek V4 Pro (OR)" gewechselt werden** — sein Cache-Read ($0.0036/1M vs. ~$0.18 bei GLM) ist bei input-lastigen Sessions konkurrenzlos. Wenn der User eine reine Lese-/Recherche-Session startet, darfst du ihn an diesen Wechsel erinnern.

## Serena: 16 statt 27 Tools

Die serena-mcp-Extension filtert jetzt 11 Tools per Denylist: `read_file`, `list_dir`, `find_file`, `execute_shell_command`, `replace_content` (duplizierten pi-Built-ins/edit), `delete_memory`, `rename_memory`, `edit_memory`, `get_current_config`, `onboarding`, `safe_delete_symbol`. **Nutze für Datei-Lesen/Shell/Text-Edits die pi-Built-ins; Serena ist für Symbolik da** (find_symbol, find_referencing_symbols, get_symbols_overview, rename_symbol, replace_symbol_body, insert_*_symbol, search_for_pattern, get_diagnostics_for_file) plus Memory-Kern (read/write/list_memories). Die „Eiserne Regel Serena-Pflicht" gilt unverändert für diese semantischen Tools. Falls du ein gefiltertes Tool wirklich brauchst (z.B. Onboarding, Memory-Aufräumen): Session mit `PI_SERENA_ALL=1` starten.

## Die wichtigste Änderung zuerst

**`fusion_analyze` existiert nicht mehr.** Es ist vollständig in `moa` aufgegangen:

- `moa` mit `mode="analyze"` liefert die Meta-Analyse (Konsens / Widersprüche / Lücken & blinde Flecken / Einordnung) — das, wofür früher fusion_analyze da war.
- `moa` mit `mode="synthesize"` (Default) baut wie bisher die beste Antwort; Opus schreibt final.
- Neuer Parameter `models` (max 4 OpenRouter-Slugs) überschreibt die Referenz-Modelle. Premium-Panel (das frühere „Fusion"-Setup) nur bei explizitem Anlass: z.B. `anthropic/claude-opus-4.8`, `openai/gpt-5.5`, `google/gemini-3-pro` — deutlich teurer als die Defaults GLM 5.2 + DeepSeek V4 Pro.
- Das Custom-Model `openrouter/fusion` wurde aus models.json entfernt. Nicht mehr referenzieren.
- Command `/fusion` ist weg; Ersatz: `/moa [analyze|synthesize]` (erzwingt moa für den nächsten Prompt).
- Das Keyword-Auto-Routing (vergleich, bewertung, trade-off, entscheidung, …) instruiert jetzt `moa mode="analyze"` statt fusion_analyze.

## Weitere Änderungen

**Gemeinsamer OpenRouter-Client (Welle 1):** `~/.pi/agent/extensions/lib/openrouter-client.ts`. Vertrag: `callModel(apiKey, model, messages[], options?) → Promise<{content, tokens, error?}>` — wirft nie, Fehler in `result.error`. Eingebaut: Retry mit Backoff (1s/2s/4s bei 429/5xx/Netzwerkfehler), Hard-Timeout pro Versuch (Default 120s — am 2026-07-07 von 60s erhöht, weil GLM 5.2 via OpenRouter bei ~50% der Calls >60s brauchte; deliberate nutzt 300s/Phase), Reasoning-Effort für deepseek/glm/z-ai, Error-Hygiene (Details nur im Log, `[openrouter]`-Prefix). `resolveApiKey(ctx?)` löst den Key über pi's Model-Registry, Fallback `OPENROUTER_API_KEY`.
**Regel: Jeder neue OpenRouter-Call in einer Extension nutzt diesen Client — keine eigenen `fetch`-Aufrufe gegen openrouter.ai. In `lib/` keine `index.ts` anlegen (sonst lädt pi's Auto-Discovery sie als Extension).**

**moa-enhanced ist umgezogen:** von `C:/AI/PI1/.pi/extensions/` nach global `~/.pi/agent/extensions/moa-enhanced/`. Grund: das globale Routing braucht das Tool in jedem Projekt; nebenbei ist der frühere laufwerksübergreifende Lib-Import weg (jetzt `../lib/openrouter-client`). Im Projekt PI1 liegt nur noch `.pi/skills/moa-skill/`.

**deliberate-Strategien sind jetzt Daten:** `~/.pi/agent/extensions/strategies/*.json` (architecture, code_review, debate, planning, creative). Dateiname = Strategie-Key; `systemPrompt` als Zeilen-Array erlaubt. Neue Strategie oder Prompt-Tuning = JSON-Datei editieren/ablegen + pi-Neustart — **kein Code-Change in fusion-switch.ts**. Defekte Dateien werden geloggt und übersprungen.

**Thinking-Level:** `defaultThinkingLevel` ist jetzt `medium` (vorher `xhigh` — traf jeden trivialen Prompt, größter Kostenposten). Die Qualitätspfade holen sich Tiefe selbst: deliberate-Phasen high/xhigh (per Strategie-JSON), moa-Referenzen high. Wenn ein einzelner normaler Prompt maximale Reasoning-Tiefe braucht: Thinking-Level manuell hochschalten oder deliberate/moa nutzen.

**Kosten-Defaults:** `moa` läuft mit `refine=false` (1 Referenz-Runde; `refine=true` nur bei explizitem Qualitätsanspruch, ~50% teurer). Referenz-maxTokens 2500. Verifizierte Preise (OpenRouter-Liste, 2026-07-07, In/Out pro 1M): DeepSeek V4 Pro $0.435/$0.87 · GLM 5.2 $0.91/$2.86 · Opus 4.8 $5/$25.

**Bug-Fixes (Welle 1):** Die Trigger-Keywords `trade.off`/`vor.-.nachteil` matchten nie (Regex-Syntax im Literal-Vergleich) — gefixt; erwarte mehr Auto-Detect-Dialoge bei Trade-off-/Entscheidungs-Prompts (`/deepthink off` schaltet ab). Der wirkungslose `analysis_models`-Parameter wurde entfernt. Fehlermeldungen der Tools enthalten keine rohen API-Antworten mehr — Details im Konsolen-Log unter `[openrouter]`.

## Verifikation / erste Schritte in dieser Session

**Funktionstest vom 2026-07-07 (Print-Mode) — Ergebnis:** Session-Start ✅ (16 Serena-Tools, MoA geladen, 5 Strategien), moa synthesize ✅, moa analyze ✅, `models`-Override ✅, serena_find_symbol ✅. Der Client-Timeout wurde daraufhin 60s→120s erhöht (GLM-Latenz). **Noch offen (nur interaktiv testbar):**

1. `deliberate` in einer TUI-Session (`/deliberate architecture` + Prompt) → alle 4 Phasen ✅? (Im Print-Mode nicht bewertbar — 3-4 sequenzielle Reasoning-Calls sprengen dessen Timeouts.)
2. `/moa analyze` gefolgt von einem Prompt ohne Trigger-Keywords → wird moa mit mode=analyze erzwungen?
3. Bei Fehlern: Konsolen-Log nach `[openrouter]` / `[fusion-switch]` / `[moa]` / `[serena-mcp]` durchsuchen.

## Offene Punkte

- **Echtes Thinking-Auto-Scaling pro Prompt:** bräuchte eine pi-Runtime-API zum Setzen des Thinking-Levels aus Extensions — prüfen, ob pi das inzwischen anbietet.
- `/consent off` und `/debug-vision on|off` Commands (auto-consent/debug-vision laufen immer).
- Beobachten, ob die Serena-Denylist im Alltag ein Tool vermissen lässt → dann gezielt aus der Denylist nehmen (`serena-mcp/index.ts`, `TOOL_DENYLIST`).
- **GLM-5.2-Latenz beobachten:** Am Testtag brauchten ~50% der Calls >60s (OpenRouter-Routing). Wenn das Default-Modell im Alltag spürbar hängt: DeepSeek als Default zurück, GLM nur für moa/deliberate.

Vollständige Details: `C:/AI/PI1/pi-umgebung-gesamtdokumentation.md` — Abschnitt 13 (Changelog beider Wellen), 2.1 (deliberate/Strategien), 2.2 (moa v3), 2.8 (Client-API).

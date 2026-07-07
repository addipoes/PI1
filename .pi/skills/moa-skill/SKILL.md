# MoA Skill — Multi-Perspektiven-Tools für pi

## Übersicht

Du hast Zugriff auf zwei Multi-Perspektiven-Tools (seit 2026-07-07 — das frühere
`fusion_analyze` ist in `moa` aufgegangen: `mode="analyze"` ersetzt es):

| Tool | Was es tut | Modelle | Kosten | Opus schreibt? |
|------|-----------|---------|--------|----------------|
| **`moa`** (mode=synthesize, Default) | Synthese: beste Antwort aus Perspektiven | GLM 5.2 + DSv4 → **Opus** | ~2× | ✅ Ja |
| **`moa`** (mode=analyze) | Meta-Analyse: Konsens, Widersprüche, Lücken | GLM 5.2 + DSv4 → **Opus** | ~2× | ✅ Ja |
| **`deliberate`** | Struktur: Planung, Review, Architektur | 1 Modell, 3-4 Rollen | ~3× | Falls aktiv |

## Wann welches Tool?

```
Brauchst du MEHRERE MODELL-PERSPEKTIVEN?
│
├── Ja → moa
│   ├── Willst du die BESTE ANTWORT (Synthese)?
│   │   └── moa mode=synthesize   ← Default für komplexe Aufgaben
│   │
│   └── Willst du PERSPEKTIVEN VERGLEICHEN (Analyse)?
│       └── moa mode=analyze      ← Wenn Widersprüche/Lücken wichtiger sind
│
└── Nein
    └── deliberate                 ← Strukturiertes Denken, Planung, Code-Review
```

## Faustregeln

- **`moa` (synthesize) ist der Default** für die meisten komplexen Aufgaben. Zwei starke,
  günstige Referenz-Modelle (GLM 5.2, DeepSeek V4) liefern Perspektiven, Opus schreibt
  die polierte Synthese.

- **`moa` (analyze) nimmst du**, wenn du explizit wissen willst: „Worüber sind sich die
  Modelle uneinig?" oder „Was übersehen alle?" — also wenn die Meta-Analyse wichtiger
  ist als die Synthese. Typisch bei Vergleichen, Bewertungen, Trade-offs, Entscheidungen.

- **`models`-Parameter** nur überschreiben, wenn explizit ein Premium-Panel gewünscht ist
  (z.B. `anthropic/claude-opus-4.8`, `openai/gpt-5.5`, `google/gemini-3-pro` — das frühere
  „Fusion"-Setup, deutlich teurer). Max 4 Modelle.

- **`refine=false` ist Default** (1 Runde). `refine=true` (2 Runden Peer-Refinement,
  ~50% teurer) nur bei explizitem Qualitätsanspruch.

- **`deliberate` nimmst du** für Architektur-Entwürfe, Code-Reviews, RFCs, Roadmaps —
  Aufgaben, bei denen ein Modell durch strukturierte Rollen (Architekt→Kritiker→Synthesizer)
  bessere Ergebnisse liefert als mehrere Modelle. Strategien liegen als JSON in
  `~/.pi/agent/extensions/strategies/` und sind dort erweiterbar.

- **Opus schreibt immer den finalen Output** bei `moa` (beide Modi). Das ist gewollt —
  Opus hat den besten Schreibstil aller verfügbaren Modelle.

## Preise (OpenRouter, Stand 2026-07-07, pro 1M Tokens)

| Modell | Input | Output |
|--------|------:|-------:|
| DeepSeek V4 Pro | $0.435 | $0.87 |
| GLM 5.2 | $0.91 | $2.86 |
| Claude Opus 4.8 | $5.00 | $25.00 |

## Anti-Patterns

- ❌ `moa` für triviale Fragen („Was ist 2+2?") → Einfach direkt antworten
- ❌ `mode=analyze` wenn du nur eine gute Antwort brauchst → `mode=synthesize` reicht
- ❌ `mode=synthesize` wenn du eine Meta-Analyse willst → `mode=analyze`
- ❌ Frontier-`models`-Panel ohne expliziten Anlass → Default-Referenzen reichen fast immer
- ❌ Beide Tools hintereinander für dieselbe Frage → Pick one

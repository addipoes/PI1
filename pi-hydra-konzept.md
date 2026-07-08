# pi/HYDRA — Agent Harness für Text, Code & Infrastruktur

> Optimierte Konzeptfassung · 2026-07-07  
> Quellen: Damon ADE (Pat Simmons), pi/ADE, pi/MESH, pi/HYDRA (deliberate-Synthese), moa-Analyse

---

## 0. Executive Summary

**pi/HYDRA** ist eine Orchestrierungsschicht über dem bestehenden pi-Ökosystem. Sie löst zwei Kernprobleme:

1. **Fragmentierung**: Statt verteilter Chat-Fenster (pi, Claude Code, Terminal) → eine zentrale Arbeitsumgebung
2. **Kontext-Chaos**: Statt flüchtigem RAM-Kontext → strukturierte Sessions mit Git-Worktree-Isolation und RAG-Memory

Anders als Damon ADE (YouTube/Content-Fokus) ist pi/HYDRA für **Textarbeit, Programmierung und Infrastruktur** optimiert. Anders als ein reines Tabs-Modell nutzt es ein **Hybrid aus interaktivem Fast-Path und asynchronem Actor-Path**.

---

## 1. Architektur

### 1.1 Dual-Path-Modell

```
┌─────────────────────────────────────────────────────────┐
│                    pi/HYDRA UI                            │
│  ┌───────────────────┐  ┌─────────────────────────────┐ │
│  │   FAST-PATH       │  │   ACTOR-PATH                 │ │
│  │   (Tabs)          │  │   (Task Board)               │ │
│  │                   │  │                              │ │
│  │  Interaktiv       │  │  Asynchron                   │ │
│  │  Sofort-Antwort   │  │  Task → Claim → Work → Done  │ │
│  │  Shared Workspace │  │  Git-Worktree-Isolation      │ │
│  │  Für: Text, Query │  │  Für: Code, Refactor, Infra  │ │
│  └────────┬──────────┘  └──────────────┬──────────────┘ │
│           │                            │                 │
├───────────┼────────────────────────────┼─────────────────┤
│           ▼                            ▼                 │
│  ┌────────────────────────────────────────────────────┐  │
│  │              AGENT REGISTRY                         │  │
│  │  Pydantic-validierte Profile (YAML → Code)         │  │
│  │  Definieren: Identität, Default-Model, Capabilities │  │
│  └────────────────────────────────────────────────────┘  │
│           │                            │                 │
│           ▼                            ▼                 │
│  ┌──────────────┐    ┌─────────────────────────────────┐ │
│  │ WORKSPACE    │    │ ACTOR CLUSTER                   │ │
│  │ MANAGER      │    │ (Supervision Tree)              │ │
│  │              │    │                                 │ │
│  │ Shared WS    │    │ Orchestrator → Architekt        │ │
│  │ (Text/Infra) │    │             → Implementer       │ │
│  │              │    │             → Reviewer          │ │
│  │ Git-Worktree │    │             → RetroActor        │ │
│  │ (Code-Tasks) │    │                                 │ │
│  └──────┬───────┘    └───────────────┬─────────────────┘ │
│         │                            │                   │
│         ▼                            ▼                   │
│  ┌────────────────────────────────────────────────────┐  │
│  │           EVENT BUS + RAG MEMORY                   │  │
│  │  Append-only Event-Log (SQLite)                    │  │
│  │  Vektor-Embeddings → semantische Queries           │  │
│  │  Nie voll geladen, nur Top-K relevant              │  │
│  └────────────────────────────────────────────────────┘  │
│                         │                                 │
│                         ▼                                 │
│  ┌────────────────────────────────────────────────────┐  │
│  │              CAPABILITY LAYER                       │  │
│  │  moa · deliberate · sprint · serena · browser      │  │
│  └────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────┤
│                     pi CORE                              │
│  Extensions · Skills · Memories · OpenRouter            │
└─────────────────────────────────────────────────────────┘
```

### 1.2 Fast-Path ↔ Actor-Path

| Kriterium | Fast-Path | Actor-Path |
|-----------|-----------|------------|
| **UI** | Tabs (Kategorie → Agent → Tab) | Task Board (Submitted → In-Progress → Done) |
| **Latenz** | <5s (direkte LLM-Antwort) | Minuten bis Stunden (Multi-Step) |
| **Isolation** | Shared Workspace (RAM-Kontext) | Git-Worktree (physisch isoliert) |
| **Memory** | Session-Kontext + minimale RAG-Query | Volles Event-Log + RAG |
| **Fehler** | User sieht sofort | Supervision Tree restartet |
| **Typische Tasks** | "Erkläre X", "Schreibe Docstring" | "Implementiere OAuth2", "Refaktoriere Auth" |

---

## 2. Agent-Typen

### 2.1 Kategorien (adaptiert für Text/Code/Infra)

```
📂 Text & Content
  ├── technical-writer    (moa:synthesize, deliberate:creative)
  ├── doc-reviewer        (deliberate:code_review für Doku)
  └── spec-writer         (deliberate:planning + moa:analyze)

📂 Programming
  ├── code-architect      (deliberate:architecture + serena:readonly + moa:analyze)
  ├── implementer         (sprint:lead-worker + serena:full)
  ├── code-reviewer       (deliberate:code_review + serena:readonly + moa:synthesize)
  ├── debugger            (serena:full + browser für Repro)
  └── refactorer          (deliberate:debate + serena:full)

📂 Infrastructure
  ├── infra-planner       (deliberate:architecture + moa:analyze)
  ├── iac-generator       (sprint + serena für bestehende IaC)
  ├── security-auditor    (deliberate:code_review + moa:analyze)
  └── deployment-strategist (deliberate:planning + moa:synthesize)

📂 Meta
  ├── orchestrator        (sprint:lead — verteilt an andere Agents)
  ├── researcher          (browser + moa:synthesize)
  └── retro-agent         (moa:analyze → Memory-Updates, asynchron)
```

### 2.2 Agent-Profil (Beispiel)

```yaml
# ~/.pi/agents/code-architect/agent.yaml
agent: "code-architect"
category: "programming"
description: "Architekturentscheidungen treffen, Systemdesign bewerten"

competencies:
  - serena:readonly       # Code analysieren, nicht editieren
  - deliberate:architecture
  - deliberate:debate
  - moa:analyze           # Multi-Perspektiven-Analyse

default_model: "anthropic/claude-opus-4.8"
fallback_model: "z-ai/glm-5.2"

memory_scope: "project"

system_prompt: "agent.md"   # Identität & Arbeitsweise
handoff_targets: ["implementer", "code-reviewer", "infra-planner"]
```

---

## 3. Memory & State

### 3.1 Drei logische Tiers, eine technische Basis

```
┌────────────────────────────────────┐
│ SESSION (pro Tab/Task)             │
│ → In-Memory, serialisierbar        │
│ → Wird bei Handoff übergeben       │
├────────────────────────────────────┤
│ PROJECT (<project>/.pi/memory/)    │
│ → Entscheidungen (ADR-Format)      │
│ → Konventionen, Patterns           │
│ → RAG-queriable, nie voll geladen  │
├────────────────────────────────────┤
│ AGENT (~/.pi/agents/<agent>/)      │
│ → agent.md: Identität & Style      │
│ → memory.md: Gelernte Muster       │
│ → Self-improving (asynchron)       │
└────────────────────────────────────┘
```

### 3.2 Event-Log (technische Basis)

Jede Aktion wird als Event gespeichert. Memory wird nicht "geladen" — sondern **semantisch abgefragt**:

```json
// events/2026-07-07/task-auth-001/004-decision.json
{
  "event": "decision_made",
  "task": "task-auth-001",
  "decision": "PKCE flow statt Implicit Grant",
  "rationale": "Security-Audit fordert PKCE. Implicit Grant ist deprecated per OAuth 2.1.",
  "actor": "code-architect",
  "timestamp": "2026-07-07T15:30:00Z"
}
```

**Query (RAG):** *"Warum haben wir PKCE gewählt?"* → Top-3 relevante Events in Kontext injiziert.

### 3.3 Self-Improving (asynchron, nie blockierend)

```
Session endet
  → RetroActor subscribed auf Task-Events (Hintergrund)
  → Analysiert Event-Log: Was lief gut? Wo gab's 3 Retries?
  → Schreibt Erkenntnisse in agent/memory.md
  → Erstellt Improvement-Vorschlag → User reviewed später
```

---

## 4. Capability Negotiation

Statt starrer Kompetenz-Bündel: **Default-Caps + Laufzeit-Anforderung**.

```
ImplementerActor arbeitet an Code-Task
  → Braucht browser-Capability für OAuth-Callback-Test
  → browser nicht in initialen Caps → sendet CapabilityRequest
  → User sieht in UI: "Implementer needs browser for: OAuth callback test. [Approve] [Deny]"
  → User approved → Capability temporär gebunden
  → Nach Task-Ende: Capability freigegeben
```

---

## 5. Umsetzungsplan (optimiert)

### Phase 1: MVP — Fast-Path (2 Wochen)

**Ziel**: Interaktive Tabs mit Agent-Profilen laufen.

- [ ] TUI-Shell (Tabs) als pi-Extension
- [ ] Agent Registry (Pydantic-validiert, 5 Basis-Agenten)
- [ ] Model-Picker (OpenRouter, bestehend)
- [ ] Shared Workspace für Text-Tasks
- [ ] **Nicht**: Actor-System, Git-Worktrees, RAG — das kommt später

**Lieferbar**: User kann Agenten-Profil wählen und interaktiv arbeiten.

### Phase 2: Event-Infrastruktur (2 Wochen)

- [ ] Append-only Event-Store (SQLite)
- [ ] Basis-Event-Typen (TaskSubmitted, DecisionMade, CodeModified)
- [ ] Vektor-Embeddings + RAG-Query (lokal, kein externer Service)
- [ ] Project-Memory als RAG-queriable Sammlung

### Phase 3: Workspace-Isolation (2 Wochen)

- [ ] Git-Worktree-Manager (create/review/merge/cleanup)
- [ ] Actor-Path-Trigger: Code-Task → Worktree statt Shared
- [ ] Basis-Merge-Workflow (kein Auto-Resolver)

### Phase 4: Actor-System (3 Wochen)

- [ ] Basis-Actor-Framework (asyncio, Mailbox, Receive-Loop)
- [ ] OrchestratorActor, ImplementerActor, ReviewerActor
- [ ] Capability-Request-Protokoll + UI
- [ ] Supervision Tree (Timeouts, Circuit-Breaker)

### Phase 5: Retro + Sharing (2 Wochen)

- [ ] RetroActor (asynchron, Event-Log-Analyse)
- [ ] Agent-Template-Export/Import (GitHub)
- [ ] Sandboxing für importierte Skills (AST-Validation)

**Gesamt**: 11 Wochen (optimistisch) bis 14 Wochen (realistisch mit Puffer).

---

## 6. Risiko-Matrix

| # | Risiko | Impact | Wahrsch. | Mitigation |
|---|--------|:---:|:---:|---|
| 1 | **Komplexitätsspirale**: Hybrid aus Tabs + Task-Board wird unübersichtlich | Hoch | Mittel | MVP nur Tabs (Phase 1). Task-Board erst ab Phase 4. User testet schrittweise. |
| 2 | **Fast-Path-Memory-Divergenz**: Fast-Path umgeht RAG → anderes Memory als Actor-Path → Inkonsistenz | Mittel | Hoch | Fast-Path nutzt minimale RAG-Query (Top-3 Events, <500ms). Gleicher Event-Store. |
| 3 | **GLM 5.2 Latenz**: Default-Modell unzuverlässig → Fast-Path nicht "fast" | Mittel | Mittel | Fallback auf DeepSeek V4 Pro. Timeout-Erkennung + Auto-Switch. |

---

## 7. Bewertung: Over-Engineering?

**Nein, aber phasenweise Einführung ist kritisch.**

Der Fehler wäre, alle 5 Phasen vor dem ersten User-Test zu bauen. Stattdessen:

1. **Phase 1 allein** liefert bereits Mehrwert (strukturierte Agent-Tabs statt Chaos)
2. **Phasen 2-3** adressieren echte Schmerzpunkte (Memory, Code-Isolation)
3. **Phasen 4-5** sind "nice to have" — nur bauen wenn Phase 1-3 sich bewährt

Das Hybrid-Modell klingt komplex, ist aber in der Praxis simpler als die Alternative (reiner Actor-Path für alles). Der Fast-Path verhindert, dass triviale Tasks den vollen Actor-Overhead zahlen.

---

## 8. Was wir NICHT bauen (bewusste Auslassungen)

- ❌ **Eigenes Actor-Framework von Grund auf** — Leichtgewichtige asyncio-Implementierung reicht
- ❌ **Community Agent Registry** (Phase 5) — Erst wenn interne Agents stabil
- ❌ **Web-Dashboard** — TUI first, Web ist optionale Spätphase
- ❌ **Auto-Merge-Resolver** — Merge-Konflikte sind bewusst manuell (User-Entscheidung)
- ❌ **Vollständiges RAG-System mit externer Vektor-DB** — Lokale Embeddings (SQLite + sqlite-vss) reichen

---

## 9. Nächster Schritt

**Phase 1 starten**: pi-Extension `pi-hydra` mit Tabs + Agent-Registry + Model-Picker.  
Aufsetzpunkt: `~/.pi/agent/extensions/pi-hydra/index.ts` (analog zu moa-enhanced).  
Skills liegen in `~/.pi/agents/` als YAML + MD.

---

*Dokument versioniert in: C:/AI/PI1/pi-hydra-konzept.md*

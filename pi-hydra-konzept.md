# pi/HYDRA — Agent Harness für Text, Code & Infrastruktur

> Optimierte Konzeptfassung v2 · 2026-07-07  
> Reviews: deliberate (Architekt→Kritiker→Gegenentwurf→Synthesizer) + moa:analyze (GLM 5.2 + DeepSeek V4 Pro → Opus)  
> Quellen: Damon ADE (Pat Simmons), pi/ADE, pi/MESH

---

## 0. Strategische Vorfragen (vor jeder Architektur-Entscheidung)

### 0.1 Für wen ist das?

| Option | Konsequenzen |
|--------|-------------|
| **Personal Tool** (1 Dev, lokaler Rechner) | Kein Auth, kein Multi-Tenant, kein Cluster. `asyncio`-Tasks reichen. MVP in 4-6 Wo. |
| **Team Platform** (3-10 Devs, geteilter Server) | Auth, Isolation, Queues, Monitoring nötig. Eher 15-20 Wo. |

**Entscheidung steht aus.** Fast alle Architektur-Fragen (Actor-Cluster ja/nein, Sandbox-Typ, UI) hängen davon ab. **Vor Phase 1 klären.**

### 0.2 Build vs. Adopt?

Existierende Harnesses wurden nicht als Alternative geprüft:

| Tool | Stärke | Schwäche für unseren Case |
|------|--------|--------------------------|
| **Aider** | Code-zentriert, Git-nativ, Multi-Model | Kein Infra/Text-Fokus, kein Agent-Profil-System |
| **OpenHands** | Web-UI, Docker-Sandbox, Multi-Agent | Schwergewichtig, Python-Stack, nicht pi-nativ |
| **Claude Code** | Exzellentes Coding, Terminal-nativ | Vendor-Lock-in, kein OpenRouter, keine eigenen Agents |
| **pi selbst** | Extensions, Skills, OpenRouter | Kein Task-Board, keine Worktree-Isolation |

**Fazit:** Kein bestehendes Tool deckt unseren Stack (pi + OpenRouter + moa/deliberate/sprint + Text/Code/Infra) ab. Eigenbau ist gerechtfertigt, aber der MVP muss sich an der Einfachheit von Aider/Claude Code messen lassen — nicht an der Komplexität von OpenHands.

---

## 1. Executive Summary

**pi/HYDRA** ist eine Orchestrierungsschicht über dem bestehenden pi-Ökosystem. Sie löst zwei Kernprobleme:

1. **Fragmentierung**: Statt verteilter Chat-Fenster (pi, Claude Code, Terminal) → eine zentrale Arbeitsumgebung
2. **Kontext-Chaos**: Statt flüchtigem RAM-Kontext → strukturierte Sessions mit Workspace-Isolation und durchsuchbarem Memory

Anders als Damon ADE (YouTube/Content-Fokus) ist pi/HYDRA für **Textarbeit, Programmierung und Infrastruktur** optimiert. Anders als ein reines Tabs-Modell nutzt es ein **Hybrid aus interaktivem Fast-Path und asynchronem Actor-Path**.

---

## 2. Architektur

### 2.1 Dual-Path-Modell

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
│  │ WORKSPACE    │    │ TASK RUNNER                     │ │
│  │ MANAGER      │    │ (asyncio-Tasks + Timeouts,      │ │
│  │              │    │  später: Celery/Redis)          │ │
│  │ Shared WS    │    │                                 │ │
│  │ (Text/Infra) │    │ Orchestrator → Architekt        │ │
│  │              │    │             → Implementer       │ │
│  │ Git-Worktree │    │             → Reviewer          │ │
│  │ (Code-Tasks) │    │             → RetroActor        │ │
│  └──────┬───────┘    └───────────────┬─────────────────┘ │
│         │                            │                   │
│         ▼                            ▼                   │
│  ┌────────────────────────────────────────────────────┐  │
│  │           MEMORY LAYER                              │  │
│  │  SQLite/JSON-Historie (MVP) → RAG (später)         │  │
│  │  Nie voll geladen, nur relevante Einträge           │  │
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

### 2.2 Fast-Path ↔ Actor-Path

| Kriterium | Fast-Path | Actor-Path |
|-----------|-----------|------------|
| **UI** | Tabs (Kategorie → Agent → Tab) | Task Board (Submitted → In-Progress → Done) |
| **Latenz** | <5s (direkte LLM-Antwort) | Minuten bis Stunden (Multi-Step) |
| **Isolation** | Shared Workspace (RAM-Kontext) | Git-Worktree (physisch isoliert) |
| **Memory** | Session-Kontext + JSON-Historie | SQLite-Event-Log + Embedding-Suche (später) |
| **Fehler** | User sieht sofort | Retry mit Timeout, User-Benachrichtigung |
| **Typische Tasks** | "Erkläre X", "Schreibe Docstring" | "Implementiere OAuth2", "Refaktoriere Auth" |
| **Approval** | Keiner (interaktiv) | Approval-Gates für destructive Actions |

### 2.3 Was wir NICHT im MVP bauen

- ❌ Supervision Trees mit Circuit-Breakern → einfache `asyncio`-Tasks mit Timeouts
- ❌ Eigenes Actor-Framework → später Redis/NATS + Celery/Temporal
- ❌ RAG mit Vektor-DB → SQLite/JSON-Historie reicht für MVP
- ❌ Event Sourcing → zu schwergewichtig für Phase 1. JSON-Event-Log in SQLite.
- ❌ Community Agent Registry → erst wenn intern stabil
- ❌ Web-Dashboard → TUI first

---

## 3. Agent-Typen

### 3.1 Kategorien

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

### 3.2 Agent-Profil (Beispiel)

```yaml
# ~/.pi/agents/code-architect/agent.yaml
agent: "code-architect"
category: "programming"
description: "Architekturentscheidungen treffen, Systemdesign bewerten"

competencies:
  - serena:readonly
  - deliberate:architecture
  - deliberate:debate
  - moa:analyze

default_model: "anthropic/claude-opus-4.8"
fallback_model: "z-ai/glm-5.2"

memory_scope: "project"
system_prompt: "agent.md"
handoff_targets: ["implementer", "code-reviewer", "infra-planner"]
```

---

## 4. Memory & State

### 4.1 MVP: SQLite/JSON-Historie

Jeder Task/Session produziert einen JSON-Eintrag. Kein RAG, keine Embeddings — einfache Datei-basierte Suche.

```json
// ~/.pi/hydra/memory/task-auth-001.json
{
  "task": "task-auth-001",
  "agent": "implementer",
  "timestamp": "2026-07-07T15:30:00Z",
  "summary": "OAuth2 PKCE flow implemented in src/auth.py",
  "decisions": [
    {"what": "PKCE statt Implicit Grant", "why": "Security-Audit fordert PKCE, Implicit deprecated"}
  ],
  "files_changed": ["src/auth.py", "tests/test_auth.py"],
  "status": "done"
}
```

### 4.2 Später: RAG (nur bei Bedarf)

Wenn SQLite/JSON-Suche nicht mehr reicht (>100 Einträge):
- Embeddings via lokalem Modell (kein externer Service)
- Vektor-Suche in SQLite (sqlite-vss)
- Nur Top-K relevante Einträge in Kontext injiziert

### 4.3 Self-Improving (asynchron, nie blockierend)

```
Session endet
  → RetroActor subscribed auf Task-Events (Hintergrund)
  → Analysiert: Was lief gut? Wo gab's Retries?
  → Schreibt Erkenntnisse in agent/memory.md
  → User reviewed später
```

---

## 5. Sicherheit

### 5.1 Execution-Sandbox (MVP: Docker)

Jeder Code/Infra-Task läuft in einem Docker-Container — kein direkter Host-Zugriff.

```
Actor-Path-Task "Implementiere OAuth2"
  → Workspace Manager erstellt Git-Worktree
  → Task Runner startet Docker-Container mit Worktree-Volume
  → ImplementerActor arbeitet IM Container
  → ReviewerActor reviewed Diff von außen
  → Merge nur nach Approval
```

### 5.2 Approval-Gates (MVP: interaktiv)

| Aktion | Approval |
|--------|----------|
| Code edit im Worktree | Keiner (isoliert) |
| Merge Worktree → Main | User-Bestätigung |
| Shell-Befehl im Container | Keiner (isoliert) |
| Shell-Befehl auf Host | **User-Bestätigung + 2FA-artig: "BIST DU SICHER?"** |
| `terraform apply` / `kubectl delete` | User-Bestätigung + Dry-Run-Vorschau |
| Datei außerhalb Worktree löschen | **Immer blockiert** |

### 5.3 Prompt Injection (strukturell)

Referenz-Outputs von Agents werden immer in `<agent_output>`-Tags gekapselt und als untrusted markiert (gleicher Mechanismus wie moa-enhanced).

---

## 6. Kritische Design-Entscheidung: Merge-Strategie

**Das größte architektonische Risiko** (von GLM 5.2 identifiziert, von DeepSeek übersehen):

Zwei Actors arbeiten parallel in isolierten Worktrees. Beide modifizieren `src/auth.py`. Was passiert beim Merge?

```
Worktree A: Refactored login() → neue Signatur
Worktree B: Fügt MFA zu login() hinzu → alte Signatur
Merge: KONFLIKT
```

**Strategie: Sequentiell, nicht parallel (MVP)**

- Actor-Path-Tasks für dasselbe Modul werden **sequentiell** abgearbeitet
- Task B startet erst, wenn Task A gemerged ist
- Parallele Tasks nur für **disjunkte** Module (Auth ≠ Frontend)
- Kein Auto-Merge-Resolver — Konflikte sind bewusst manuell

Das begrenzt Parallelität, eliminiert aber die häufigste Fehlerquelle.

---

## 7. Risiko-Matrix

| # | Risiko | Impact | Wahrsch. | Mitigation |
|---|--------|:---:|:---:|---|
| 1 | **Merge-Hölle**: Parallele Worktree-Tasks erzeugen Konflikte | Hoch | Mittel | Sequentiell pro Modul (MVP). Kein Auto-Merge. |
| 2 | **UX-Bruch TUI/Async**: Tabs (synchron) + Task-Board (asynchron) in einer TUI verwirrt | Mittel | Hoch | In Tabs starten, Task-Board erst ab Phase 3. DeepSeek hat recht: langfristig Web-UI. |
| 3 | **Komplexitätsspirale**: MVP wächst unkontrolliert → Over-Engineering | Hoch | Mittel | MVP radikal beschneiden. Abbruchkriterium nach jeder Phase. |
| 4 | **Prompt Injection in Agents**: Actor führt schädlichen Code aus | Kritisch | Niedrig | Docker-Sandbox + Approval-Gates + Output-Tagging |
| 5 | **GLM 5.2 Latenz**: Default-Modell unzuverlässig → Fast-Path nicht "fast" | Mittel | Mittel | Fallback DeepSeek. Timeout-Erkennung + Auto-Switch. |

---

## 8. Umsetzungsplan

### Phase 1: MVP — Fast-Path (3-4 Wochen)

**Ziel**: Interaktive Tabs mit Agent-Profilen laufen. Kein Actor-Path.

- [ ] TUI-Shell (Tabs) als pi-Extension
- [ ] Agent Registry (Pydantic-validiert, 3 Basis-Agenten: Text, Code, Infra)
- [ ] Model-Picker (OpenRouter, bestehend)
- [ ] Shared Workspace
- [ ] **Nicht**: Actor-System, Git-Worktrees, RAG, Docker-Sandbox

**Abbruchkriterium**: Wenn Tabs nicht spürbar besser als rohes pi → Konzept überdenken.

### Phase 2: Workspace-Isolation (2-3 Wochen)

- [ ] Git-Worktree-Manager (create/review/merge/cleanup)
- [ ] Docker-Sandbox für Code-Tasks
- [ ] Approval-Gates für Merge
- [ ] SQLite/JSON-Memory

### Phase 3: Asynchrone Tasks (3-4 Wochen)

- [ ] Task-Board-UI
- [ ] `asyncio`-basierte Task-Ausführung mit Timeouts
- [ ] Sequentiell pro Modul, parallel für disjunkte Module
- [ ] Retry-Logik (max 3x, dann User-Benachrichtigung)

### Phase 4: Actor-Intelligenz (3-4 Wochen)

- [ ] OrchestratorActor, ImplementerActor, ReviewerActor
- [ ] Capability-Request-Protokoll + UI
- [ ] RetroActor (asynchron, Memory-Updates)
- [ ] Redis/NATS für Task-Queue (ersetzt reines asyncio)

### Phase 5: Produktion (optional, fortlaufend)

- [ ] RAG (Embedding-Suche statt JSON)
- [ ] Web-Dashboard
- [ ] Agent-Template-Sharing (GitHub)
- [ ] Multi-User (falls Team-Platform-Entscheidung)

**Gesamt realistische Schätzung: 13-19 Wochen** (nicht 11).

---

## 9. Zusammenfassung

**pi/HYDRA ist gerechtfertigt, aber der MVP muss sich radikal beschneiden lassen.**

Der größte Wert liegt in **Phase 1 allein**: strukturierte Agent-Tabs mit Profilen, Model-Picker, Shared Workspace. Das eliminiert das Chat-Fenster-Chaos ohne eine Zeile Actor-Code.

Alles darüber hinaus (Worktrees, asynchrone Tasks, RAG) nur bauen, wenn Phase 1 sich im Alltag bewährt hat.

---

*Review-Stats: 2 Modelle, 0 Widersprüche, 5 blinde Flecken identifiziert, 4 strategische Lücken geschlossen.*

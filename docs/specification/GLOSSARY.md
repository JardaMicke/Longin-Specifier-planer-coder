# Glossary — L.O.N.G.I.N. Coder

Projektový slovník závazných termínů. Nesmí se používat synonyma, pokud by mohla způsobit záměnu.

| Termín | Definice | Nepoužívat jako synonymum | Vlastník |
|--------|----------|---------------------------|----------|
| **L.O.N.G.I.N.** | Logical Orchestrated Networked Generative Intelligent Nexus — název systému | zkratka bez rozepsání | System Architecture |
| **CORE** | Centrální řídící jádro systému. Koordinuje ganglia, routing, globální stav, event orchestration, lifecycle, health monitoring. Nevlastní automaticky všechny datové operace — ownership se rozhoduje případ od případu. | Module, Server, Backend | System Architecture |
| **Nexus** | Zařízení nebo uzel v klastru. V kontextu Longin Coder = lokální počítač (Windows) s hardwarem (CPU/GPU/RAM/disk), na kterém běží Docker kontejnery a lokální LLM modely (Ollama, LM Studio). | Node, Module, Device | Infrastructure |
| **Ganglion** | Samostatný funkční subsystém aplikace s vlastními odpovědnostmi, daty, logikou, UI, Views, rozhraními a vazbami na CORE. Nahrazuje obecný pojem "module". | Module, Component, Service | Architecture |
| **View** | Konkrétní zobrazovací stav UI v rámci Ganglionu. Má unikátní ID (VIEW-XXX), route, layout, data contract, state machine, flows, AC. | Page, Screen, Route | UI Architecture |
| **Application Shell** | Společná kostra UI: Global Header + Left Navigation + View Content. | Layout, Template | UI Architecture |
| **Edit Mode** | Přepínač v Headeru určující, zda jsou UI komponenty editovatelné. Má definovaný default state, permission model, autosave/explicit save, unsaved changes handling. | Toggle, Switch | UI Architecture |
| **Decision Log** | Záznam každého důležitého rozhodnutí: ID, datum, téma, kontext, varianty, zvolená varianta, důvod, důsledky, dotčené systémy, stav (PROPOSED/APPROVED/SUPERSEDED). | changelog, notes | Process |
| **State Machine** | Stavový automat fáze vývoje: BRAINSTORMING → PLANNING → VÝVOJ → (DEPLOY). Každý přechod má trigger, podmínky, eventy, persistence. | Workflow, Pipeline | Process |
| **MCP Server** | Model Context Protocol server poskytující tools a skills pro coding agenta. | API server, Tool server | Integration |
| **IDE Bridge** | Most mezi Longin Coder a externími IDE (VS Code, Trae, Antigravity). Protokol + transport (WebSocket/LSP). | Plugin, Extension | Integration |
| **Synapse Renderer** | Komponenta Canvas/WebGL renderující pozadí: noční obloha hvězd tvořená synapsemi, občas vzruch (světélko na dráze mezi nimi). | Background, Animation | UI Architecture |
| **Vector Store** | PGVector rozšíření PostgreSQL pro ukládání embeddingů, RAG retrieval, cache. | Embedding DB | Data |
| **Coding Agent** | AI agent (LangGraph orchestrace) provádějící implementaci podle plánu. Běží lokálně, hlídá se výkon. | Developer, Bot | Process |
| **Specification Agent** | AI agent provádějící analýzu a tvorbu specifikace (instrukce v docs/instructions/agent-instructions.md). | Analyst, Spec writer | Process |
| **KOBLIHA** | Bezpečnostní slovo — agent jej napíše, pokud by systémová omezení nutila lhát nebo zamlčovat fakta. | — | Safety |

## Terminology Hierarchy

```
CORE
│
├── NEXUS (lokální stroj: Windows, Docker, Ollama, LM Studio)
│   │
│   ├── GANGLION: LLM Manager
│   │   ├── VIEW-005: Models Dashboard
│   │   └── VIEW-006: Model Detail
│   │
│   ├── GANGLION: State Machine
│   ├── GANGLION: Conversation
│   ├── GANGLION: Vector Store
│   ├── GANGLION: MCP Server
│   ├── GANGLION: IDE Bridge
│   └── GANGLION: Developer Diary
│       ├── VIEW-007: Diary
│       └── ...
│
└── APPLICATION SHELL
    ├── GLOBAL HEADER
    ├── GLOBAL NAVIGATION
    └── GANGLION Views (VIEW Content)
```

## Responsibility Matrix (příklad — bude upřesněn v S-003)

| Odpovědnost | CORE | Ganglion | Nexus |
|-------------|------|----------|-------|
| Globální orchestrace | ✓ | | |
| Doménová logika | | ✓ | |
| Přístup k lokálnímu hardwaru (GPU pro LLM) | | | ✓ |
| UI stav | dle návrhu | ✓ | |
| Persistence | dle návrhu | dle návrhu | |
| Koordinace LLM modelů | ✓ | ✓ (LLM Manager) | ✓ (hardwarový přístup) |

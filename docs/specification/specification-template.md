# Páteřní specifikační dokument — L.O.N.G.I.N. Coder

> **Stav specifikace:** Stupeň 2 — DEFINED
> **Verze:** v0.2
> **Datum:** 2025-01-15
> **Schválená rozhodnutí:** DL-001 až DL-004 (viz DEVELOPER-DIARY.md)

---

## Jak používat tento dokument

Tento dokument je **páteřní šablonou** pro specifikaci celé aplikace L.O.N.G.I.N. Coder. Každá sekce má kolonky, které se vyplňují během BRAINSTORMING fáze.

**Pravidla vyplňování:**
- Každá kolonka má stav: `[ ] specifikovat` / `[ ] neúplné doplnit` / `[x] specifikováno`
- Před změnou poznámky zhodnoť stav kroku
- Pokud něco není jasné, označ jako OPEN DECISION a navrhni 3+1 varianty
- Nikdy nevymýšlej — pokud není definováno, zeptej se

---

## 1. Executive Summary

- Stav: `[x] specifikováno`
- Obsah: L.O.N.G.I.N. Coder (Logical Orchestrated Networked Generative Intelligent Nexus) je lokální, single-user interaktivní vývojářský deník. Uživatel vyvíjí software formou rozhovoru s AI agenty: Specification Agent převádí nápady do technické specifikace, Coding Agent implementuje podle plánu. Vývoj je řízen stavovým automatem (BRAINSTORMING → PLANNING → VÝVOJ → DEPLOY). Veškerá inference i persistence běží lokálně v Docker kontejnerech na jediném stroji (NEXUS).

## 2. Vision

- Stav: `[x] specifikováno`
- Obsah: Software vyvíjený přirozeným rozhovorem s AI, kde agenti běží lokálně a respektují lokální hardware (CPU/GPU/RAM/disk). Deník aplikace je zároveň nástrojem pro vývoj — aplikace „píše sama sebe". Dlouhodobě: jeden uživatel, jeden stroj, plná kontrola nad daty a modely.

## 3. Goals

- Stav: `[x] specifikováno`
- Obsah (měřitelné cíle):
  - G-001: Uživatel vede rozhovor v GANGLION Conversation → AI generuje/aktualizuje specifikaci uloženou v persistenci.
  - G-002: Coding Agent implementuje kroky plánu (P-xxx → D-xxx) se záznamem postupu do Developer Diary.
  - G-003: LLM Manager umožní načíst/přepnout lokální model (Ollama, LM Studio) a cloud model (OpenAI-compatible) za běhu.
  - G-004: Vector Store (PGVector) poskytuje RAG retrieval nad specifikací a deníkem.
  - G-005: MCP Server poskytuje skills/tools pro coding agenta.
  - G-006: IDE Bridge připojí externí IDE (VS Code / Trae / Antigravity) přes WebSocket/LSP.
  - G-007: Vše běží v Docker Compose s healthchecky a lokální sítí mezi kontejnery.

## 4. Non-goals

- Stav: `[x] specifikováno`
- Obsah: Aplikace NENÍ:
  - Multi-user systém (jediný lokální uživatel).
  - Cloud-hosted SaaS (vše lokálně).
  - Massivně škálovatelná (ignorujeme load balancing pro tisíce uživatelů, distribuované cloud DB).
  - Veřejně přístupná služba (žádné vzdálené přihlášení).

## 5. Terminology

- Stav: `[x] specifikováno` (viz GLOSSARY.md)
- Odkaz: `docs/specification/GLOSSARY.md`

## 6. User Personas / Actors

- Stav: `[x] specifikováno`
- Obsah:
  - **Single User (Hlavní actor)** — lokální vývojář, jediný lidský uživatel aplikace. Vede rozhovory, schvaluje rozhodnutí (Decision Log), aktivuje Edit Mode, přepíná modely, spouští plán.
  - **Specification Agent** — AI agent tvořící specifikaci podle instrukcí v `docs/instructions/agent-instructions.md`. Navrhuje 3+1 varianty, nevybírá potají.
  - **Coding Agent** — AI agent (LangGraph orchestrace) implementující podle plánu (P-xxx/D-xxx). Běží lokálně, sleduje výkon.
  - **Orchestrator (CORE)** — centrální koordinační jádro (ne lidský actor, systémová role): routing, globální stav, event orchestration, lifecycle ganglií.

## 7. User Stories

- Stav: `[x] specifikováno`
- Obsah:
  - US-001: Jako uživatel chci vybrat lokální LLM model, aby AI odpovídala na mém hardwaru.
  - US-002: Jako uživatel chci vést rozhovor, ze kterého Specification Agent vytvoří specifikaci bez domýšlení.
  - US-003: Jako uživatel chci vidět plán kroků a po každém dvou mít commit/push.
  - US-004: Jako uživatel chci, aby Coding Agent implementoval krok a zapsal postup + selhání testů do deníku.
  - US-005: Jako uživatel chci RAG nad specifikací a deníkem (Vector Store) pro kontextové odpovědi.
  - US-006: Jako uživatel chci připojit své IDE (VS Code / Trae / Antigravity) a ovládat akce z aplikace.
  - US-007: Jako uživatel chci přepínat fáze (BRAINSTORMING → PLANNING → VÝVOJ) s ověřením podmínek přechodu.
  - US-008: Jako uživatel chci Edit Mode pro úpravu obsahu Views s řízeným ukládáním (autosave/explicit save, unsaved changes handling).
  - US-009: Jako uživatel chci vidět Matrix-style UI (černé pozadí, zelené prvky, synapse pozadí) v každém View.
  - US-010: Jako uživatel chci, aby MCP Server poskytoval skills pro coding agenta (čtení souborů, spouštění příkazů, retrieval).

## 8. Functional Requirements

- Stav: `[x] specifikováno`
- Obsah (každé s ID, popisem, prioritou P0/P1/P2):
  - FR-001 (P0): CORE koordinuje lifecycle ganglií (init/ready/error) a globální stav fáze (BRAINSTORMING/PLANNING/VÝVOJ/DEPLOY).
  - FR-002 (P0): State Machine Ganglion spravuje přechody fází s triggery, podmínkami, eventy a persistencí.
  - FR-003 (P0): LLM Manager načítá/přepíná modely z Ollama, LM Studio a OpenAI-compatible API; sleduje stav (INIT/LOADING/READY/ERROR).
  - FR-004 (P0): Conversation Ganglion uchovává zprávy, session, token tracking; persistuje do PostgreSQL.
  - FR-005 (P0): Vector Store (PGVector) ukládá embeddingy, poskytuje RAG retrieval a cache.
  - FR-006 (P0): MCP Server poskytuje tools/skills pro coding agenta se schématy a API kontraktem.
  - FR-007 (P1): IDE Bridge připojuje externí IDE přes WebSocket/LSP, přenáší akce a eventy.
  - FR-008 (P1): Developer Diary Ganglion zaznamenává postup, zdůvodnění a selhání testů; editovatelné záznamy.
  - FR-009 (P1): Application Shell obsahuje Header (L.O.N.G.I.N. + akronym, datum/čas, identita uživatele, Edit Mode) a sbratitelnou Left Navigation.
  - FR-010 (P1): Synapse Renderer (Canvas/WebGL) vykresluje pozadí (noční obloha hvězd tvořených synapsemi, občas vzruch).
  - FR-011 (P2): Každý View má unikátní ID (VIEW-XXX), route, layout, data contract, state model, flows, AC.

## 9. Non-functional Requirements

- Stav: `[x] specifikováno`
- Obsah:
  - NFR-001: Výkon optimalizovaný pro lokální hardware — LLM inference na GPU, cache v Vector Store.
  - NFR-002: Spolehlivost komunikace mezi lokálními kontejnery (healthchecky, retry).
  - NFR-003: Konzistence dat v PostgreSQL (transakce, integrita).
  - NFR-004: Latence UI — synapse renderer neblokuje interakci (requestAnimationFrame / offscreen canvas).
  - NFR-005: Single-user — žádná autentizační zátěž pro vzdálené uživatele.

## 10. Core Systems

- Stav: `[x] specifikováno`
- Obsah: Přehled všech GANGLIONů:
  - G-001: LLM Manager — správa lokálních (LM Studio, Ollama) a cloud modelů.
  - G-002: State Machine — přechody fází vývoje.
  - G-003: Conversation — rozhovor jako prostředek vývoje, záznam do deníku.
  - G-004: Vector Store — PGVector pro RAG, embedding, cache.
  - G-005: MCP Server — skills pro coding agenta.
  - G-006: IDE Bridge — most pro VS Code / Trae / Antigravity.
  - G-007: Developer Diary — záznam postupu, zdůvodnění, selhání testů.
  - **CORE** — centrální řídicí jádro (koordinace, globální stav, routing, event orchestration).

## 11. System Specifications

- Stav: `[ ] neúplné doplnit` — viz samostatné soubory S-003 až S-010
- Odkaz: `docs/specification/ganglion-*.md`

## 12. Data Model

- Stav: `[ ] neúplné doplnit` — detailní entity v S-003 (CORE) a per-ganglion (S-004..S-010)
- Obsah (přehled entit, detaily v per-ganglion specifikacích):
  - `Project` — id, name, current_phase, created_at, updated_at
  - `Decision` — id, code (DL-xxx), topic, context, chosen_variant, rationale, affected_systems, status
  - `Model` — id, provider (ollama/lm_studio/openai), model_name, state, loaded_at
  - `ConversationSession` — id, project_id, model_id, created_at
  - `Message` — id, session_id, role, content, token_count, created_at
  - `DiaryEntry` — id, step_id, content, timestamp, rationale
  - `Embedding` — id, source_type, source_id, vector, metadata
  - `Skill` (MCP) — id, name, schema, description
  - `IDEConnection` — id, ide_type, transport, state, connected_at

## 13. State Machines

- Stav: `[x] specifikováno` (přehled; detailní přechody v S-003 a per-ganglion)
- Obsah:
  - **SM-001: Stavový automat vývoje** (BRAINSTORMING → PLANNING → VÝVOJ → DEPLOY). Trigger: uživatel iniciovaný po ověření podmínek. Eventy: `phase.entered`, `phase.transition.requested`, `phase.transition.denied`. Persistence: `Project.current_phase`.
  - **SM-002: Načítání LLM modelu** (INIT → LOADING → READY → ERROR). Trigger: uživatel vybere model / přepnutí za běhu. Eventy: `model.load.started`, `model.ready`, `model.error`. Persistence: `Model.state`, `Model.loaded_at`.
  - **SM-003: View state machine** (UNINITIALIZED → LOADING → READY → EDIT/EMPTY/ERROR → SAVING → SAVED). Trigger: otevření View, akce uživatele. Eventy: `view.loading`, `view.ready`, `view.error`, `view.save.started`, `view.saved`. Persistence: per-View UI state (client) + data (server).
  - **SM-004: Edit Mode** (OFF → ON). Trigger: přepínač v Header. Eventy: `edit.enabled`, `edit.disabled`. Persistence: session UI state.
  - **SM-005: IDE Bridge connection** (DISCONNECTED → CONNECTING → CONNECTED → ERROR → DISCONNECTED). Trigger: connect/disconnect akce. Eventy: `ide.connecting`, `ide.connected`, `ide.disconnected`, `ide.error`. Persistence: `IDEConnection.state`.

## 14. Event Model

- Stav: `[ ] neúplné doplnit` — detaily v S-003 (CORE event orchestration)
- Obsah: Event ID, trigger, payload, producer, consumers, timing, persistence (viz SM-001..SM-005 pro výchozí sadu eventů)

## 15. Algorithms

- Stav: `[ ] neúplné doplnit` — detaily v S-004 (LLM), S-007 (Vector Store)
- Obsah: RAG retrieval, token management, context window management, synapse rendering (Canvas/WebGL)

## 16. UI/UX

- Stav: `[x] specifikováno` (přehled) — detaily v S-011, S-012, S-013
- Obsah:
  - Design system: Matrix theme — černé pozadí, zelené prvky, synapse pozadí, průhledné panely.
  - Application Shell: Header + Left Navigation (sbalitelná) + View Content.
  - Header obsahuje: identitu aplikace (L.O.N.G.I.N. EGO Systém + akronym DL-004), datum/čas, identitu uživatele, Edit Mode.
  - Views: VIEW-001 Dashboard, VIEW-002 Brainstorming, VIEW-003 Planning, VIEW-004 Development, VIEW-005 LLM Manager, VIEW-006 Models, VIEW-007 Diary, VIEW-008 MCP/Skills, VIEW-009 IDE Bridge, VIEW-010 Settings.

## 17. User Flows

- Stav: `[ ] neúplné doplnit` — detaily v S-012, S-013
- Obsah: Flow pro otevření View, načtení, editaci, uložení, chybu, navigaci (definováno v per-View specifikacích)

## 18. Economy / Rules / Progression

- Stav: `[x] N/A`
- Důvod: Nejedná se o hru ani ekonomický systém

## 19. AI

- Stav: `[ ] neúplné doplnit` — detaily v S-004, S-006, S-008
- Obsah: LangGraph orchestrace, agenti (specification, coding), kontextové okno, retrieval, tool calling

## 20. Networking

- Stav: `[x] specifikováno` (přehled) — detaily v S-014
- Obsah: Lokální Docker síť, komunikace mezi kontejnery:
  - backend ↔ postgres (PGVector)
  - backend ↔ ollama
  - backend ↔ lm-studio-proxy
  - frontend ↔ backend (REST/WebSocket)
  - mcp-server ↔ backend
  - ide-bridge ↔ externí IDE (WebSocket/LSP přes host)

## 21. Persistence

- Stav: `[x] specifikováno` (přehled)
- Obsah: PostgreSQL 16 + PGVector. Ukládá se: specifikace, Decision Log, deník, zprávy (Conversation), modely (stav), konfigurace, embeddingy (RAG/cache).

## 22. Configuration

- Stav: `[ ] neúplné doplnit` — detaily v S-014
- Obsah: `.env`, `docker-compose.yml`, konfigurace LLM providerů (Ollama/LM Studio/OpenAI endpointy, API klíče)

## 23. Error Handling

- Stav: `[ ] neúplné doplnit` — detaily v S-003 a per-ganglion
- Obsah: Chybové stavy pro každý GANGLION, retry strategie, fallback (např. model LOAD ERROR → UI ERROR state + retry)

## 24. Security

- Stav: `[x] specifikováno`
- Obsah: Lokální single-user — žádné vzdálené přihlášení. API klíče pro cloud modely šifrovaně (local secret store). Žádná data neopouštějí stroj kromě explicitních cloud LLM volání (uživatelem konfigurovaných).

## 25. Performance

- Stav: `[x] specifikováno` (přehled)
- Obsah: Optimalizace pro lokální hardware — GPU pro LLM inference, PGVector cache, neblokující synapse renderer.

## 26. Scalability

- Stav: `[x] N/A` pro masivní škálování
- Důvod: Single-user lokální aplikace
- Poznámka: Relevantní pouze škálování kontextového okna a databáze

## 27. Architecture

- Stav: `[x] specifikováno` (přehled) — detaily v S-003 (CORE), docs/architecture/
- Obsah: CORE/GANGLION/NEXUS/VIEW hierarchie (viz GLOSSARY.md), dependency graph. Backend (Python 3.11+ FastAPI + LangGraph + MCP SDK) + Frontend (React + Vite + TS) + PostgreSQL/PGVector + Ollama/LM Studio, vše v Docker Compose.

## 28. Dependencies

- Stav: `[x] specifikováno` (přehled)
- Obsah:
  - Python: fastapi, langgraph, mcp-sdk, psycopg, pgvector, openai, httpx
  - npm: react, vite, typescript, zustand, framer-motion
  - Docker images: postgres:16-pgvector, ollama, lm-studio (proxy), backend (python:3.11), frontend (node), mcp-server

## 29. Testing

- Stav: `[ ] neúplné doplnit` — detaily v P-014
- Obsah: Unit, integration, E2E, persistence, performance testy proti reálným testovacím datům

## 30. Acceptance Criteria

- Stav: `[ ] neúplné doplnit` — per-funkce v per-ganglion specifikacích (S-003..S-013)
- Obsah: AC objektivně měřitelná (viz Quality Gate checklist)

## 31. Implementation Plan

- Stav: `[x] specifikováno` — viz PLANNING fáze (P-001 až P-015)
- Odkaz: `docs/diary/DEVELOPER-DIARY.md`
- Pořadí dle DL-003: Specifikace první → Docker infra → UI

## 32. Implementation Tasks

- Stav: `[ ] neúplné doplnit` — viz PLANNING/VÝVOJ fáze
- Obsah: Task ID, cíl, kontext, dependencies, soubory, API změny, AC, testy, edge cases, DoD

## 33. Decision Log

- Stav: `[x] specifikováno` — viz DEVELOPER-DIARY.md (DL-001 až DL-004)
- Odkaz: `docs/diary/DEVELOPER-DIARY.md`

## 34. Open Questions

- Stav: `[x] specifikováno` (počáteční)
- Obsah: Detailní otázky budou vznikat v S-003..S-014 a řešeny dle bodu 7 instrukcí (3+1 varianty).

## 35. Risks

- Stav: `[x] specifikováno` (počáteční)
- Obsah:
  - R-001: Omezení kontextového okna lokálních modelů → mitigace: token management, RAG retrieval.
  - R-002: Výkon lokálního LLM na slabším GPU → mitigace: výběr modelu, cache.
  - R-003: Bootstrapping — aplikace vyvíjí sama sebe → mitigace: striktní specifikace před implementací (DL-003).
  - R-004: Konzistence Docker kontejnerů lokálně → mitigace: healthchecky, retry.

## 36. Future Extensions

- Stav: `[x] specifikováno`
- Obsah: Více agentů, více IDE, cloud sync (volitelné — mimo NFR).

## 37. Change Log

- Stav: `[x] specifikováno`
- Obsah:
  - v0.1 (2025-01-15) — Základní šablona vytvořena, DL-001 až DL-004 schváleny
  - v0.2 (2025-01-15) — Vyplněny sekce 1–4, 6–10, 13, 16, 20–21, 24–28, 31, 33–37 na základě DL-001..DL-004 a zadání. Stupeň 2 — DEFINED.

---

## QUALITY GATE CHECKLIST

Před označením jako COMPLETE (stupeň 7):

### GLOBAL SHELL
- [ ] Header definován?
- [ ] Navigation definována?
- [ ] Sbalení Navigation definováno?
- [ ] Edit Mode definován?
- [ ] User Identity definována?
- [ ] Čas definován?
- [ ] Datum definován?
- [ ] Identita aplikace definována?
- [x] Akronym definován? (DL-004)

### VIEWS
- [ ] Každý View má ID?
- [ ] Každý View má route?
- [ ] Každý View má parent Ganglion?
- [ ] Každý View má layout?
- [ ] Každý View má data contract?
- [ ] Každý View má state model?
- [ ] Každý View má flow?
- [ ] Každý View má permission model?
- [ ] Každý View má error state?
- [ ] Každý View má AC?

### DATA
- [ ] Víme odkud pochází každá zobrazená hodnota?
- [ ] Víme kdo ji vlastní?
- [ ] Víme kdo ji může změnit?
- [ ] Víme kdy se aktualizuje?

### INTERACTION
- [ ] Každé kliknutí definováno?
- [ ] Každá změna definována?
- [ ] Validace definována?
- [ ] Feedback definován?
- [ ] Chyba definována?
- [ ] Uložení definováno?
- [ ] Zrušení definováno?

### FLOWS
- [ ] Všechny hlavní User Flows existují?
- [ ] Alternativní cesty definovány?
- [ ] Failure paths definovány?
- [ ] Edge cases definovány?

---

## STUPNĚ DOKONČENÍ (aktuální stav)

| Stupeň | Název | Stav |
|--------|-------|------|
| 0 | RAW IDEA | [x] — základní šablona |
| 1 | ANALYZED | [x] — DL-001..DL-004, instrukce analyzovány |
| 2 | DEFINED | [x] — sekce 1–4, 6–10, 13, 16, 20–28, 31, 33–37 vyplněny |
| 3 | DESIGNED | [ ] — probíhá v S-003 (CORE) a per-ganglion |
| 4 | SPECIFIED | [ ] |
| 5 | IMPLEMENTATION READY | [ ] |
| 6 | VALIDATED | [ ] |
| 7 | COMPLETE | [ ] |

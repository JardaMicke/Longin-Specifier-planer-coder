# Vývojářský deník — Longin Coder

**Projekt:** L.O.N.G.I.N. Coder — interaktivní vývojářský deník s LLM agenty
**Akronym:** Logical Orchestrated Networked Generative Intelligent Nexus
**Stav:** FÁZE 1 — BRAINSTORMING / SPECIFIKACE

---

## KONTEXT

Tento deník zaznamenává každý krok vývoje aplikace "Longin Coder" — interaktivního vývojářského deníku, který sám o sobě slouží k vývoji softwaru pomocí rozhovoru s AI. Vývoj probíhá ve fázích řízených stavovým automatem:

1. **BRAINSTORMING** — specifikace nápadu, vyplnění kolonek, 3+1 varianty
2. **PLANNING** — krok za krokem plán implementace pro coding agenta
3. **VÝVOJ** — implementace s průběžným testováním a zápisem postupu

---

## SCHVÁLENÁ ROZHODNUTÍ (Decision Log)

### DL-001 — Backend stack
- **Datum:** 2025-01-15
- **Téma:** Technologický stack backendu (CORE)
- **Kontext:** Aplikace vyžaduje LangGraph orchestraci, MCP server, PGVector, Ollama/LM Studio integraci
- **Dostupné varianty:** (A) Python + FastAPI, (B) Python BE + React FE, (C) TypeScript E2E
- **Zvolená varianta:** B — Python BE + React FE
- **Důvod:** Nejlepší kombinace — Python pro LangGraph/MCP/LLM ekosystém, React pro bohaté interaktivní Matrix UI
- **Důsledky:** Dva stacky vyžadují Docker orchestraci; API kontrakt mezi BE a FE musí být striktně specifikován

- **Dotčené systémy:** CORE, všechny GANGLIONy, infrastruktura
- **Stav:** APPROVED

### DL-002 — Frontend technologie
- **Datum:** 2025-01-15
- **Téma:** Frontend framework pro Matrix-style UI
- **Kontext:** Požadováno černé pozadí, zelené prvky, synapse pozadí s Canvas/WebGL, průhledné panely
- **Dostupné varianty:** (A) React + Vite, (B) Vue 3 + Vite, (C) Next.js
- **Zvolená varianta:** A — React + Vite
- **Důvod:** Největší ekosystém komponent, reaktivní správa stavu, Canvas/WebGL podpora
- **Důsledky:** Potřeba Zustand/Redux pro stav, Framer Motion pro animace
- **Dotčené systémy:** Application Shell, všechny VIEWs, Synapse renderer
- **Stav:** APPROVED

### DL-003 — Pořadí vývoje
- **Datum:** 2025-01-15
- **Téma:** Sekvence vývojových fází
- **Kontext:** Návrh předepisuje BRAINSTORMING → PLANNING → VÝVOJ
- **Dostupné varianty:** (A) Specifikace první, (B) Docker infrastruktura, (C) UI prototyp
- **Zvolená varianta:** A — Specifikace první
- **Důvod:** Odpovídá zadání — bez kompletní specifikace nelze implementovat bez domýšlení
- **Důsledky:** Docker a UI až po dokončení specifikace; specifikace může být na více částí
- **Dotčené systémy:** Celý projekt
- **Stav:** APPROVED

### DL-004 — Akronym
- **Datum:** 2025-01-15
- **Téma:** Význam akronymu L.O.N.G.I.N.
- **Kontext:** Bod 44 instrukcí vyžaduje definici, nesmí se vymýšlet
- **Zvolená varianta:** Logical Orchestrated Networked Generative Intelligent Nexus
- **Stav:** APPROVED

---

## PLÁN VÝVOJE — NANO KROKY

Každý krok má: ID, cíl, kontext, závislosti, definition of done, stav.
Stav: [ ] nezačato / [~] probíhá / [x] hotovo / [!] blokováno

### FÁZE 1: BRAINSTORMING / SPECIFIKACE

- [x] **S-001** — Založení vývojářského deníku a Decision Logu
  - Cíl: Vytvořit deník, glossary, instrukce agenta jako referenční pravidla
  - Kontext: Bod 10, 33-37 instrukcí — dokumentace musí předcházet implementaci
  - Závislosti: žádné
  - DoD: docs/diary/, docs/instructions/, docs/specification/ existují
  - Test: adresářová struktura + deník obsahuje plán
  - Stav: HOTovo 2025-01-15

- [x] **S-002** — Páteřní specifikační dokument (brainstorming template)
  - Cíl: Vyplnit šablonu specifikace podle bodů 33-37 instrukcí (37 sekcí) na základě DL-001..DL-004 a zadání
  - Kontext: Šablona musí obsahovat kolonky pro veškeré specifikace — nic nesmí být domýšleno
  - Závislosti: S-001
  - DoD: docs/specification/specification-template.md obsahuje všechny 37 sekcí se stavem kolonek
  - Test: každá sekce má strukturu, edge cases, AC; stupeň dokumentu = DEFINED (2)
  - Stav: HOTovo 2026-08-13

- [x] **S-003** — Specifikace CORE — centrální řídicí jádro
  - Cíl: Detailně specifikovat CORE (globální stav, koordinace ganglií, routing, event orchestration)
  - Kontext: Bod 55-60, 69 instrukcí — responsibility matrix
  - Závislosti: S-002
  - DoD: CORE má data model, state machine, API, events, edge cases, AC
  - Test: lze podle specifikace implementovat bez domýšlení
  - Stav: HOTovo 2026-08-13

- [ ] **S-004** — Specifikace GANGLION: LLM Manager
  - Cíl: Specifikovat správu lokálních (LM Studio, Ollama) a cloud modelů
  - Kontext: Plné ovládání výběru a načítání modelů, tokenizéry, cache
  - Závislosti: S-002, S-003
  - DoD: Data model modelů, seznam API (Ollama, LM Studio, OpenAI-compatible), state machine načítání, edge cases
  - Test: AC pro načtení modelu, selhání, přepnutí
  - Stav: NEZAČATO

- [ ] **S-005** — Specifikace GANGLION: State Machine (Stavový automat)
  - Cíl: Specifikovat přechody BRAINSTORMING → PLANNING → VÝVOJ → (DEPLOY)
  - Kontext: Bod 13 instrukcí — state machine s přechody, podmínkami, eventy
  - Závislosti: S-002, S-003
  - DoD: Diagram stavů, přechody, trigger, podmínky, eventy, persistence
  - Test: AC pro každý přechod
  - Stav: NEZAČATO

- [ ] **S-006** — Specifikace GANGLION: Conversation/Chat
  - Cíl: Rozhovor jako prostředek vývoje, zápis do deníku
  - Kontext: LangGraph orchestrace, kontextové okno, token management
  - Závislosti: S-002, S-003, S-004
  - DoD: Data model zprávy, session, token tracking, persistence do PG/PGVector
  - Test: AC pro zprávu, kontext, retrieval
  - Stav: NEZAČATO

- [ ] **S-007** — Specifikace GANGLION: Vector Store & Knowledge Base
  - Cíl: PGVector pro RAG, embedding, cache
  - Kontext: Bod zadání — "endbardinky tokenizéry databáze postgres s PG vektor rozšířením ready pro cache"
  - Závislosti: S-002, S-003
  - DoD: Schema, embedding pipeline, retrieval API, cache strategie, edge cases
  - Test: AC pro insert, search, cache hit/miss
  - Stav: NEZAČATO

- [ ] **S-008** — Specifikace GANGLION: MCP Server
  - Cíl: MCP server se skills pro coding agenta
  - Kontext: Bod zadání — "vytvoř i mcp server a skilly pro agenta programátora"
  - Závislosti: S-002, S-003
  - DoD: Seznam tools, schemas, skill definice, API kontrakt
  - Test: AC pro volání tool, kontext retrieval
  - Stav: NEZAČATO

- [ ] **S-009** — Specifikace GANGLION: IDE Bridge
  - Cíl: Most pro VS Code / Trae / Antigravity
  - Kontext: Bod zadání — "Most pro IDE typu vs code nebo Trade nebo antigravity"
  - Závislosti: S-002, S-003
  - DoD: Protokol, transport (WebSocket/LSP), akce, eventy, edge cases
  - Test: AC pro připojení, akci, odpojení
  - Stav: NEZAČATO

- [ ] **S-010** — Specifikace GANGLION: Developer Diary
  - Cíl: Zápis postupu, zdůvodnění, selhání testů
  - Kontext: Bod zadání — "zaznamenávan do deníku ke každému kroku postup a zdůvodnění"
  - Závislosti: S-002, S-003
  - DoD: Data model záznamu, typy záznamů, API, persistence, edge cases
  - Test: AC pro vytvoření, editaci, retrieval
  - Stav: NEZAČATO

- [ ] **S-011** — Specifikace UI: Design System (Matrix theme)
  - Cíl: Barvy (černá/zelená), typografie, komponenty, synapse pozadí
  - Kontext: Bod 42 instrukcí — design system, bod zadání — Matrix vizualizace
  - Závislosti: S-002
  - DoD: Palette, typography, komponenty (Button, Input, Card...), synapse renderer spec
  - Test: AC pro každý stav komponenty
  - Stav: NEZAČATO

- [ ] **S-012** — Specifikace UI: Application Shell
  - Cíl: Header (L.O.N.G.I.N. + datum/čas + user + Edit Mode), Left Navigation (collapsible), View Content
  - Kontext: Bod 43-50 instrukcí
  - Závislosti: S-011
  - DoD: Layout, data contract, state machine, flows, AC
  - Test: AC viz bod 65
  - Stav: NEZAČATO

- [ ] **S-013** — Specifikace VIEWs (všechny view dle bodu 48, 64)
  - Cíl: VIEW-001 Dashboard, VIEW-002 Brainstorming, VIEW-003 Planning, VIEW-004 Development, VIEW-005 LLM Manager, VIEW-006 Models, VIEW-007 Diary, VIEW-008 MCP/Skills, VIEW-009 IDE Bridge, VIEW-010 Settings
  - Kontext: Bod 48, 64 instrukcí — šablona pro každý view
  - Závislosti: S-012
  - DoD: Každý view má ID, route, layout, data, states, actions, flows, AC
  - Test: AC viz bod 65, 75
  - Stav: NEZAČATO

- [ ] **S-014** — Specifikace infrastruktury (Docker)
  - Cíl: docker-compose, služby (postgres+pgvector, ollama, lm-studio-proxy, backend, frontend, mcp-server)
  - Kontext: Bod zadání — "vše poběží v docker kontejnerech"
  - Závislosti: S-003 až S-013
  - DoD: Schema, sítě, volumy, env, healthchecks
  - Test: AC pro startup, health, komunikaci
  - Stav: NEZAČATO

- [ ] **S-015** — Quality Gate specifikace
  - Cíl: Kontrola dle bodu 11, 34, 75 instrukcí
  - Kontext: Specifikace není COMPLETE dokud neprojde QG
  - Závislosti: S-002 až S-014
  - DoD: Quality Gate checklist vyplněný
  - Test: všechny odpovědi ANO
  - Stav: NEZAČATO

### FÁZE 2: PLANNING (detailní plán implementace)

- [ ] **P-001** — Plán infrastruktury (Docker) krok za krokem
- [ ] **P-002** — Plán backend CORE krok za krokem
- [ ] **P-003** — Plán GANGLION LLM Manager krok za krokem
- [ ] **P-004** — Plán GANGLION State Machine krok za krokem
- [ ] **P-005** — Plán GANGLION Conversation krok za krokem
- [ ] **P-006** — Plán GANGLION Vector Store krok za krokem
- [ ] **P-007** — Plán GANGLION MCP Server krok za krokem
- [ ] **P-008** — Plán GANGLION IDE Bridge krok za krokem
- [ ] **P-009** — Plán GANGLION Developer Diary krok za krokem
- [ ] **P-010** — Plán UI Design System krok za krokem
- [ ] **P-011** — Plán Application Shell krok za krokem
- [ ] **P-012** — Plán VIEWs krok za krokem
- [ ] **P-013** — Plán propojení BE-FE krok za krokem
- [ ] **P-014** — Plán testů (všechny druhy) krok za krokem
- [ ] **P-015** — Plán nasazení krok za krokem

### FÁZE 3: VÝVOJ

- [ ] **D-001** — Docker infrastruktura
- [ ] **D-002** — Backend CORE
- [ ] **D-003** — GANGLION LLM Manager
- [ ] **D-004** — GANGLION State Machine
- [ ] **D-005** — GANGLION Conversation
- [ ] **D-006** — GANGLION Vector Store
- [ ] **D-007** — GANGLION MCP Server
- [ ] **D-008** — GANGLION IDE Bridge
- [ ] **D-009** — GANGLION Developer Diary
- [ ] **D-010** — UI Design System
- [ ] **D-011** — Application Shell
- [ ] **D-012** — VIEWs
- [ ] **D-013** — Propojení BE-FE
- [ ] **D-014** — Testy
- [ ] **D-015** — Nasazení

---

## ZÁZNAMY KROKŮ

### 2025-01-15 — S-001: Založení deníku a Decision Logu

**Akce:**
- Analyzován návrh aplikace Longin Coder
- Identifikovány 4 klíčové nejasnosti (backend stack, frontend tech, pořadí, akronym)
- Položeny otázky uživateli (bod 7 — 3+1 varianty + doporučení)
- Prijata rozhodnutí DL-001 až DL-004
- Vytvořena adresářová struktura: docs/specification, docs/diary, docs/architecture, docs/instructions
- Vytvořen tento vývojářský deník s plánem (S-001 až S-015, P-001 až P-015, D-001 až D-015)

**Zdůvodnění:**
- Podle bodu 28 instrukcí — neřešit celý projekt v jedné odpovědi, rozdělit na fáze
- Podle bodu 10 — Decision Log pro každé rozhodnutí
- Podle bodu 33-37 — finální dokumentace musí mít 37 sekcí

**Stav:** HOTovo
**Kontrolní seznam:**
- [x] Logika dokončena — plán pokrývá BRAINSTORMING → PLANNING → VÝVOJ
- [x] Testováno proti reálným datům — repositář existuje, git na master, adresáře vytvořeny
- [x] Jak plánováno — deník, glossary, instrukce agenta
- [x] UI má ovládací prvky pro každou funkci — specifikováno v S-011 až S-013 (pending)

---

### 2026-08-13T20:10Z — S-002: Dokončení páteřního specifikačního dokumentu

**Akce:**
- Vyplněny sekce 1–4 (Executive Summary, Vision, Goals, Non-goals) na základě DL-001..DL-004 a zadání
- Vyplněna sekce 5 (Terminology — odkaz na GLOSSARY.md)
- Vyplněna sekce 6 (User Personas/Actors — Single User, Specification Agent, Coding Agent, Orchestrator/CORE)
- Vyplněna sekce 7 (User Stories US-001..US-010)
- Vyplněna sekce 8 (Functional Requirements FR-001..FR-011 s prioritami P0/P1/P2)
- Vyplněna sekce 9 (Non-functional Requirements NFR-001..NFR-005)
- Vyplněna sekce 10 (Core Systems — přehled 7 GANGLIONů + CORE)
- Vyplněna sekce 13 (State Machines SM-001..SM-005)
- Vyplněny sekce 16, 20–21, 24–28, 31, 33–37
- Aktualizován Quality Gate checklist (akronym checked dle DL-004)
- Stupeň dokumentu povýšen: RAW IDEA → DEFINED (stupeň 2)

**Zdůvodnění:**
- DL-003 (Specifikace první) určuje, že bez kompletní specifikace nelze implementovat — proto páteřní dokument vyplňován dříve než Docker/UI
- DL-004 (akronym) zanesen do Executive Summary, Vision a Quality Gate
- DL-001/DL-002 (stack) zaneseny do sekce 27 (Architecture) a 28 (Dependencies)
- Bod 11 instrukcí — Quality Gate: akronym (DL-004) je první checked kolonka

**Stav:** HOTovo
**Kontrolní seznam:**
- [x] Logika dokončena — 37 sekcí existuje, kolonky mají stav
- [x] Testováno proti reálným datům — soubor docs/specification/specification-template.md aktualizován, obsahuje 37 sekcí
- [x] Jak plánováno — stupeň DEFINED, DoD S-002 splněno
- [x] UI má ovládací prvky pro každou funkci — UI sekce (16) odkazuje na S-011..S-013 (pending)

---

### 2026-08-13T20:20Z — S-003: Specifikace CORE

**Akce:**
- Vytvořen soubor `docs/specification/ganglion-core.md` (specifikace CORE, stupeň 4 — SPECIFIED pro CORE)
- Sekce 1: Účel a rozsah CORE (koordinace, globální stav, event bus, API, health)
- Sekce 2: Responsibility Matrix (CORE vs Ganglion vs Nexus) — závazná
- Sekce 3: Data Model — 4 entity (`core.project`, `core.ganglion_registry`, `core.event_log`, `core.system_health`) s atributy, typy, validací, persistence
- Sekce 4: State Machines — SM-CORE-01 (Ganglion lifecycle), SM-CORE-02 (fáze reference, deleguje na G-002), SM-CORE-03 (Health check cyklus)
- Sekce 5: Event Model — event kontrakt, 9 CORE eventů, doručení (sync + persist + WS)
- Sekce 6: API kontrakt — REST (status, project, phase/transition, health, events) + WebSocket (/api/core/stream)
- Sekce 7: Edge Cases EC-001..EC-010 (postgres down, souběžné přechody, výjimka consumeru, WS odpojení, flap, DEPLOY projekt, retence event logu, ping timeout, dlouhá init, IDE odpojení)
- Sekce 8: Acceptance Criteria AC-001..AC-010 (objektivně měřitelné)
- Sekce 9: Implementation Tasks odkaz (IT-CORE-01..07, detaily v P-002)
- Sekce 10: DoD splněno

**Zdůvodnění:**
- DL-003 (Specifikace první) — CORE specifikován před implementací (D-002)
- DL-001 (Python BE + React FE) — API kontrakt (REST + WebSocket) pro React frontend
- Bod 5 instrukcí — Data Model, State Machine, UI/UX propojení; pro CORE specifikováno data model + state machine + API
- Bod 8 instrukcí — Edge cases povinně definovány (10 scénářů)
- Bod 9 instrukcí — traceability: US → FR → CORE → IT → AC (záznam v sekci 9)
- Bod 11 instrukcí — Quality Gate: lze podle CORE specifikace implementovat bez domýšlení (všechny rozhodné otázky odpovězeny DL-001..DL-004)
- MVC dodrženo — CORE = Controller/Model vrstva, Views v S-011..S-013, Ganglion = doménové moduly

**Stav:** HOTovo
**Kontrolní seznam:**
- [x] Logika dokončena — CORE má data model, state machine, API, events, edge cases, AC
- [x] Testováno proti reálným datům — soubor existuje, obsahuje 10 sekcí, DoD splněno; konzistence ověřena vůči GLOSSARY (terminologie CORE/Ganglion/Nexus/View) a vůči DL-001..DL-004
- [x] Jak plánováno — DoD S-003 splněno, stupeň SPECIFIED pro CORE
- [x] UI má ovládací prvky pro každou funkci — CORE API (REST/WS) poskytuje data pro Dashboard (VIEW-001), LLM Manager (VIEW-005), Diary (VIEW-007); UI kontroly (retry tlačítko při ERROR, phase transition button) specifikovány v API kontraktu a edge cases

---

# Páteřní specifikační dokument — L.O.N.G.I.N. Coder

> **Stav specifikace:** Stupeň 0 — RAW IDEA
> **Verze:** v0.1
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

- Stav: `[ ] specifikovat`
- Obsah: Jednoodstavcové shrnutí — co je L.O.N.G.I.N. Coder, k čemu slouží, kdo ho používá
- Kontext: Interaktivní vývojářský deník, rozhovor s AI, stavový automat, lokální single-user

## 2. Vision

- Stav: `[ ] specifikovat`
- Obsah: Dlouhodobá vize — software vyvíjený rozhovorem s AI, agenti běžící lokálně

## 3. Goals

- Stav: `[ ] specifikovat`
- Obsah: Měřitelné cíle (např. "uživatel vede rozhovor, AI generuje specifikaci a kód")

## 4. Non-goals

- Stav: `[ ] specifikovat`
- Obsah: Co aplikace NENÍ (multi-user, cloud-hosted SaaS, masivní škálování)

## 5. Terminology

- Stav: `[x] specifikováno` (viz GLOSSARY.md)
- Odkaz: `docs/specification/GLOSSARY.md`

## 6. User Personas / Actors

- Stav: `[ ] specifikovat`
- Obsah: 
  - **Single User** — lokální vývojář, jediný uživatel aplikace
  - **Specification Agent** — AI tvořící specifikaci
  - **Coding Agent** — AI implementující podle plánu
  - **Orchestrator (CORE)** — LangGraph koordinace

## 7. User Stories

- Stav: `[ ] specifikovat`
- Obsah: US-001 "Jako uživatel chci vybrat lokální LLM model..."
  - US-001 až US-0XX

## 8. Functional Requirements

- Stav: `[ ] specifikovat`
- Obsah: FR-001 až FR-0XX (každé s ID, popisem, prioritou)

## 9. Non-functional Requirements

- Stav: `[ ] specifikovat`
- Obsah: Výkon (lokální CPU/GPU/RAM), spolehlivost kontejnerů, konzistence dat

## 10. Core Systems

- Stav: `[ ] specifikovat`
- Obsah: Přehled všech GANGLIONů:
  - G-001: LLM Manager
  - G-002: State Machine
  - G-003: Conversation
  - G-004: Vector Store
  - G-005: MCP Server
  - G-006: IDE Bridge
  - G-007: Developer Diary

## 11. System Specifications

- Stav: `[ ] specifikovat` — viz samostatné soubory S-003 až S-010
- Odkaz: `docs/specification/ganglion-*.md`

## 12. Data Model

- Stav: `[ ] specifikovat`
- Obsah: Entity, atributy (ID, typ, default, validace, povinnost, zdroj, persistence)

## 13. State Machines

- Stav: `[ ] specifikovat`
- Obsah: 
  - SM-001: Stavový automat vývoje (BRAINSTORMING → PLANNING → VÝVOJ → DEPLOY)
  - SM-002: Načítání LLM modelu (INIT → LOADING → READY → ERROR)
  - SM-003: View state machine (UNINITIALIZED → LOADING → READY → EDIT/EMPTY/ERROR → SAVING → SAVED)

## 14. Event Model

- Stav: `[ ] specifikovat`
- Obsah: Event ID, trigger, payload, producer, consumers, timing, persistence

## 15. Algorithms

- Stav: `[ ] specifikovat`
- Obsah: RAG retrieval, token management, context window management, synapse rendering

## 16. UI/UX

- Stav: `[ ] specifikovat` — viz S-011, S-012, S-013
- Obsah: Design system (Matrix theme), Application Shell, Views

## 17. User Flows

- Stav: `[ ] specifikovat`
- Obsah: Flow pro otevření view, načtení, editaci, uložení, chybu, navigaci

## 18. Economy / Rules / Progression

- Stav: `[x] N/A`
- Důvod: Nejedná se o hru ani ekonomický systém

## 19. AI

- Stav: `[ ] specifikovat`
- Obsah: LangGraph orchestrace, agenty (specification, coding), kontextové okno, retrieval, tool calling

## 20. Networking

- Stav: `[ ] specifikovat`
- Obsah: Lokální Docker síť, komunikace mezi kontejnery (backend ↔ postgres, backend ↔ ollama, backend ↔ lm-studio, frontend ↔ backend, mcp ↔ backend, ide-bridge ↔ IDE)

## 21. Persistence

- Stav: `[ ] specifikovat`
- Obsah: PostgreSQL + PGVector, co se ukládá (specifikace, deník, zprávy, modely, konfigurace)

## 22. Configuration

- Stav: `[ ] specifikovat`
- Obsah: .env, docker-compose.yml, konfigurace LLM providerů

## 23. Error Handling

- Stav: `[ ] specifikovat`
- Obsah: Chybové stavy pro každý GANGLION, retry strategie, fallback

## 24. Security

- Stav: `[ ] specifikovat`
- Obsah: Lokální single-user — API klíče pro cloud modely (šifrované), žádné vzdálené přihlášení

## 25. Performance

- Stav: `[ ] specifikovat`
- Obsah: Optimalizace pro lokální hardware (GPU pro LLM inference, cache)

## 26. Scalability

- Stav: `[x] N/A` pro masivní škálování
- Důvod: Single-user lokální aplikace
- Poznámka: Relevatní pouze škálování kontextové okna a databáze

## 27. Architecture

- Stav: `[ ] specifikovat` — viz S-003 (CORE), docs/architecture/
- Obsah: CORE/GANGLION/NEXUS/VIEW hierarchie, dependency graph

## 28. Dependencies

- Stav: `[ ] specifikovat`
- Obsah: Python balíčky, npm balíčky, Docker images, externí služby

## 29. Testing

- Stav: `[ ] specifikovat`
- Obsah: Unit, integration, E2E, persistence, performance testy

## 30. Acceptance Criteria

- Stav: `[ ] specifikovat`
- Obsah: AC pro každou funkci (objektivně měřitelné)

## 31. Implementation Plan

- Stav: `[ ] specifikovat` — viz PLANNING fáze (P-001 až P-015)
- Odkaz: `docs/diary/DEVELOPER-DIARY.md`

## 32. Implementation Tasks

- Stav: `[ ] specifikovat`
- Obsah: Task ID, cíl, kontext, dependencies, soubory, API změny, AC, testy, edge cases, DoD

## 33. Decision Log

- Stav: `[x] specifikováno` — viz DEVELOPER-DIARY.md (DL-001 až DL-004)
- Odkaz: `docs/diary/DEVELOPER-DIARY.md`

## 34. Open Questions

- Stav: `[ ] specifikovat`
- Obsah: Seznam nevyřešených otázek (bude se plnit průběžně)

## 35. Risks

- Stav: `[ ] specifikovat`
- Obsah: Technologická rizika, kontextové okno, výkon lokálního LLM, bootstrapping

## 36. Future Extensions

- Stav: `[ ] specifikovat`
- Obsah: Více agentů, více IDE, cloud sync (volitelné)

## 37. Change Log

- Stav: `[x] specifikováno`
- Obsah:
  - v0.1 (2025-01-15) — Základní šablona vytvořena, DL-001 až DL-004 schváleny

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
- [ ] Datum definováno?
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
| 1 | ANALYZED | [ ] |
| 2 | DEFINED | [ ] |
| 3 | DESIGNED | [ ] |
| 4 | SPECIFIED | [ ] |
| 5 | IMPLEMENTATION READY | [ ] |
| 6 | VALIDATED | [ ] |
| 7 | COMPLETE | [ ] |

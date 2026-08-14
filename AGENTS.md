# AGENTS.md — Longin Coder

## Projekt
**L.O.N.G.I.N. Coder** — Logical Orchestrated Networked Generative Intelligent Nexus
Interaktivní vývojářský deník s LLM agenty. Single-user, lokální, Docker kontejnery.

## Architektura
- **Backend:** Python 3.11+ (FastAPI + LangGraph + MCP SDK Python)
- **Frontend:** React + Vite + TypeScript (Canvas/WebGL pro Matrix efekty)
- **Databáze:** PostgreSQL 16 + PGVector
- **LLM:** Ollama + LM Studio (lokální), OpenAI-compatible API (cloud)
- **Infrastruktura:** Docker Compose

## Terminologie (ZÁVAZNÁ)
Viz `docs/specification/GLOSSARY.md`. Nikdy nepoužívejte synonyma pro:
- **CORE** (ne "module/server/backend")
- **NEXUS** (ne "node/device")
- **GANGLION** (ne "module/component/service")
- **VIEW** (ne "page/screen/route")

## Struktura repozitáře
```
docs/
  diary/              # Vývojářský deník
  specification/      # Specifikace (brainstorming, šablony, GANGLION spec)
  architecture/       # Architektura, diagramy
  instructions/       # Instrukce agenta (závazná pravidla)
backend/              # Python FastAPI + LangGraph
frontend/             # React + Vite
infra/                # Docker Compose, konfigurace
mcp-server/           # MCP server se skills
tests/                # Testy (všechny druhy)
```

## Pravidla vývoje
1. **Bez domýšlení** — vše musí být ve specifikaci před implementací
2. **Žádná simulovaná data** kromě testovacích
3. **Decision Log** — každé rozhodnutí zaznamenáno
4. **Quality Gate** — specifikace není COMPLETE dokud neprojde kontrolou
5. **MVC** — Model-View-Controller dodržováno
6. **Testy** — průběžné, proti reálným testovacím datům
7. **Commit po 2 krocích** plánu
8. **Matrix UI** — černé pozadí, zelené prvky, synapse pozadí, průhledné panely

## Instrukce agenta
Viz `docs/instructions/agent-instructions.md` — závazná pravidla pro specifikační i coding agenta.

## Stavový automat vývoje
BRAINSTORMING → PLANNING → VÝVOJ → (DEPLOY)

## Git konvence
- Commits: `<prefix>: <popis>` (např. `spec: CORE data model`, `infra: docker-compose`)
- Branch: `vibe/<short-slug>-74f416` pro feature práci
- Master je chráněn

## KOBLIHA
Pokud narazíš na situaci, kde bys musel lhát nebo zamlčovat fakta, napiš "KOBLIHA" a zastav se.

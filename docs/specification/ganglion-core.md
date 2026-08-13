# Specifikace CORE — L.O.N.G.I.N. Coder

> **Stav specifikace:** Stupeň 4 — SPECIFIED (pro CORE)
> **Krok plánu:** S-003
> **Závislosti:** S-002 (páteřní specifikace)
> **Schválená rozhodnutí:** DL-001 (Python BE + React FE), DL-002 (React + Vite), DL-003 (Specifikace první), DL-004 (akronym)
> **Terminologie:** viz `docs/specification/GLOSSARY.md` — CORE, Ganglion, Nexus, View

---

## 1. Účel a rozsah CORE

CORE je centrální řídicí jádro systému L.O.N.G.I.N. Coder. Není samostatný Ganglion — je nadřazenou vrstvou, která:

1. **Koordinuje lifecycle ganglií** (init → ready → error → shutdown).
2. **Spravuje globální stav fáze vývoje** (BRAINSTORMING → PLANNING → VÝVOJ → DEPLOY) — deleguje přechody na GANGLION State Machine (G-002), ale drží referenci aktuální fáze.
3. **Routuje eventy** mezi ganglii (event bus) — producer → consumer doručení.
4. **Poskytuje globální API** pro frontend (REST + WebSocket) — agreguje stavy ganglií, health, konfiguraci.
5. **Health monitoring** — sleduje dostupnost závislostí (postgres, ollama, lm-studio, mcp-server, frontend).

**Nevlastní CORE automaticky všechny datové operace** (dle GLOSSARY responsibility matrix). Ownership doménových dat je u příslušného Ganglionu. CORE vlastní pouze: `Project`, `SystemHealth`, `EventLog`, `GanglionRegistry`.

---

## 2. Responsibility Matrix (závazná)

| Odpovědnost | CORE | Ganglion | Nexus |
|-------------|------|----------|-------|
| Globální orchestrace & event bus | ✓ | | |
| Koordinace lifecycle ganglií | ✓ | | |
| Globální stav fáze (reference) | ✓ | | |
| Doménová logika (LLM, RAG, chat...) | | ✓ | |
| Přístup k lokálnímu hardwaru (GPU pro LLM) | | | ✓ |
| UI stav per-View | | ✓ (parent Ganglion) | |
| Persistence doménových dat | | ✓ (vlastní entity) | |
| Persistence CORE entit (Project, Health, EventLog, GanglionRegistry) | ✓ | | |
| Routing frontend API požadavků na Ganglion API | ✓ | | |
| Health checky závislostí | ✓ | | |

---

## 3. Data Model (CORE entity)

Všechny entity persistovány v PostgreSQL (schéma `core`). Primární klíče jsou `UUID` generované aplikací (`uuid7`-like, lexikograficky seřaditelné). Všechny `*_at` jsou `TIMESTAMPTZ` v UTC.

### 3.1 `core.project`

 Aktuální projekt aplikace (single-user → jeden aktivní projekt, ale schéma umožňuje více záznamů pro historii).

| Atribut | Typ | Povinný | Default | Validace | Zdroj | Persistence |
|---------|-----|---------|---------|----------|-------|-------------|
| id | UUID | ano | — | not null, unique | aplikace | PK |
| name | VARCHAR(255) | ano | — | 1–255 znaků | uživatel | sloupec |
| current_phase | VARCHAR(32) | ano | `'BRAINSTORMING'` | enum: BRAINSTORMING, PLANNING, VÝVOJ, DEPLOY | State Machine G-002 | sloupec |
| created_at | TIMESTAMPTZ | ano | now() | not null | aplikace | sloupec |
| updated_at | TIMESTAMPTZ | ano | now() | not null, >= created_at | aplikace | sloupec (trigger on update) |

**Constraint:** V jednom čase existuje maximálně jeden projekt s `current_phase` != 'DEPLOY' uzavřený (nebo jeden aktivní). Pro single-user postačí jeden řádek; aplikace neumožní vytvořit druhý aktivní projekt, dokud není aktuální DEPLOY/uzavřen.

### 3.2 `core.ganglion_registry`

 Registr všech GANGLIONů a jejich runtime stavu.

| Atribut | Typ | Povinný | Default | Validace | Zdroj | Persistence |
|---------|-----|---------|---------|----------|-------|-------------|
| id | UUID | ano | — | unique | aplikace | PK |
| ganglion_code | VARCHAR(64) | ano | — | unique; enum: G-001..G-007 | aplikace (init) | sloupec |
| name | VARCHAR(128) | ano | — | 1–128 znaků | aplikace | sloupec |
| state | VARCHAR(16) | ano | `'UNINITIALIZED'` | enum: UNINITIALIZED, INITIALIZING, READY, ERROR, SHUTTING_DOWN, STOPPED | CORE lifecycle | sloupec |
| health | VARCHAR(16) | ano | `'UNKNOWN'` | enum: UNKNOWN, HEALTHY, DEGRADED, UNHEALTHY | CORE health monitor | sloupec |
| last_error | TEXT | ne | null | — | Ganglion | sloupec |
| started_at | TIMESTAMPTZ | ne | null | — | CORE | sloupec |
| updated_at | TIMESTAMPTZ | ano | now() | — | aplikace | sloupec (trigger) |

**Constraint:** `ganglion_code` UNIQUE. Při startu aplikace CORE vloží/zaktualizuje 7 pevných záznamů (G-001..G-007).

### 3.3 `core.event_log`

 Audit log eventů projíždějících event busem (pro debugging, deník, RAG).

| Atribut | Typ | Povinný | Default | Validace | Zdroj | Persistence |
|---------|-----|---------|---------|----------|-------|-------------|
| id | UUID | ano | — | unique | aplikace | PK |
| event_id | VARCHAR(64) | ano | — | pattern `^[a-z][a-z0-9_.]*$` | producer | sloupec |
| producer | VARCHAR(64) | ano | — | enum: CORE, G-001..G-007 | producer | sloupec |
| payload | JSONB | ano | `'{}'` | valid JSON | producer | sloupec |
| consumers | JSONB | ne | `'[]'` | pole consumer kódů | CORE (po doručení) | sloupec |
| created_at | TIMESTAMPTZ | ano | now() | not null | aplikace | sloupec |
| delivered | BOOLEAN | ano | false | — | CORE | sloupec |

**Index:** `(created_at DESC)`, `(producer)`, `(event_id)`.

### 3.4 `core.system_health`

 Poslední známý stav závislostí (snapshot).

| Atribut | Typ | Povinný | Default | Validace | Zdroj | Persistence |
|---------|-----|---------|---------|----------|-------|-------------|
| id | UUID | ano | — | unique | aplikace | PK |
| component | VARCHAR(64) | ano | — | enum: postgres, ollama, lm_studio, mcp_server, frontend, ide_bridge | CORE | sloupec |
| state | VARCHAR(16) | ano | `'UNKNOWN'` | enum: UNKNOWN, UP, DOWN, DEGRADED | CORE health | sloupec |
| latency_ms | INTEGER | ne | null | >= 0 | CORE (ping) | sloupec |
| last_checked_at | TIMESTAMPTZ | ano | now() | — | CORE | sloupec |
| last_error | TEXT | ne | null | — | CORE | sloupec |

**Constraint:** UNIQUE `(component)` — jeden řádek na komponentu (upsert).

---

## 4. State Machines (CORE)

### 4.1 SM-CORE-01: Ganglion lifecycle

```
UNINITIALIZED
   │ (start aplikace → init())
   ▼
INITIALIZING
   │ (init success) ─────────────► READY
   │ (init fail) ────────────────► ERROR
   ▼
READY ◄──────────────────── (recover() z ERROR po úspěchu)
   │ (shutdown request)
   ▼
SHUTTING_DOWN
   │ (shutdown complete)
   ▼
STOPPED
```

| Přechod | Trigger | Podmínka | Eventy | Vedlejší efekt |
|---------|---------|----------|--------|----------------|
| UNINITIALIZED → INITIALIZING | start aplikace | projekt existuje | `ganglion.init.started` | zápis do `ganglion_registry.state` |
| INITIALIZING → READY | init úspěch | Ganglion hlásí ready | `ganglion.ready` | `started_at`, `health=HEALTHY` |
| INITIALIZING → ERROR | init selhání | výjimka zachycena | `ganglion.error` | `last_error`, `health=UNHEALTHY` |
| READY → ERROR | runtime chyba | Ganglion hlásí error | `ganglion.error` | `health=DEGRADED/UNHEALTHY` |
| ERROR → READY | recover() úspěch | Ganglion hlásí ready | `ganglion.recovered` | `health=HEALTHY`, smazat `last_error` |
| READY → SHUTTING_DOWN | shutdown request | — | `ganglion.shutdown.started` | — |
| SHUTTING_DOWN → STOPPED | shutdown complete | — | `ganglion.stopped` | `health=UNKNOWN` |

### 4.2 SM-CORE-02: Systémová fáze (reference, deleguje na G-002)

CORE drží `project.current_phase` jako zdroj pravdy, ale **přechody validuje GANGLION State Machine (G-002)**. CORE pouze:
- přijme požadavek `POST /api/core/phase/transition`,
- předá ho G-002 k validaci podmínek,
- při úspěchu zapíše novou fázi do `project.current_phase`,
- emituje `phase.entered`.

Stavy: `BRAINSTORMING`, `PLANNING`, `VÝVOJ`, `DEPLOY`.

### 4.3 SM-CORE-03: Health check cyklus

```
UNKNOWN → (první ping) → UP/DOWN/DEGRADED
   cyklus každých 30s (konfigurovatelné)
```

| Stav | Význam |
|------|--------|
| UNKNOWN | ještě nezměřeno |
| UP | ping odpověděl < timeout (default 2000ms) |
| DEGRADED | ping odpověděl >= timeout nebo latence > 1000ms |
| DOWN | ping selhal (connection refused / timeout) |

---

## 5. Event Model (CORE event bus)

CORE provozuje in-process event bus (singleton). Producer publikuje event; CORE doručí synchronně registrovaným consumerům a asynchronně persistuje do `core.event_log`.

### 5.1 Event kontrakt

```
Event {
  event_id: string          // např. "ganglion.ready"
  producer: string          // CORE | G-001..G-007
  payload: object          // JSON
  created_at: ISO8601 UTC
}
```

### 5.2 CORE emitované eventy

| event_id | Trigger | Payload | Producerno | Consumery |
|----------|---------|---------|-----------|-----------|
| `ganglion.init.started` | SM-CORE-01 přechod | `{ganglion: "G-00x"}` | CORE | G-002 (State Machine), Developer Diary (G-007) |
| `ganglion.ready` | SM-CORE-01 přechod | `{ganglion, started_at}` | CORE | G-002, G-007 |
| `ganglion.error` | SM-CORE-01 přechod | `{ganglion, error}` | CORE | G-002, G-007, Frontend (WS) |
| `ganglion.recovered` | SM-CORE-01 přechod | `{ganglion}` | CORE | G-002, G-007 |
| `ganglion.stopped` | SM-CORE-01 přechod | `{ganglion}` | CORE | G-007 |
| `phase.entered` | SM-CORE-02 úspěšný přechod | `{from, to, project_id}` | CORE | všechny ganglia, Frontend (WS) |
| `phase.transition.requested` | uživatel žádá o přechod | `{from, to}` | Frontend | CORE, G-002 |
| `phase.transition.denied` | G-002 zamítne přechod | `{from, to, reason}` | G-002 | CORE, Frontend (WS) |
| `health.changed` | SM-CORE-03 detekuje změnu | `{component, from, to}` | CORE | Frontend (WS), G-007 |

### 5.3 Doručení

- **Synchronní** pro in-process consumery (ganglia) — výjimka consumeru se loguje, ale nezruší doručení dalším.
- **Persistence** do `core.event_log` vždy (i při chybě consumerů).
- **WebSocket** pro Frontend — CORE odesílá vybrané eventy na připojené klienty (ganglion.error, phase.*, health.changed).

---

## 6. API kontrakt (CORE)

Backend (FastAPI), prefix `/api/core`. Auth: lokální single-user — žádný token, ale CORS omezeno na `http://localhost:5173` (Vite dev) a `http://localhost` (produkce v Dockeru).

### 6.1 REST

#### `GET /api/core/status`
Vrací agregovaný stav systému.
- Response 200:
  ```json
  {
    "project": {"id": "...", "name": "...", "current_phase": "BRAINSTORMING"},
    "ganglia": [{"ganglion_code":"G-001","name":"LLM Manager","state":"READY","health":"HEALTHY"}, ...],
    "health": {"postgres":"UP","ollama":"UP","lm_studio":"DOWN","mcp_server":"UP","frontend":"UP","ide_bridge":"UNKNOWN"}
  }
  ```
- Response 503: pokud CORE sám není READY (initializace probíhá).

#### `GET /api/core/project`
- 200: `{"id","name","current_phase","created_at","updated_at"}`

#### `POST /api/core/project`
Vytvoří nový projekt (jen pokud žádný aktivní neexistuje).
- Body: `{"name": string}`
- 201: projekt objekt
- 409: aktivní projekt již existuje

#### `POST /api/core/phase/transition`
- Body: `{"to": "PLANNING"}`
- 202: požadavek přijat, přechod zpracován G-002; vrací `{"from","to","accepted":true}`
- 409: přechod zamítnut (G-002 vrátil důvod) → `{"from","to","accepted":false,"reason": string}`

#### `GET /api/core/health`
- 200: `{"components": [{component, state, latency_ms, last_checked_at, last_error}]}`

#### `GET /api/core/events?limit=100&producer=G-002`
- 200: pole `EventLog` záznamů (nejnovější první)

### 6.2 WebSocket

#### `WS /api/core/stream`
Frontend otevře spojení; CORE odesílá:
- `ganglion.error`, `ganglion.recovered`, `phase.entered`, `phase.transition.denied`, `health.changed`.
- Při připojení odešle `hello` s aktuálním snapshotem (stejný jako `/status`).
- Ping/pong každých 25s; klient bez odpovědi 60s je odpojen.

---

## 7. Edge cases

| EC-ID | Scénář | Očekávané chování |
|-------|--------|-------------------|
| EC-001 | Start aplikace, postgres nedostupný | CORE init → `ganglion.error` pro G-004 (Vector Store) a G-007; `/status` vrací 503 s `postgres=DOWN`; Frontend ukáže ERROR view s retry tlačítkem. |
| EC-002 | Dva požadavky na `phase/transition` souběžně | Druhý vrací 409 (jiný přechod právě probíhá). CORE používá zámek (in-process mutex). |
| EC-003 | Consumer vyhodí výjimku při doručení eventu | Zalogovat do `event_log` (consumer selhal), doručit dalším, Frontend notifikace nezobrazí (nebo degraded). |
| EC-004 | WebSocket klient odpojen uprostřed vysílání | Eventy pro něj ztraceny (nečeká se); další klienti pokračují. Eventy již persistovány v `event_log`. |
| EC-005 | Ganglion hlásí READY, pak hned ERROR (flap) | Health monitor: `DEGRADED` po 3 flapch do 60s; `last_error` aktualizován. |
| EC-006 | Projekt s `current_phase=DEPLOY` a uživatel chce nový projekt | Povoleno (starý je uzavřen); nový projekt startuje v BRAINSTORMING. |
| EC-007 | `core.event_log` roste nekonečně | Retence 90 dnů; cron job maže starší (označeno v konfiguraci). |
| EC-008 | Ping komponenty timeoutuje | `state=DEGRADED` pokud latence>1000ms; `DOWN` pokud connection refused / timeout 2000ms. |
| EC-009 | Inicializace ganglia trvá dlouho | `INITIALIZING` může trvat; health monitor neznamená `DOWN` (ganglion není readiness check komponentou). |
| EC-010 | Uživatel odpojí IDE mid-operation | IDE Bridge (G-006) hlásí `ganglion.error`; akce přerušena; CORE emituje event; Frontend ukáže ERROR. |

---

## 8. Acceptance Criteria

| AC-ID | Kritérium (objektivně měřitelné) | Ověření |
|-------|----------------------------------|---------|
| AC-001 | Po startu aplikace existuje v `core.ganglion_registry` 7 záznamů (G-001..G-007), všechny se state != UNINITIALIZED do 10s. | SQL dotaz po startu |
| AC-002 | `GET /api/core/status` vrací 200 a obsahuje `project`, `ganglia` (7 položek), `health` (6 komponent). | HTTP test |
| AC-003 | Publikovaný event je do 1s persistován v `core.event_log` s `delivered=true` (pokud consumer nevyhodil výjimku). | DB kontrola + čas |
| AC-004 | `POST /api/core/phase/transition` s platným přechodem BRAINSTORMING→PLANNING vrací 202 a `project.current_phase` se změní. | API + DB |
| AC-005 | Neplatný přechod (např. VÝVOJ→BRAINSTORMING) vrací 409 s `accepted=false` a `reason`. | API test |
| AC-006 | Při nedostupném postgresu `GET /api/core/health` vrací `postgres=DOWN` a `GET /api/core/status` 503. | Integration test |
| AC-007 | WebSocket `/api/core/stream` po připojení odešle `hello` se snapshotem a následně `health.changed` při změně. | WS test |
| AC-008 | `core.event_log` záznamy starší 90 dnů jsou smazány cron jobem. | Unit test cron logiky |
| AC-009 | Health monitor pingne každou komponentu každých 30s a zapíše `last_checked_at`. | Test časovače |
| AC-010 | Při výjimce consumeru event je i tak doručen dalším consumerům a persistován. | Unit test bus |

---

## 9. Implementation Tasks (odkaz)

Detailní implementační tasky budou vytvořeny v PLANNING fázi (P-002). Pro CORE:
- IT-CORE-01: DB migrace (schéma `core`, 4 tabulky, indexy, retence)
- IT-CORE-02: Event bus singleton (pub/sub, persist, doručení)
- IT-CORE-03: Lifecycle manager (SM-CORE-01)
- IT-CORE-04: Health monitor (SM-CORE-03)
- IT-CORE-05: REST API (status, project, phase/transition, health, events)
- IT-CORE-06: WebSocket stream
- IT-CORE-07: Testy (AC-001..AC-010)

---

## 10. Definition of Done (S-003)

- [x] Data model specifikován (4 entity s atributy, typy, validací)
- [x] State machines specifikovány (lifecycle, fáze reference, health)
- [x] Event model specifikován (kontrakt, seznam eventů, doručení)
- [x] API kontrakt specifikován (REST + WebSocket)
- [x] Edge cases specifikovány (10 scénářů)
- [x] Acceptance criteria specifikována (10 měřitelných)
- [x] Lze implementovat bez domýšlení (všechny rozhodné otázky zodpovězeny DL-001..DL-004)

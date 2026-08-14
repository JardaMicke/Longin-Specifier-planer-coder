# Specifikace GANGLION: State Machine (G-002)

> **Stav specifikace:** Stupeň 4 — SPECIFIED
> **Krok plánu:** S-005
> **Ganglion:** G-002 — State Machine
> **Závislosti:** S-002 (páteřní specifikace), S-003 (CORE — SM-CORE-02)
> **Schválená rozhodnutí:** DL-001 (Python BE + React FE), DL-003 (Specifikace první), DL-004 (akronym)
> **Terminologie:** viz `docs/specification/GLOSSARY.md` — State Machine, CORE, Ganglion, View
> **Views:** VIEW-002 Brainstorming, VIEW-003 Planning, VIEW-004 Development (parent Views pro fáze)
> **Konzistence s CORE:** CORE drží `project.current_phase` (zdroj pravdy), G-002 **validuje přechody** (SM-CORE-02 deleguje na G-002)

---

## 1. Účel a rozsah

GANGLION State Machine (G-002) je vlastníkem logiky přechodů mezi fázemi vývoje v systému L.O.N.G.I.N. Coder. Poskytuje:

1. **Definici stavů fází** — BRAINSTORMING, PLANNING, VÝVOJ, DEPLOY.
2. **Validaci přechodů** — pro každý přechod kontroluje podmínky (gates) a vrací accepted/denied + reason.
3. **Persistence přechodů** — záznam historie přechodů (audit trail) + aktivace/výběr fáze.
4. **Event emitování** — `phase.transition.requested`, `phase.transition.denied`, `phase.transition.accepted` (po validaci, před zápisem do `project.current_phase`).
5. **Gate evaluaci** — pro každý přechod existuje sada podmínek (Definition of Ready fází).

**Rozdělení zodpovědnosti s CORE (SM-CORE-02):**
- CORE přijme `POST /api/core/phase/transition`, předá požadavek G-002 metodou `validate_transition(from, to)`.
- G-002 vyhodnotí gaty, vrátí `accepted: bool` + `reason` (pokud denied) + `accepted_at`.
- Při `accepted=true` CORE zapíše novou fázi do `project.current_phase` a emituje `phase.entered`.
- Při `accepted=false` CORE emituje `phase.transition.denied` (producer G-002).
- G-002 sám **nezapisuje** `project.current_phase` (zdroj pravdy drží CORE); G-002 zapisuje pouze vlastní tabulky historie a gate stavů.

**Nevlastní G-002:** Obsah specifikace/plánu/kódu (to vlastní příslušné Gangliony a CORE persistence). G-002 vlastní pouze metadata přechodů a výsledky gate evaluace.

---

## 2. Responsibility Matrix (G-002)

| Odpovědnost | CORE | G-002 State Machine | Jiný Ganglion |
|-------------|------|---------------------|--------------|
| Zdroj pravdy `current_phase` | ✓ (`core.project`) | | |
| Validace přechodu (gaty) | | ✓ | |
| Zápis nové fáze do DB | ✓ (po accepted) | | |
| Historie přechodů (audit) | | ✓ (`sm.transition_log`) | |
| Gate stav (splněno/nesplněno) | | ✓ (`sm.gate_status`) | |
| Emit `phase.entered` | ✓ | | |
| Emit `phase.transition.denied` | | ✓ (přes CORE bus) | |
| Obsah specifikace (S-xxx) | | | G-007 (Diary) + persistence |
| Obsah plánu (P-xxx) | | | G-007 + persistence |
| Obsah kódu (D-xxx) | | | coding agent + persistence |

---

## 3. Stavy a přechody (SM-G02-01)

### 3.1 Stavy

| Stav | Popis | Defaultní View |
|------|-------|----------------|
| `BRAINSTORMING` | Specifikace nápadu, vyplňování kolonek, 3+1 varianty. Výchozí stav nového projektu. | VIEW-002 Brainstorming |
| `PLANNING` | Detailní krok-za-krokem plán implementace (P-001..P-015). | VIEW-003 Planning |
| `VÝVOJ` | Implementace s průběžným testováním (D-001..D-015). | VIEW-004 Development |
| `DEPLOY` | Projekt uzavřen/nasazen. Projekt nelze dále modifikovat (jen archivovat). | — (read-only archiv) |

### 3.2 Diagram

```
        ┌─────────────────────────────────────────────┐
        │                                             ▼
BRAINSTORMING ──(gate B→P)──► PLANNING ──(gate P→V)──► VÝVOJ ──(gate V→D)──► DEPLOY
        ▲                          │                   │                  │
        │                          │                   │                  │
        └──(rollback P→B, podmínka)┘                   │                  │
                                   └──(rollback V→P, podmínka)──┘         │
                                                                        │
   DEPLOY ──(nelze návrat; nový projekt start v BRAINSTORMING)───────────┘
```

### 3.3 Povolené přechody (matice)

| Z \ Do | BRAINSTORMING | PLANNING | VÝVOJ | DEPLOY |
|--------|---------------|----------|-------|--------|
| BRAINSTORMING | — | ✓ (gate B→P) | ✗ | ✗ |
| PLANNING | ✓ (rollback, gate P→B) | — | ✓ (gate P→V) | ✗ |
| VÝVOJ | ✗ | ✓ (rollback, gate V→P) | — | ✓ (gate V→D) |
| DEPLOY | ✗ (nový projekt = nový záznam) | ✗ | ✗ | — |

Všechny nepovolené přechody vrací `accepted=false`, `reason="invalid_transition"`.

### 3.4 Brány (gates) — podmínky přechodu

Každý gate je sada kontrol. Všechny musí být splněny (AND). Výsledek evaluace se persistuje v `sm.gate_status`.

#### Gate B→P (BRAINSTORMING → PLANNING)
- **G-BP-01:** Páteřní specifikace má stupeň ≥ DEFINED (2) — `specification-template.md` sekce 1–4, 6–10 označeny `[x]`.
- **G-BP-02:** Existuje alespoň 1 schválené rozhodnutí v Decision Logu (DL-xxx stav APPROVED).
- **G-BP-03:** Všechny plánované GANGLIONy (G-001..G-007) mají alespoň stupeň DEFINED (2) v per-ganglion specifikaci — nebo jsou označeny jako N/A s důvodem.
- **G-BP-04:** Quality Gate checklist „Akronym definován?" = ANO (DL-004).
- **G-BP-05:** Neexistuje nevyřešená OPEN DECISION s blokujícím příznakem.

#### Gate P→V (PLANNING → VÝVOJ)
- **G-PV-01:** Existuje plán implementace (P-001..P-015) s alespoň pro P-001 (Docker infra) a P-002 (CORE) stavem ≠ NEZAČATO.
- **G-PV-02:** Docker infrastruktura (S-014) je specifikována (stupeň ≥ SPECIFIED, 4).
- **G-PV-03:** Všechny P0 functional requirements (FR-001..FR-006) mají implementační task (IT-xxx) s přiřazenými soubory.
- **G-PV-04:** Alespoň 1 lokální LLM model je registrován a READY (přes G-001) — nebo cloud model je konfigurován.
- **G-PV-05:** Postgres + PGVector kontejner je health-checked UP (přes CORE health monitor).

#### Gate V→D (VÝVOJ → DEPLOY)
- **G-VD-01:** Všechny P0 acceptance criteria (AC-001..AC-010 pro CORE, AC-LLM-01..14 pro LLM Manager) prošly testem.
- **G-VD-02:** Aplikace startuje a `GET /api/core/status` vrací 200 se všemi 7 ganglii READY.
- **G-VD-03:** Všechny GANGLIONy (G-001..G-007) mají `ganglion_registry.state=READY` nebo explicitně označeny N/A.
- **G-VD-04:** Decision Log nemá nevyřešená PROPOSED rozhodnutí blokující DEPLOY.
- **G-VD-05:** Developer Diary (G-007) obsahuje záznam dokončení pro všechny D-xxx kroky (nebo jsou označeny N/A).

#### Rollback P→B (PLANNING → BRAINSTORMING)
- **G-PB-01:** Uživatel explicitně potvrdil rollback (UI confirm dialog).
- **G-PB-02:** Neexistuje rozpracovaný implementační task (IT-xxx) ve stavu IN_PROGRESS (vše IN_PROGRESS musí být ukončeno — completed nebo cancelled).
- **G-PB-03:** Rollback zaznamená Change Impact Analysis do Decision Logu (nový DL-xxx, typ ROLLBACK).

#### Rollback V→P (VÝVOJ → PLANNING)
- **G-VP-01:** Uživatel explicitně potvrdil rollback.
- **G-VP-02:** Neexistuje rozpracovaný D-xxx krok ve stavu IN_PROGRESS.
- **G-VP-03:** Change Impact Analysis do Decision Logu (typ ROLLBACK).

---

## 4. Data Model (G-002 entity)

Schéma `sm` v PostgreSQL. UUID PK, `TIMESTAMPTZ` UTC.

### 4.1 `sm.transition_log`

 Audit log všech pokusů o přechod (accepted i denied).

| Atribut | Typ | Povinný | Default | Validace | Zdroj | Persistence |
|---------|-----|---------|---------|----------|-------|-------------|
| id | UUID | ano | — | unique | aplikace | PK |
| project_id | UUID | ano | — | FK → core.project.id | aplikace | sloupec |
| from_phase | VARCHAR(32) | ano | — | enum: BRAINSTORMING, PLANNING, VÝVOJ, DEPLOY | aktuální fáze | sloupec |
| to_phase | VARCHAR(32) | ano | — | enum (stejný) | požadavek | sloupec |
| accepted | BOOLEAN | ano | — | not null | G-002 evaluace | sloupec |
| reason | VARCHAR(256) | ne | null | povinné pokud accepted=false | G-002 | sloupec |
| gate_results | JSONB | ano | `'[]'` | pole `{gate_id, passed:bool}` | G-002 | sloupec |
| requested_by | VARCHAR(64) | ano | — | enum: USER, SYSTEM | trigger | sloupec |
| created_at | TIMESTAMPTZ | ano | now() | — | aplikace | sloupec |

**Index:** `(project_id, created_at DESC)`, `(accepted)`.

### 4.2 `sm.gate_status`

 Poslední výsledek evaluace gate pro přechod (cache pro UI zobrazení „co chybí").

| Atribut | Typ | Povinný | Default | Validace | Zdroj | Persistence |
|---------|-----|---------|---------|----------|-------|-------------|
| id | UUID | ano | — | unique | aplikace | PK |
| project_id | UUID | ano | — | FK → core.project.id | aplikace | sloupec |
| transition | VARCHAR(16) | ano | — | enum: B_TO_P, P_TO_V, V_TO_D, P_TO_B, V_TO_P | aplikace | sloupec |
| gate_id | VARCHAR(16) | ano | — | pattern `^G-[A-Z]{2}-[0-9]{2}$` (např. G-BP-01) | aplikace | sloupec |
| passed | BOOLEAN | ano | — | not null | G-002 evaluace | sloupec |
| detail | TEXT | ne | null | popis proč passed/failed | G-002 | sloupec |
| evaluated_at | TIMESTAMPTZ | ano | now() | — | G-002 | sloupec |

**Constraint:** UNIQUE `(project_id, transition, gate_id)`. Upsert při každé evaluaci. **Index:** `(project_id, transition)`.

### 4.3 `sm.phase_config`

 Konfigurace fází (povolené přechody, popisky) — seed data.

| Atribut | Typ | Povinný | Default | Validace | Zdroj | Persistence |
|---------|-----|---------|---------|----------|-------|-------------|
| id | UUID | ano | — | unique | aplikace | PK |
| from_phase | VARCHAR(32) | ano | — | enum | aplikace (seed) | sloupec |
| to_phase | VARCHAR(32) | ano | — | enum | aplikace (seed) | sloupec |
| transition_code | VARCHAR(16) | ano | — | enum: B_TO_P, P_TO_V, V_TO_D, P_TO_B, V_TO_P | aplikace | sloupec |
| is_rollback | BOOLEAN | ano | false | — | aplikace | sloupec |
| requires_user_confirm | BOOLEAN | ano | false | true pro rollback | aplikace | sloupec |

**Constraint:** UNIQUE `(from_phase, to_phase)`. Seed: 5 povolených přechodů z matice 3.3.

---

## 5. Event Model (G-002)

G-002 publikuje eventy přes CORE event bus (S-003 sekce 5). Konzistentní s CORE SM-CORE-02.

| event_id | Trigger | Payload | Produceno | Consumery |
|----------|---------|---------|-----------|-----------|
| `phase.transition.requested` | uživatel žádá o přechod (přes CORE) | `{project_id, from, to, requested_by}` | Frontend (přes CORE) | CORE, G-002 |
| `phase.transition.validated` | G-002 dokončí evaluaci | `{project_id, from, to, accepted, reason?, gate_results}` | G-002 | CORE, G-007 (Diary), Frontend (WS) |
| `phase.transition.denied` | accepted=false | `{project_id, from, to, reason, gate_results}` | G-002 | CORE, Frontend (WS) |
| `phase.transition.accepted` | accepted=true (před zápisem CORE) | `{project_id, from, to, accepted_at}` | G-002 | CORE (zapíše current_phase + emit phase.entered), G-007, Frontend (WS) |
| `gate.evaluated` | každá gate check | `{project_id, transition, gate_id, passed, detail}` | G-002 | G-007 (metriky), Frontend (WS pro „co chybí") |

CORE přeposílá na Frontend přes WebSocket (`/api/core/stream`): `phase.transition.validated`, `phase.transition.denied`, `phase.transition.accepted`, `gate.evaluated`.

**Poznámka k `phase.entered`:** tento event emituje **CORE** (po zápisu `project.current_phase`), ne G-002 — viz S-003 sekce 5.2. G-002 pouze validuje.

---

## 6. API kontrakt (G-002)

G-002 vystavuje API pro Frontend (zobrazení gate stavů, „co chybí pro přechod") a interní volání z CORE. Hlavní transition endpoint je na CORE (`POST /api/core/phase/transition` — viz S-003 sekce 6.1). G-002 přidává read-only dotazy.

Backend (FastAPI), prefix `/api/sm`.

### 6.1 REST

#### `GET /api/sm/transitions`
Seznam povolených přechodů z aktuální fáze.
- 200: `[{from_phase, to_phase, transition_code, is_rollback, requires_user_confirm}]`
- Příklad pro `current_phase=BRAINSTORMING`: `[{from:"BRAINSTORMING", to:"PLANNING", transition_code:"B_TO_P", is_rollback:false, requires_user_confirm:false}]`

#### `GET /api/sm/gates/{transition_code}`
Aktuální stav gate pro přechod (co chybí).
- 200: `[{gate_id, passed, detail, evaluated_at}]`
- 404: neznámý transition_code

#### `POST /api/sm/gates/{transition_code}/evaluate`
Spustí re-evaluaci gate (uživatel tlačítko „Re-check" v UI).
- 200: `[{gate_id, passed, detail, evaluated_at}]` (upsert do `sm.gate_status`)
- 409: transition není povolen z aktuální fáze

#### `GET /api/sm/history?limit=50`
Historie pokusů o přechod.
- 200: `[{from_phase, to_phase, accepted, reason?, gate_results, requested_by, created_at}]`

### 6.2 Interní kontrakt (CORE → G-002)

Python interface (ne HTTP — in-process volání, jelikož CORE a G-002 běží ve stejném backend kontejneru):

```python
class StateMachineGanglion:
    def validate_transition(self, project_id: UUID, from_phase: str, to_phase: str,
                            requested_by: str = "USER") -> TransitionResult:
        """
        Vyhodnotí přechod. Vrací:
          - accepted: bool
          - reason: str | None  (pokud denied)
          - gate_results: list[{gate_id, passed, detail}]
          - accepted_at: datetime | None
        Persistuje do sm.transition_log (vždy) a sm.gate_status (upsert).
        Emituje phase.transition.validated (+ denied nebo accepted).
        """

    def get_transition_gates(self, transition_code: str) -> list[GateResult]:
        """Vrací aktuální gate stavy pro přechod."""

    def evaluate_gates(self, project_id: UUID, transition_code: str) -> list[GateResult]:
        """Re-evaluuje gaty, upsert do sm.gate_status, emituje gate.evaluated pro každý."""
```

`TransitionResult`: `{accepted: bool, reason: str | None, gate_results: list, accepted_at: datetime | None}`
`GateResult`: `{gate_id, passed, detail, evaluated_at}`

---

## 7. Gate evaluace — implementační detail

Každý gate (G-BP-01..G-VD-05) je implementován jako funkce `(project_id) -> GateResult`. G-002 volá sadu funkcí pro daný `transition_code`. Funkce se dotahují na potřebná data:

- **Specifikace stupeň** — čte metadata specifikace (verze, stupeň) — v produkci přes G-007/persistence metadata.
- **Decision Log** — čte persistence (DL-xxx záznamy).
- **Ganglion registry** — čte `core.ganglion_registry` (state, health).
- **Health monitor** — čte `core.system_health`.
- **Implementační tasky** — čte metadata P-xxx/D-xxx (přes G-007/persistence).
- **LLM modely** — čte `llm.model` (is_active, state) přes G-001 (volání API nebo in-process).

**Izolace:** G-002 nečte cizí tabulky přímo (kromě `core.ganglion_registry` a `core.system_health` — read-only). Pro specifikace/plány/tasky používá G-007 (Developer Diary/persistence) nebo CORE API. Závislosti jsou explicitní v IT-SM-* (viz sekce 11).

**Konzistence:** Pokud gate check selže (např. G-001 není READY při P→V gate), `passed=false`, `detail="no_active_llm_model"`. Uživatel vidí v UI co přesně chybí.

---

## 8. Edge cases

| EC-ID | Scénář | Očekávané chování |
|-------|--------|-------------------|
| EC-SM-01 | Dva souběžné požadavky na `phase/transition` | Druhý vrací 409 (přechod právě probíhá); in-process mutex na `(project_id)` v G-002. |
| EC-SM-02 | Přechod B→P když gate G-BP-03 selhává (ganglion N/A) | `accepted=false`, `reason="gate_failed"`, `gate_results` obsahuje `{G-BP-03, passed:false, detail:"G-006 IDE Bridge not DEFINED"}`; UI zobrazí konkrétní gate. |
| EC-SM-03 | Rollback P→B bez potvrzení uživatelem | `accepted=false`, `reason="user_confirmation_required"`; UI vyžaduje confirm dialog. |
| EC-SM-04 | Gate check závisí na G-001, který je ERROR | `G-PV-04` → `passed=false`, `detail="llm_provider_error"`; uživatel musí vyřešit LLM Manager stav. |
| EC-SM-05 | Přechod na stejnou fázi (BRAINSTORMING→BRAINSTORMING) | `accepted=false`, `reason="invalid_transition"` (z matice 3.3). |
| EC-SM-06 | DEPLOY projekt, uživatel chce přechod | Všechny přechody `accepted=false`, `reason="project_closed"`; UI nabídne „Nový projekt" (start v BRAINSTORMING). |
| EC-SM-07 | Gate evaluace vyhodí výjimku (např. DB nedostupná) | `passed=false`, `detail="evaluation_error: <msg>"`; přechod denied; `phase.transition.denied`; Frontend ukáže ERROR s retry. |
| EC-SM-08 | `sm.transition_log` roste | Retence 365 dnů (cron maže starší); `sm.gate_status` upsert (neroste). |
| EC-SM-09 | Uživatel re-evaluuje gaty, ale nic se nezměnilo | Upsert stejných hodnot; `gate.evaluated` emitován (pro UI refresh); `evaluated_at` aktualizováno. |
| EC-SM-10 | Rollback z PLANNING když IT-xxx IN_PROGRESS | `accepted=false`, `reason="in_progress_tasks"`, `detail` seznam IT-xxx; uživatel musí ukončit tasky. |
| EC-SM-11 | Gate check závisí na externí službě (ollama DOWN při P→V) | `G-PV-04`/`G-PV-05` → `passed=false` dle závislosti; detail jasný; přechod blokován dokud health UP. |
| EC-SM-12 | Změna konfigurace gate (nový gate přidán) | `sm.phase_config`/gate registry aktualizován; staré `sm.gate_status` záznamy pro neexistující gate se ignorují (nebo mažou) při evaluaci. |

---

## 9. Acceptance Criteria

| AC-ID | Kritérium (objektivně měřitelné) | Ověření |
|-------|----------------------------------|---------|
| AC-SM-01 | `GET /api/sm/transitions` pro `current_phase=BRAINSTORMING` vrací přesně 1 přechod (B_TO_P); pro PLANNING vrací 2 (P_TO_V, P_TO_B). | API test |
| AC-SM-02 | `validate_transition(BRAINSTORMING→PLANNING)` s nesplněným gate G-BP-01 vrací `accepted=false` a `gate_results` obsahuje `{G-BP-01, passed:false}`. | Unit test |
| AC-SM-03 | Po splnění všech gate B→P `validate_transition` vrací `accepted=true`; `sm.transition_log` má záznam s `accepted=true`; CORE následně emituje `phase.entered`. | Integration test |
| AC-SM-04 | `POST /api/sm/gates/B_TO_P/evaluate` upsertuje `sm.gate_status` a vrací aktuální stavy; `gate.evaluated` event emitován pro každý gate. | API + event test |
| AC-SM-05 | Neplatný přechod (VÝVOJ→BRAINSTORMING) → `accepted=false`, `reason="invalid_transition"`. | Unit test |
| AC-SM-06 | Rollback P→B bez potvrzení → `accepted=false`, `reason="user_confirmation_required"`. | Unit test |
| AC-SM-07 | Souběžné `validate_transition` (2 vlákna, stejný projekt) → druhý denied s `reason="transition_in_progress"`. | Concurrency test |
| AC-SM-08 | `phase.transition.denied` event obsahuje `reason` a `gate_results`. | Event listener test |
| AC-SM-09 | DEPLOY projekt: všechny přechody denied s `reason="project_closed"`. | API test |
| AC-SM-10 | Gate evaluace výjimka (DB down) → `passed=false`, `detail` obsahuje `evaluation_error`; přechod denied; žádný crash. | Fault injection test |
| AC-SM-11 | `sm.transition_log` záznamy starší 365 dnů smazány cronem; `sm.gate_status` neroste (upsert). | Cron unit test |
| AC-SM-12 | `GET /api/sm/history?limit=50` vrací max 50 záznamů seřazených `created_at DESC`. | API test |

---

## 10. UI propojení (Views)

G-002 je napojen na tři Views (detailní specifikace v S-013):

### VIEW-002: Brainstorming (parent: G-002 + G-007)
- **Ovládací prvky:**
  - Zobrazení aktuální fáze badge (BRAINSTORMING).
  - Tlačítko „Přejít do PLANNING" (disabled pokud gate B→P nesplněn) → `POST /api/core/phase/transition {to:"PLANNING"}`.
  - Panel „Co chybí pro přechod" → `GET /api/sm/gates/B_TO_P` — seznam gate s `passed` (zelená/červená) a `detail`.
  - Tlačítko „Re-check gates" → `POST /api/sm/gates/B_TO_P/evaluate`.
  - Odkaz na specifikaci (editovatelná v Edit Mode dle specifikace S-013).
- **Stavy View:** LOADING, READY, ERROR (gate evaluation error), EDIT (Edit Mode specifikace).

### VIEW-003: Planning (parent: G-002 + G-007)
- **Ovládací prvky:**
  - Zobrazení aktuální fáze badge (PLANNING).
  - Tlačítko „Přejít do VÝVOJ" (disabled pokud gate P→V nesplněn) → `POST /api/core/phase/transition {to:"VÝVOJ"}`.
  - Tlačítko „Rollback do BRAINSTORMING" → confirm dialog → `POST /api/core/phase/transition {to:"BRAINSTORMING"}` (přechod P_TO_B).
  - Panel „Co chybí pro VÝVOJ" → `GET /api/sm/gates/P_TO_V`.
  - Seznam plánovacích kroků P-001..P-015 (z G-007/persistence) se stavem.
  - Tlačítko „Re-check gates" → `POST /api/sm/gates/P_TO_V/evaluate`.
- **Stavy View:** LOADING, READY, ERROR, EDIT.

### VIEW-004: Development (parent: G-002 + G-007)
- **Ovládací prvky:**
  - Zobrazení aktuální fáze badge (VÝVOJ).
  - Tlačítko „Přejít do DEPLOY" (disabled pokud gate V→D nesplněn) → `POST /api/core/phase/transition {to:"DEPLOY"}`.
  - Tlačítko „Rollback do PLANNING" → confirm dialog → `POST /api/core/phase/transition {to:"PLANNING"}` (přechod V_TO_P).
  - Panel „Co chybí pro DEPLOY" → `GET /api/sm/gates/V_TO_D`.
  - Seznam implementačních kroků D-001..D-015 (z G-007) se stavem a výsledky testů.
  - Tlačítko „Re-check gates" → `POST /api/sm/gates/V_TO_D/evaluate`.
- **Stavy View:** LOADING, READY, ERROR, EDIT.

### Společné pro všechny fáze Views
- **Historie přechodů** (přístupné z každého fáze View): `GET /api/sm/history` — tabulka `from→to, accepted, reason, created_at`.
- **WebSocket** notifikace: při `phase.transition.validated/denied/accepted` a `gate.evaluated` — UI aktualizuje gate panel a badge v reálném čase.

---

## 11. Implementation Tasks (odkaz)

Detailní tasky v PLANNING fázi (P-004). Pro G-002:
- IT-SM-01: DB migrace (schéma `sm`, 3 tabulky, indexy, seed `sm.phase_config`)
- IT-SM-02: Transition matic (povolené přechody, validation logic)
- IT-SM-03: Gate evaluátory (G-BP-*, G-PV-*, G-VD-*, G-PB-*, G-VP-*)
- IT-SM-04: `validate_transition` in-process interface (pro CORE)
- IT-SM-05: Event emitování (phase.transition.*, gate.evaluated)
- IT-SM-06: REST API (transitions, gates, evaluate, history)
- IT-SM-07: Retence cron (transition_log 365 dnů)
- IT-SM-08: Testy (AC-SM-01..AC-SM-12)

**Závislosti:**
- IT-SM-04 závisí na CORE `POST /api/core/phase/transition` (S-003 IT-CORE-05).
- IT-SM-03 gate G-PV-04 závisí na G-001 (S-004) — `llm.model` is_active/READY.
- IT-SM-03 gate G-PV-05 závisí na CORE health monitor (S-003 SM-CORE-03).

---

## 12. Definition of Done (S-005)

- [x] Stavy a přechody specifikovány (4 fáze, matice povolených přechodů, diagram)
- [x] Brány (gates) specifikovány (5 přechodů, 25 konkrétních gate checků)
- [x] Data model specifikován (3 entity: transition_log, gate_status, phase_config)
- [x] Event model specifikován (5 eventů, konzistentní s CORE event busem a SM-CORE-02)
- [x] API kontrakt specifikován (REST + in-process interface pro CORE)
- [x] Gate evaluační logika specifikována (izolace, závislosti)
- [x] Edge cases specifikovány (12 scénářů)
- [x] Acceptance criteria specifikována (12 měřitelných)
- [x] UI propojení specifikováno (VIEW-002/003/004 s ovládacími prvky pro přechod, rollback, re-check)
- [x] Konzistence s CORE SM-CORE-02 (delegace validace, zdroj pravdy v CORE)
- [x] Lze implementovat bez domýšlení (konzistentní s DL-001..DL-004, GLOSSARY, S-003 CORE, S-004 LLM Manager)

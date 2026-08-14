# Specifikace GANGLION: LLM Manager (G-001)

> **Stav specifikace:** Stupeň 4 — SPECIFIED
> **Krok plánu:** S-004
> **Ganglion:** G-001 — LLM Manager
> **Závislosti:** S-002 (páteřní specifikace), S-003 (CORE)
> **Schválená rozhodnutí:** DL-001 (Python BE + React FE), DL-002 (React + Vite), DL-003 (Specifikace první), DL-004 (akronym)
> **Terminologie:** viz `docs/specification/GLOSSARY.md` — LLM Manager, Nexus, CORE, View
> **Views:** VIEW-005 Models Dashboard, VIEW-006 Model Detail

---

## 1. Účel a rozsah

GANGLION LLM Manager (G-001) spravuje životní cyklus LLM modelů v systému L.O.N.G.I.N. Coder. Poskytuje:

1. **Správu providerů** — lokální (Ollama, LM Studio) a cloud (OpenAI-compatible API).
2. **Načítání/přepínání modelů** za běhu bez restartu aplikace.
3. **Sledování stavu modelu** (INIT → LOADING → READY → ERROR → UNLOADED).
4. **Tokenizér a token counting** — počítání tokenů zpráv pro context window management.
5. **Inference proxy** — jednotné rozhraní pro chat/completion volání napříč providery.
6. **Cache** — opakované dotazy s identickým obsahem (stejný model + prompt + parametry) vrací cached odpověď (reakce na zadání „tokenizéry databáze postgres s PG vektor rozšířením ready pro cache").

**Nevlastní:** Conversation (G-003) vlastní zprávy a session; Vector Store (G-004) vlastní embeddingy a RAG. LLM Manager vlastní pouze `llm.provider`, `llm.model`, `llm.inference_cache`.

**Nexus (hardwarová vrstva):** LLM inference běží na lokálním GPU/CPU NEXUS stroje. LLM Manager deleguje hardwarový přístup na Ollama/LM Studio kontejnery (samotný hardware není přímo spravován G-001, ale přes providery).

---

## 2. Responsibility Matrix (G-001)

| Odpovědnost | CORE | G-001 LLM Manager | Nexus | Jiný Ganglion |
|-------------|------|-------------------|-------|---------------|
| Lifecycle ganglia G-001 | ✓ (registry) | ✓ (init/logika) | | |
| Načítání/přepínání modelu | | ✓ | | |
| Inference volání (chat/completion) | | ✓ | ✓ (hardware) | |
| Token counting | | ✓ | | |
| Inference cache | | ✓ | | G-004 (PGVector infra) |
| Ukládání zpráv/session | | | | G-003 Conversation |
| Embedding generace | | ✓ (volá provider) | | G-004 (storage) |
| API klíče (cloud) | | ✓ (čte z secret store) | | |
| Health providera (Ollama/LM Studio) | ✓ (health monitor) | ✓ (provozní stav) | | |

---

## 3. Provider kontrakty

LLM Manager abstrahuje tři typy providerů přes jednotné vnitřní rozhraní `LLMProvider`. Každý provider má vlastní HTTP/SDK klient.

### 3.1 Provider: Ollama (lokální)

- **Endpoint:** `http://ollama:11434` (Docker service; konfigurovatelné v `.env` jako `OLLAMA_BASE_URL`)
- **Dostupné modely:** `GET /api/tags` → seznam lokálně stažených modelů (`name`, `size`, `modified_at`).
- **Načtení modelu:** `POST /api/generate` s `model` parametrem — Ollama lazy-loads model při prvním volání. LLM Manager volá warmup `POST /api/generate {model, prompt:"", stream:false}` pro přechod do READY.
- **Chat/Completion:** `POST /api/chat` (messages) nebo `POST /api/generate` (prompt). Podpora streamingu (`stream:true`).
- **Embedding:** `POST /api/embeddings` → vektor (delegováno G-004 pro storage).
- **Token counting:** Ollama nevrací přesný token count; LLM Manager odhaduje přes `ollama-tokenizer` (lokální Python balíček `tiktoken` fallback pro kompatibilní modely) nebo čte `prompt_eval_count`/`eval_count` z odpovědi `/api/chat` response.
- **Auth:** žádná (lokální).
- **Health:** `GET /api/tags` 200 = UP; connection refused = DOWN.

### 3.2 Provider: LM Studio (lokální)

- **Endpoint:** `http://lm-studio-proxy:1234/v1` (OpenAI-compatible; Docker service `lm-studio-proxy`; konfigurovatelné `LMSTUDIO_BASE_URL`)
- **Dostupné modely:** `GET /v1/models` → seznam načtených modelů v LM Studio.
- **Načtení modelu:** LM Studio načítá model uživatelem v GUI; LLM Manager detekuje přítomnost přes `GET /v1/models`. Pokud model není přítomen, vrací seznam dostupných a stav modelu = NOT_LOADED (nelze force-load přes API — LM Studio omezení).
- **Chat/Completion:** `POST /v1/chat/completions` (OpenAI-compatible). Streaming přes SSE.
- **Embedding:** `POST /v1/embeddings` (pokud model podporuje).
- **Token counting:** LM Studio vrací `usage.prompt_tokens`/`completion_tokens` v response.
- **Auth:** volitelný `Authorization: Bearer <local-key>` (konfigurovatelné, default prázdné).
- **Health:** `GET /v1/models` 200 = UP.

### 3.3 Provider: OpenAI-compatible (cloud)

- **Endpoint:** konfigurovatelné `OPENAI_BASE_URL` (např. `https://api.openai.com/v1` nebo self-hosted kompatibilní server).
- **Dostupné modely:** `GET /v1/models` (vyžaduje API klíč).
- **Načtení modelu:** cloud modely jsou vždy „načtené" — přechod do READY ihned po validaci klíče a existence modelu.
- **Chat/Completion:** `POST /v1/chat/completions`. Streaming přes SSE.
- **Embedding:** `POST /v1/embeddings`.
- **Token counting:** `usage.prompt_tokens`/`completion_tokens` v response.
- **Auth:** `Authorization: Bearer <api_key>` — API klíč čten z lokálního secret store (šifrovaný, viz Security S-024). Klíč nikdy nevracen v API response ani v logu.
- **Health:** `GET /v1/models` 200 = UP; 401/403 = DOWN (auth); timeout = DOWN.
- **Poznámka:** Cloud volání opouští stroj — pouze uživatelem explicitně konfigurovaná (NFR/Security).

### 3.4 Sjednocený vnitřní kontrakt `LLMProvider`

Každý provider implementuje:
```
interface LLMProvider {
  code: "ollama" | "lm_studio" | "openai"
  list_models() -> [ModelDescriptor]
  load_model(model_id) -> void          // warmup / validate
  chat(messages, params) -> ChatResult  // non-stream
  chat_stream(messages, params) -> AsyncIterator[ChatChunk]
  embed(text, model_id) -> [float]
  count_tokens(text, model_id) -> int
  health() -> HealthState
}
```

`ChatResult`: `{content, model, usage:{prompt_tokens, completion_tokens, total_tokens}, cached:bool, latency_ms}`
`ChatChunk`: `{delta, finish_reason|null}`

---

## 4. Data Model (G-001 entity)

Schéma `llm` v PostgreSQL. UUID primární klíče, `TIMESTAMPTZ` UTC.

### 4.1 `llm.provider`

 Registry konfigurovaných providerů (uloženo při startu z `.env`).

| Atribut | Typ | Povinný | Default | Validace | Zdroj | Persistence |
|---------|-----|---------|---------|----------|-------|-------------|
| id | UUID | ano | — | unique | aplikace | PK |
| code | VARCHAR(32) | ano | — | unique; enum: ollama, lm_studio, openai | aplikace | sloupec |
| base_url | VARCHAR(512) | ano | — | valid URL | `.env` | sloupec |
| auth_required | BOOLEAN | ano | false | — | aplikace | sloupec |
| has_api_key | BOOLEAN | ano | false | true pokud auth_required | secret store | sloupec (jen indikace, ne hodnota) |
| created_at | TIMESTAMPTZ | ano | now() | — | aplikace | sloupec |
| updated_at | TIMESTAMPTZ | ano | now() | — | aplikace | sloupec |

**Constraint:** `code` UNIQUE. API klíč sám se nikdy neukládá do DB — pouze v secret store (šifrovaný soubor/env runtime).

### 4.2 `llm.model`

 Registry modelů a jejich runtime stavu.

| Atribut | Typ | Povinný | Default | Validace | Zdroj | Persistence |
|---------|-----|---------|---------|----------|-------|-------------|
| id | UUID | ano | — | unique | aplikace | PK |
| provider_code | VARCHAR(32) | ano | — | FK → llm.provider.code | aplikace | sloupec |
| model_name | VARCHAR(128) | ano | — | 1–128 znaků; unique per provider | provider `list_models` | sloupec |
| display_name | VARCHAR(128) | ne | null | — | uživatel (volitelné) | sloupec |
| state | VARCHAR(16) | ano | `'INIT'` | enum: INIT, LOADING, READY, ERROR, NOT_LOADED, UNLOADED | G-001 lifecycle | sloupec |
| context_window | INTEGER | ne | null | > 0 | provider metadata | sloupec |
| loaded_at | TIMESTAMPTZ | ne | null | — | G-001 | sloupec |
| last_error | TEXT | ne | null | — | G-001 | sloupec |
| is_active | BOOLEAN | ano | false | max 1 aktivní model v jednu chasu | uživatel | sloupec |
| created_at | TIMESTAMPTZ | ano | now() | — | aplikace | sloupec |
| updated_at | TIMESTAMPTZ | ano | now() | — | aplikace | sloupec |

**Constraint:** UNIQUE `(provider_code, model_name)`. `is_active=true` smí být pouze jeden řádek v celé tabulce (aktivní model pro inference). Partial unique index na `(is_active) WHERE is_active = true`.

### 4.3 `llm.inference_cache`

 Cache pro identické inference požadavky (reakce na zadání o cache).

| Atribut | Typ | Povinný | Default | Validace | Zdroj | Persistence |
|---------|-----|---------|---------|----------|-------|-------------|
| id | UUID | ano | — | unique | aplikace | PK |
| model_id | UUID | ano | — | FK → llm.model.id | aplikace | sloupec |
| prompt_hash | VARCHAR(64) | ano | — | sha256 hex | aplikace | sloupec |
| params_hash | VARCHAR(64) | ano | — | sha256 hex (json params) | aplikace | sloupec |
| response | JSONB | ano | — | valid JSON | provider | sloupec |
| usage | JSONB | ne | null | {prompt_tokens, completion_tokens} | provider | sloupec |
| created_at | TIMESTAMPTZ | ano | now() | — | aplikace | sloupec |
| hit_count | INTEGER | ano | 0 | >= 0 | G-001 | sloupec |

**Constraint:** UNIQUE `(model_id, prompt_hash, params_hash)`. **TTL:** 7 dnů (cron maže starší). **Hash vstup:** kanonický JSON `(messages, params)` seřazený klíče → sha256.

### 4.4 `llm.token_estimate`

 Volitelná tabulka pro prediktivní token counting (pokud provider nepodporuje přesný count).

| Atribut | Typ | Povinný | Default | Validace | Zdroj | Persistence |
|---------|-----|---------|---------|----------|-------|-------------|
| id | UUID | ano | — | unique | aplikace | PK |
| model_id | UUID | ano | — | FK | aplikace | sloupec |
| text_hash | VARCHAR(64) | ano | — | sha256 hex | aplikace | sloupec |
| token_count | INTEGER | ano | — | >= 0 | tokenizer | sloupec |
| created_at | TIMESTAMPTZ | ano | now() | — | aplikace | sloupec |

**Constraint:** UNIQUE `(model_id, text_hash)`. Slouží jako cache pro drahé token counting operace.

---

## 5. State Machine — SM-LLM-01 (načítání modelu)

```
INIT
 │ (user request load_model)
 ▼
LOADING
 │ (warmup success / cloud validation ok) ──► READY
 │ (load fail / not available) ────────────► ERROR
 │ (LM Studio: model not present) ─────────► NOT_LOADED
 ▼
READY ◄──────────── (recover z ERROR po retry úspěchu)
 │ (user switch to different model)
 ▼
UNLOADED
 │ (user re-selects)
 ▼
LOADING (cycle)
```

| Přechod | Trigger | Podmínka | Eventy | Vedlejší efekt |
|---------|---------|----------|--------|----------------|
| INIT → LOADING | `POST /api/llm/models/{id}/load` | provider UP | `model.load.started` | `llm.model.state=LOADING` |
| LOADING → READY | warmup/validation úspěch | provider hlásí dostupnost | `model.ready` | `loaded_at`, `state=READY` |
| LOADING → ERROR | výjimka / timeout | zachyceno | `model.error` | `last_error`, `state=ERROR` |
| LOADING → NOT_LOADED | LM Studio model chybí | `GET /v1/models` neobsahuje model | `model.error` (reason=not_loaded) | `state=NOT_LOADED`, `last_error` |
| READY → ERROR | inference selhání | runtime chyba | `model.error` | `state=ERROR`, `last_error` |
| ERROR → LOADING | `POST .../load` (retry) | uživatel | `model.load.started` | `state=LOADING` |
| READY → UNLOADED | uživatel přepne aktivní model | jiný model se stává active | `model.unloaded` | `state=UNLOADED`, `is_active=false` |
| UNLOADED → LOADING | uživatel reaktivuje | — | `model.load.started` | `state=LOADING` |

**Persistence:** `llm.model.state`, `loaded_at`, `is_active`, `last_error`.

**Atomicita aktivace:** Přepnutí aktivního modelu probíhá v transakci: `BEGIN; UPDATE llm.model SET is_active=false WHERE is_active=true; UPDATE llm.model SET is_active=true, state='LOADING' WHERE id=?; COMMIT;`. Následně async warmup.

---

## 6. Event Model (G-001)

G-001 publikuje eventy přes CORE event bus (viz S-003 sekce 5).

| event_id | Trigger | Payload | Produceno | Consumery |
|----------|---------|---------|-----------|-----------|
| `model.load.started` | SM-LLM-01 přechod | `{model_id, provider_code, model_name}` | G-001 | CORE, G-003 (Conversation), G-007 (Diary), Frontend (WS) |
| `model.ready` | SM-LLM-01 → READY | `{model_id, loaded_at}` | G-001 | CORE, G-003, G-007, Frontend |
| `model.error` | SM-LLM-01 → ERROR/NOT_LOADED | `{model_id, error, reason?}` | G-001 | CORE, G-007, Frontend (WS) |
| `model.unloaded` | SM-LLM-01 → UNLOADED | `{model_id}` | G-001 | G-003, G-007, Frontend |
| `model.switched` | aktivace jiného modelu | `{from_model_id, to_model_id}` | G-001 | G-003, G-007, Frontend (WS) |
| `inference.cache_hit` | cache zásah | `{model_id, cache_id}` | G-001 | G-007 (metriky) |
| `provider.health.changed` | změna stavu providera | `{provider_code, from, to}` | G-001 | CORE, Frontend (WS) |

CORE přeposílá vybrané eventy na Frontend přes WebSocket (`/api/core/stream`): `model.ready`, `model.error`, `model.switched`, `provider.health.changed`.

---

## 7. API kontrakt (G-001)

Backend (FastAPI), prefix `/api/llm`. CORS omezeno dle CORE (S-003 sekce 6).

### 7.1 REST

#### `GET /api/llm/providers`
- 200: `[{code, base_url, auth_required, has_api_key, health: "UP|DOWN|UNKNOWN"}]`

#### `GET /api/llm/providers/{code}/models`
Seznam modelů providera (živý dotaz na provider).
- 200: `[{model_name, context_window?, size?}]`
- 502: provider DOWN → `{"error":"provider_unavailable","code":"..."}`
- 401: cloud provider auth selhal

#### `GET /api/llm/models`
Registry lokálně známých modelů (z DB).
- 200: `[{id, provider_code, model_name, display_name, state, is_active, loaded_at, last_error, context_window}]`

#### `POST /api/llm/models`
Registrace modelu do registry (před načtením).
- Body: `{provider_code, model_name, display_name?, context_window?}`
- 201: model objekt
- 409: model již existuje

#### `POST /api/llm/models/{id}/load`
Načtení/reaktivace modelu (přechod INIT/UNLOADED/ERROR/NOT_LOADED → LOADING).
- 202: `{model_id, state:"LOADING"}`
- 409: jiný model právě LOADING na stejném provideru (Ollama/LM Studio mohou obsloužit typicky 1 aktivní model najednou)

#### `POST /api/llm/models/{id}/activate`
Aktivace modelu (nastaví is_active=true, deaktivuje předchozí).
- 200: `{model_id, is_active:true, previous_active?}`
- 409: model není READY → `{"error":"not_ready","state":"..."}`
- 400: model ve stavu NOT_LOADED → `{"error":"not_loaded","hint":"load model first"}`

#### `DELETE /api/llm/models/{id}`
Odregistrování modelu (jen pokud není aktivní).
- 204
- 409: model je aktivní

#### `POST /api/llm/chat`
Inference (non-stream).
- Body: `{messages:[{role, content}], params?:{temperature?, max_tokens?, top_p?}, use_cache?:bool=true}`
- 200: `{content, model, usage:{prompt_tokens, completion_tokens, total_tokens}, cached, latency_ms}`
- 409: žádný aktivní model → `{"error":"no_active_model"}`
- 502: aktivní model ve stavu ERROR → `{"error":"model_error","last_error":"..."}`
- 504: inference timeout (default 120s lokální, 60s cloud)

#### `POST /api/llm/chat/stream`
Inference streaming (SSE).
- Body: stejné jako `/chat`
- 200: `text/event-stream` — `data: {delta, finish_reason?}\n\n`
- Stejné chybové kódy

#### `POST /api/llm/embed`
Generace embeddingu (delegováno na provider, storage řeší G-004).
- Body: `{text, model_id?}` (default aktivní embedding model)
- 200: `{vector:[float], model, usage?}`
- 409: žádný model pro embedding

#### `POST /api/llm/count-tokens`
Token counting.
- Body: `{text, model_id?}`
- 200: `{token_count, model_id, cached:bool}`
- 200 (fallback): pokud provider nepodporuje, vrací odhad s `{"estimated":true}`

### 7.2 WebSocket

LLM Manager sám neotevírá vlastní WS — inference streaming je přes HTTP SSE (`/api/llm/chat/stream`). Stavové eventy (`model.*`, `provider.health.changed`) jdou přes CORE WS `/api/core/stream`.

---

## 8. Cache strategie

1. **Cache key:** `sha256(canonical_json(messages) + params_hash)` pro daný `model_id`.
2. **Cache hit:** pokud záznam v `llm.inference_cache` s TTL < 7 dnů, vrátí se cached `response` a `usage`, `cached=true`.
3. **Cache bypass:** `use_cache=false` v requestu → vždy živé volání, cache se neaktualizuje.
4. **Invalidace při model error:** pokud aktivní model přejde do ERROR, cache zůstává (odpovědi jsou stále validní pro historické dotazy).
5. **Invalidace při model switch:** cache je per `model_id` — přepnutí modelu neinvaliduje cache předchozího modelu.
6. **Velikost:** pokud cache > 10000 záznamů, LRU eviction (maže nejstarší dle `created_at` s nejnižším `hit_count`).

---

## 9. Token management

1. **Přesný count:** pokud provider vrací `usage` (LM Studio, OpenAI), použije se přesný `prompt_tokens`/`completion_tokens`.
2. **Odhad:** Ollama — `tiktoken` encoding pro kompatibilní modely (cl100k_base fallback); výsledek cachován v `llm.token_estimate`.
3. **Context window:** každý model má `context_window` (z provider metadata nebo ruční konfigurace). Při sestavování kontextu (G-003 Conversation) se G-001 dotazuje na `count_tokens` a hlásí `context_window_exceeded` event, pokud prompt > context_window.
4. **Limit:** LLM Manager odmítne inference, pokud `prompt_tokens` > `context_window` → 400 `{"error":"context_window_exceeded","prompt_tokens":N,"context_window":M}`.

---

## 10. Edge cases

| EC-ID | Scénář | Očekávané chování |
|-------|--------|-------------------|
| EC-LLM-01 | Ollama kontejner DOWN při `load_model` | `model.load.started` → `model.error` (reason=provider_down); `state=ERROR`; UI (VIEW-006) ukáže ERROR s retry tlačítkem. |
| EC-LLM-02 | LM Studio model uživatelem unloadnut v GUI mid-session | `provider.health.changed` (UP→DEGRADED); příští inference → 502; `model.error` (reason=not_loaded); `state=NOT_LOADED`; UI nabídne reload. |
| EC-LLM-03 | Cloud API klíč expiroval / neplatný | `GET /providers/openai/models` → 401; `provider.health.changed` (UP→DOWN); `has_api_key` zůstává true; UI (VIEW-005) ukáže „API key invalid" v Settings (VIEW-010). |
| EC-LLM-04 | Inference timeout (lokální model pomalý) | 504 po timeout (120s lokální); `model.error`; `state=ERROR`; UI ukáže timeout s možností zvýšit timeout v nastavení. |
| EC-LLM-05 | Přepnutí aktivního modelu během běžící inference | Nová inference čeká (fronta 1) nebo vrátí 409 (jiný model active); stará inference se nijak neruší (dokončí, ale její výsledek se zahodí pokud model deactivated). |
| EC-LLM-06 | Cache zásah pro odlišný model se stejným promptem | Nezasáhne — cache key obsahuje `model_id`; každý model má vlastní cache. |
| EC-LLM-07 | Prompt přesahuje context window | `count-tokens` vrátí count; při `/chat` 400 `context_window_exceeded`; UI (VIEW Conversation) upozorní a nabídne zkrácení kontextu. |
| EC-LLM-08 | Dva souběžné `load_model` na Ollama (1 model limit) | Druhý vrací 409 (jiný model LOADING); mutex per provider. |
| EC-LLM-09 | Streaming spojení přerušeno klientem | Server ukončí stream; částečná odpověď se necachuje; žádná persistence. |
| EC-LLM-10 | Provider vrací embedding o špatné dimenzi | G-001 validuje dimenzi vůči očekávané (z konfigurace modelu); 502 `{"error":"embedding_dim_mismatch","expected":N,"actual":M}`; G-004 neuloží. |
| EC-LLM-11 | Token count estimate má velkou odchylku | `estimated:true` indikátor v response; UI může zobrazit „~" před číslem. |
| EC-LLM-12 | Aktivní model smazán (DELETE) | 409 — nelze smazat aktivní model; uživatel musí nejprve přepnout/deaktivovat. |

---

## 11. Acceptance Criteria

| AC-ID | Kritérium (objektivně měřitelné) | Ověření |
|-------|----------------------------------|---------|
| AC-LLM-01 | `GET /api/llm/providers` vrací 3 providery (ollama, lm_studio, openai) s `health` stavem. | HTTP test |
| AC-LLM-02 | `POST /api/llm/models/{id}/load` na Ollama model (UP) → model přejde INIT→LOADING→READY do 60s a `model.ready` event je emitován. | API + event listener |
| AC-LLM-03 | `POST /api/llm/models/{id}/activate` nastaví `is_active=true` a deaktivuje předchozí (`model.switched` event); pouze 1 řádek s `is_active=true` v DB. | DB kontrola + event |
| AC-LLM-04 | Identický `/chat` požadavek (stejný model+messages+params) vrací `cached=true` při druhém volání a stejný `content`. | API test |
| AC-LLM-05 | `/chat` s `use_cache=false` vrací `cached=false` a nečte z cache. | API test |
| AC-LLM-06 | `/chat` když aktivní model je ERROR → 502 s `model_error`. | API test |
| AC-LLM-07 | `/chat` když žádný model aktivní → 409 `no_active_model`. | API test |
| AC-LLM-08 | `/chat/stream` vrací SSE chunky s `delta`; po dokončení `finish_reason="stop"`. | SSE test |
| AC-LLM-09 | `/count-tokens` vrací `token_count` >= 0; pro OpenAI-compatible model přesný (neodhadovaný). | API test |
| AC-LLM-10 | `/chat` s prompt > context_window → 400 `context_window_exceeded`. | API test |
| AC-LLM-11 | Cache záznamy starší 7 dnů jsou smazány cron jobem; při > 10000 záznamů LRU eviction. | Unit test cron logiky |
| AC-LLM-12 | API klíč pro cloud provider se nikdy neobjeví v žádné API response ani v `core.event_log` payload. | Grep test nad logy/responses |
| AC-LLM-13 | Při Ollama DOWN `/providers/ollama/models` → 502 a `provider.health.changed` event. | Integration test |
| AC-LLM-14 | DELETE aktivního modelu → 409; po deaktivaci DELETE → 204. | API test |

---

## 12. UI propojení (Views)

G-001 je napojen na dva Views (detailní specifikace v S-013):

### VIEW-005: Models Dashboard (parent: G-001)
- **Ovládací prvky (kontrolní prvkya dle instrukcí):**
  - Seznam providerů se stavem health (UP/DOWN/UNKNOWN) + ikonou.
  - Tlačítko „Refresh models" → `GET /providers/{code}/models` a registrace nových.
  - Seznam modelů s `state` (INIT/LOADING/READY/ERROR/NOT_LOADED/UNLOADED), `is_active` badge.
  - Tlačítko „Load" u každého modelu → `POST /models/{id}/load`.
  - Tlačítko „Activate" u READY modelu → `POST /models/{id}/activate`.
  - Tlačítko „Delete" u neaktivního modelu → `DELETE /models/{id}` (potvrzovací dialog).
  - „Active model" indikátor vždy viditelný.
- **Stavy View:** LOADING (načítá seznam), READY (zobrazeno), EMPTY (žádné modely), ERROR (provider down), EDIT (Edit Mode — úprava `display_name`).
- **Data contract:** `GET /api/llm/models` + `GET /api/llm/providers`.

### VIEW-006: Model Detail (parent: G-001)
- **Ovládací prvky:**
  - Detail modelu: `model_name`, `display_name`, `provider`, `state`, `context_window`, `loaded_at`, `last_error`.
  - Tlačítko „Load/Reload" → `POST /models/{id}/load`.
  - Tlačítko „Activate" (jen READY).
  - Edit Mode: úprava `display_name`, `context_window` (pokud provider nevrací).
  - Sekce „Test inference" — textový input + tlačítko „Send" → `POST /chat` (non-stream); zobrazení `content`, `usage`, `cached`, `latency_ms`.
  - Sekce „Token count" — textový input → `POST /count-tokens` live.
  - Sekce „Cache stats" — `hit_count`, `created_at` pro poslední cache zásahy modelu.
- **Stavy View:** LOADING, READY, ERROR (model.error), EDIT.
- **Data contract:** `GET /api/llm/models/{id}` + akční endpointy.

---

## 13. Security

- API klíče pro cloud providery: uloženy v lokálním secret store (šifrovaný soubor, klíč odvozen z OS keychain / master password). Nikdy v DB, nikdy v logu, nikdy v API response.
- `llm.provider.has_api_key` je pouze boolean indikátor (přítomnost), ne hodnota.
- Cloud volání (`openai` provider) opouští stroj — pouze uživatelem konfigurované; UI explicitně označí cloud modely příznakem „cloud".
- CORS a endpoint ochrana děděna z CORE (single-user, lokální).

---

## 14. Configuration (.env)

| Proměnná | Default | Popis |
|----------|---------|-------|
| `OLLAMA_BASE_URL` | `http://ollama:11434` | Ollama endpoint |
| `LMSTUDIO_BASE_URL` | `http://lm-studio-proxy:1234/v1` | LM Studio endpoint |
| `OPENAI_BASE_URL` | `https://api.openai.com/v1` | Cloud OpenAI-compatible endpoint |
| `OPENAI_API_KEY` | (z secret store) | API klíč — čten runtime, ne v .env plaintext |
| `LLM_INFERENCE_TIMEOUT_LOCAL` | `120` | Timeout lokální inference (s) |
| `LLM_INFERENCE_TIMEOUT_CLOUD` | `60` | Timeout cloud inference (s) |
| `LLM_CACHE_TTL_DAYS` | `7` | TTL inference cache |
| `LLM_CACHE_MAX_ENTRIES` | `10000` | Max záznamů před LRU eviction |

---

## 15. Implementation Tasks (odkaz)

Detailní tasky v PLANNING fázi (P-003). Pro G-001:
- IT-LLM-01: DB migrace (schéma `llm`, 4 tabulky, indexy, partial unique)
- IT-LLM-02: Provider abstrakce (`LLMProvider` interface + 3 implementace)
- IT-LLM-03: Model lifecycle manager (SM-LLM-01, atomic activation)
- IT-LLM-04: Inference proxy (chat, chat/stream, embed)
- IT-LLM-05: Cache (hash, TTL, LRU, hit tracking)
- IT-LLM-06: Token management (count-tokens, estimate, context_window check)
- IT-LLM-07: REST API (providers, models, load, activate, chat, embed, count-tokens)
- IT-LLM-08: Secret store integrace (API klíče šifrovaně)
- IT-LLM-09: Testy (AC-LLM-01..AC-LLM-14)

---

## 16. Definition of Done (S-004)

- [x] Data model specifikován (4 entity: provider, model, inference_cache, token_estimate)
- [x] Provider kontrakty specifikovány (Ollama, LM Studio, OpenAI-compatible + sjednocený interface)
- [x] State machine specifikována (SM-LLM-01, 8 přechodů)
- [x] Event model specifikován (7 eventů, konzistentní s CORE event busem)
- [x] API kontrakt specifikován (REST + SSE streaming)
- [x] Cache strategie specifikována (hash, TTL, LRU, bypass)
- [x] Token management specifikován (přesný/odhad, context window check)
- [x] Edge cases specifikovány (12 scénářů)
- [x] Acceptance criteria specifikována (14 měřitelných)
- [x] UI propojení specifikováno (VIEW-005, VIEW-006 s ovládacími prvky)
- [x] Security specifikována (API klíče šifrovaně)
- [x] Konfigurace specifikována (.env proměnné)
- [x] Lze implementovat bez domýšlení (konzistentní s DL-001..DL-004, GLOSSARY, S-003 CORE)

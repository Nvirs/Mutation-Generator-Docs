# SQL Mutation Generator – Tervezési és kódlogikai dokumentáció

## 1. Cél és hatókör

Ez a dokumentum a `Mutation-Generator` projekt architektúráját, komponenseit, adatfolyamait és kódlogikáját foglalja össze.

A rendszer fő célja:
- SQL Injection payloadok szisztematikus mutálása,
- mutált payloadok automatikus tesztelése célrendszeren (pl. DVWA),
- webes és REST API alapú használat támogatása,
- log-alapú forenzikai elemzés (SSE stream + LLM támogatás).

> Fontos: a rendszer kizárólag engedélyezett, izolált, oktatási / kutatási környezetben használható.

---

## 2. Magas szintű architektúra

A projekt három fő rétegre osztható:

1. **Core Engine (domain logika)**
   - `app/payload_database.py`
   - `app/payload_mutator.py`
   - `app/services/dvwa_client.py`
   - `app/services/payload_tester.py`

2. **Interfész réteg**
   - REST API: `app/api.py`
   - Log stream API: `app/log_routes.py`

3. **Megjelenítés / kliens**
   - webes UI: `web/` (HTML/CSS/JS)
   - dokumentáció és deploy segédanyagok: `docs/`, `deploy/`

---

## 3. Könyvtár- és modulstruktúra

### 3.1. Gyökérszint
- `config.py`: környezeti és futási konfigurációk
- `payloads.json`: payload adatbázis
- `requirements.txt`: függőségek

### 3.2. `app/` modul
- `payload_database.py`: payload adatforrás kezelés
- `payload_mutator.py`: mutációs stratégiák és transzformációk
- `api.py`: FastAPI endpointok a webes vezérléshez
- `log_routes.py`: log listázás és real-time stream (SSE)
- `vegpontok/`: generate, test, analyze endpoint implementációk
- `services/dvwa_client.py`: DVWA session és login kezelés
- `services/payload_tester.py`: payload futtatás és eredményklasszifikáció
- `services/log_streamer.py`: local/ssh log olvasás + SSE események
- `core/security.py`: token validáció, rate limiter, URL normalizáció

### 3.3. `web/` modul
- `payload_frontend/html/index.html`: fő payload UI
- `payload_frontend/js/script.js`: payload UI klienslogika
- `payload_frontend/css/styles.css`: payload UI stílusok
- `logstream_frontend/html/log_viewer.html`: log monitor felület
- `logstream_frontend/js/log_viewer.js`: log monitor klienslogika
- `logstream_frontend/css/log_viewer_styles.css`: log monitor stílusok

---

## 4. Konfigurációs modell (`config.py`)

A konfiguráció elsődleges forrása a `.env` fájl, a fallback értékek a kódban vannak.

### 4.1. Fontos konfigurációs csoportok

- **Mutációs stratégiák**: `MUTATION_STRATEGIES`
  - `encoding`
  - `comment_injection`
  - `whitespace`
  - `case`
  - `keyword_substitution`

- **Tesztelési paraméterek**: `TEST_CONFIG`
  - `target_url`, `login_url`, `username`, `password`
  - `timeout`, `retries`, `delay_between_requests`
  - `security_level`

- **API futtatás**
  - `API_HOST`, `API_PORT`

- **Log stream / SSH**
  - `SSH_CONFIG` (host, port, user, auth paraméterek)

- **LLM elemzés**
  - `LLM_CONFIG` (provider, base_url, model, temperature, timeout)

---

## 5. Core domain logika

## 5.1. PayloadDatabase (`app/payload_database.py`)

Felelősség:
- payload adatbázis beolvasása (`payloads.json`),
- fallback default payloadok biztosítása,
- kategória és statisztika lekérdezések,
- bővítés (`add_payload`).

Működés:
1. Inicializáláskor betölti a JSON-t.
2. Hiba esetén default payload készletet ad vissza.
3. Ha nincs fájl, létrehozza az alap payload adatbázist.

Fő metódusok:
- `get_payloads_by_category(category)`
- `get_all_payloads()`
- `get_categories()`
- `get_statistics()`
- `add_payload(category, payload)`

## 5.2. PayloadMutator (`app/payload_mutator.py`)

Felelősség:
- bemeneti payloadból több bypass-célú variáns létrehozása.

Kimeneti adatmodell (mutáció objektum):
- `original`: eredeti payload
- `mutated`: transzformált payload
- `technique`: alkalmazott technika neve
- `explanation`: rövid leírás

### Stratégiák és algoritmusok

1. **Encoding**
   - URL encoding
   - double URL encoding
   - Unicode formátum
   - hex reprezentáció
   - SQL `CHAR()` alapú csere

2. **Comment injection**
   - komment suffix (`--`, `#`, `/**/`, stb.)
   - whitespace -> komment csere
   - kulcsszó utáni komment beszúrás

3. **Whitespace variáció**
   - space -> tab
   - space -> newline
   - többszörös space
   - space -> `/**/`

4. **Case variáció**
   - lower
   - upper
   - mixed
   - random

## 5.3. PayloadTester (`app/services/payload_tester.py`)

Felelősség:
- HTTP session kezelés retry logikával,
- DVWA login és security level beállítás,
- payload végrehajtás és eredményklasszifikáció,
- összesített report.

Megjegyzés:
- A session/login és CSRF token kezelés a `DvwaSessionClient` osztályban történik (`app/services/dvwa_client.py`).

### Teszt logika 

1. (Opcionális) login és token-kinyerés
2. payload kérés elküldése GET paraméterrel
3. response elemzése:
   - blokkolás (`403`, `406`, mod_security szabályok)
   - sebezhetőség jelzők (SQL error stringek / siker indikátorok)
4. eredmény objektum összeállítása

Eredménymezők:
- `status_code`
- `response_time`
- `response_length`
- `blocked`
- `vulnerable`
- `errors`

### Report

A `generate_report()` számolja:
- összes teszt,
- blokkoltak,
- sebezhető válaszok,
- sikeres bypass esetek.

---

## 6. Legacy CLI réteg (kivezetve)

A korábbi `main.py` CLI belépési pont kivezetésre került.

Aktív interfészek:
- REST API: `app/api.py`
- Log stream API: `app/log_routes.py`
- Webes kliens: `web/`

A további adatfolyamok és működés az API/Web útvonalra vonatkoznak.

---

## 7. REST API réteg (`app/api.py`)

A FastAPI backend a web UI számára ad egységes endpointokat.

## 7.1. Fő endpointok

- `GET /` – health/info
- `GET /api/categories` – kategória lista
- `GET /api/config` – runtime defaultok UI-hoz
- `GET /api/payloads/{category}` – kategória payloadok
- `POST /api/generate` – mutáció generálás
- `POST /api/test` – payload futtatás cél URL-en
- `POST /api/analyze` – LLM-alapú logelemzés

## 7.2. Input validáció

A `pydantic` modellek végeznek védelmi validációt:
- payload hossz,
- strategy whitelist,
- URL séma (`http/https`),
- paraméternév regex,
- biztonsági szint whitelist.

## 7.3. URL normalizáció

A `normalize_dvwa_url()` funkció (`app/core/security.py`, wrapper: `app/services/dvwa_client.py`) korrigál tipikus Docker hibákat:
- `/dvwa/` útvonal egyszerűsítése,
- `localhost:API_PORT` → `dvwa` host konverzió konténeres futásnál.

## 7.4. LLM analyzer flow (`POST /api/analyze`)

Az elemző endpoint célja, hogy a nyers naplósorból reprodukálható, strukturált JSON választ állítson elő.

### 7.4.1. Feldolgozási lépések

1. A kliens elküldi a `log_line` és `http_status` mezőket.
2. A rendszer kliens-IP alapú rate limitet alkalmaz (`429` túl sok kérés esetén).
3. A bemenetet sanitizálja (`sanitize_log_data`), hogy a prompt ne tartalmazzon problémás karaktereket.
4. System + user prompt készül, majd LM Studio OpenAI-kompatibilis végpontra megy a kérés.
5. A válaszból JSON kinyerés történik (`_extract_json`):
   - közvetlen `json.loads`,
   - fenced code block parsing,
   - első `{ ... }` blokk kinyerése fallbackként.
6. A kliens egységes `AnalysisResponse` struktúrát kap `success`, `analysis/error`, `timestamp` mezőkkel.

### 7.4.2. Kódrészlet (rövidített, valós implementáció alapján)

```python
@router.post("/api/analyze", response_model=AnalysisResponse)
async def analyze_log_line(payload: AnalysisRequest, http_request: Request):
   client_key = (http_request.client.host if http_request.client else "") or "unknown"
   if rate_limiter.is_limited(client_key):
      raise HTTPException(status_code=429, detail="Analyze rate limit exceeded")

   safe_log_line = sanitize_log_data(payload.log_line)
   safe_http_status = sanitize_log_data(payload.http_status)

   request_payload = {
      "model": LLM_CONFIG["model"],
      "messages": [
         {"role": "system", "content": system_prompt},
         {"role": "user", "content": user_prompt},
      ],
      "temperature": LLM_CONFIG.get("temperature", 0.2),
   }

   lmstudio_response = requests.post(
      f"{LLM_CONFIG['base_url'].rstrip('/')}/chat/completions",
      headers={"Content-Type": "application/json"},
      json=request_payload,
      timeout=LLM_CONFIG.get("timeout_seconds", 30),
   )

   lmstudio_data = lmstudio_response.json()
   choices = lmstudio_data.get("choices", [])
   response_text = (choices[0].get("message", {}).get("content") or "").strip()

   analysis = _extract_json(response_text)
   return AnalysisResponse(success=True, analysis=analysis, timestamp=datetime.now().isoformat())
```

## 8. Log stream réteg (`app/log_routes.py`)

A logok real-time megfigyelése SSE (`text/event-stream`) alapon történik.

### 8.1. Üzemmódok

- `ssh`: távoli VM logok olvasása `asyncssh`-val
- `local`: lokális/shared volume fájlolvasás

### 8.2. Biztonsági elemek

- tokenes hozzáférés (`X-Log-Token` vagy query token)
- constant-time token ellenőrzés (`secrets.compare_digest`)
- whitelistelt logforrások

### 8.3. Stream folyamat

1. A kliens `source`, `tail`, `token` paraméterekkel hívja a stream endpointot.
2. A backend token-t ellenőriz, majd validálja, hogy a forrás szerepel-e a `LOG_SOURCES` whitelistben.
3. A rendszer ellenőrzi, hogy a logfájl olvasható-e (SSH vagy local módban).
4. A stream indulásakor `meta` esemény megy ki, utána a történeti `tail` sorok (`historic=true`).
5. Ezután egy szeparátor esemény jelzi az élő rész kezdetét.
6. Az élő követés `tail -f` analóg módon fut, keepalive eseményekkel.

### 8.4. Kódrészlet a stream indulásáról

```python
@router.get("/stream")
async def stream_log(source: str = Query(...), tail: int = Query(default=100, ge=1, le=1000), token: str = Query(...)):
   if not validate_token(token, stream_token):
      raise HTTPException(status_code=401, detail="Invalid or missing log stream token")

   if source not in LOG_SOURCES:
      raise HTTPException(status_code=404, detail=f"Unknown log source: {source}")

   log_path = streamer.effective_log_path(LOG_SOURCES[source]["path"])
   if not await streamer.check_file(log_path):
      raise HTTPException(status_code=404, detail="Log file not found or not readable")

   return StreamingResponse(
      streamer.stream_log_sse(log_path, source, tail),
      media_type="text/event-stream",
      headers={"Cache-Control": "no-cache", "Connection": "keep-alive", "X-Accel-Buffering": "no"},
   )
```

### 8.5. SSE eseménytípusok (kliens oldali értelmezéshez)

- `meta`: forrás, útvonal, host, mód, időbélyeg.
- `line`: egy log sor (`historic=true` vagy `false`).
- `separator`: vizuális határ a történeti és élő rész között.
- `error`: stream vagy kapcsolat hiba.
- `: keepalive`: kapcsolat életben tartása inaktív periódusok alatt.

---


## 9. Hibakezelés és megbízhatóság

- Retry képes HTTP session (`requests` + `urllib3.Retry`)
- Timeout kezelések minden külső kérésnél
- Globális FastAPI exception handler
- Strukturált naplózás központi logger konfigurációval (`app/core/logging.py`)
- JSON parse fallback az LLM válaszoknál

Korlátok:
- A detektálás szabályalapú; lehet false positive / false negative.
- Bizonyos payloadok DB-specifikusak (MySQL/MSSQL/PostgreSQL eltérés).



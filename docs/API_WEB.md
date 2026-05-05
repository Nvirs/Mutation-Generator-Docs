# API + Web UI Útmutató

## Áttekintés

A projekt ket kulon webes feluletet es egy FastAPI backendet tartalmaz:

- Payload UI: mutaciok generalasa + DVWA teszteles
- Log UI: valos ideju naplofigyeles (SSE) + LLM elemzes
- Backend: kozos API mindket felulethez

## Gyors índitas

### 1. Függősegek

```bash
pip install -r requirements.txt
```

### 2. Backend índitas

```bash
uvicorn app.api:app --host 127.0.0.1 --port 8000 --reload
```

### 3. Frontend índitas

```bash
cd web
python -m http.server 8001
```

Oldalak:

- Payload UI: `http://127.0.0.1:8001/payload_frontend/html/index.html`
- Log UI: `http://127.0.0.1:8001/logstream_frontend/html/log_viewer.html`

Megjegyzes: a `payload_frontend/js/script.js` jelenleg fixen a `http://127.0.0.1:8000` API cimet hasznalja.

## API végpontok (jelenlegi állapot)

### Alap

- `GET /`
  - API informacio es endpoint lista

### Payload kezelés

- `GET /api/categories`
  - payload kategoriak es elemszam

- `GET /api/payloads/{category}`
  - payload lista egy kategoriabol

- `GET /api/config`
  - UI-hoz adott default tesztelesi beallitasok (`target_url`, `login_url`, `username`, `security_level`)

- `POST /api/generate`
  - mutaciok generalasa

Request:

```json
{
  "base_payload": "' OR '1'='1",
  "strategies": ["encoding", "comment_injection"],
  "max_mutations": 50
}
```

Valasz (minta):

```json
{
  "success": true,
  "original_payload": "' OR '1'='1",
  "mutations": [
    {
      "original": "' OR '1'='1",
      "mutated": "%27%20OR%20%271%27%3D%271",
      "technique": "url_encoding",
      "explanation": "Karakterek URL kodolassal"
    }
  ],
  "total_count": 1,
  "timestamp": "2026-02-10T10:30:00"
}
```

### DVWA tesztelés

- `POST /api/test`
  - mutalt payloadok futtatasa a cel URL ellen

Request:

```json
{
  "target_url": "http://localhost/vulnerabilities/sqli/",
  "parameter": "id",
  "payloads": ["' OR '1'='1", "admin'--"],
  "login_url": "http://localhost/login.php",
  "username": "admin",
  "password": "password",
  "security_level": "low"
}
```

Valasz (minta):

```json
{
  "success": true,
  "results": [
    {
      "payload": "' OR '1'='1",
      "success": true,
      "status_code": 200,
      "response_length": 4567,
      "blocked": false,
      "vulnerable": true,
      "errors": []
    }
  ],
  "total_tested": 2,
  "successful_bypasses": 1,
  "timestamp": "2026-02-10T10:35:00"
}
```

### LLM analyze

- `POST /api/analyze`
  - egy naplosor strukturalt LLM elemzese
  - IP alapu rate limit van az endpointon

Request:

```json
{
  "log_line": "ModSecurity: Access denied with code 403 ... [id \"942100\"]",
  "http_status": "403",
  "timestamp": "2026-02-10T10:40:00"
}
```

Valasz (minta):

```json
{
  "success": true,
  "analysis": {
    "payload": "OR 1=1",
    "blocked": true,
    "rules_triggered": [942100],
    "libinjection_fingerprint": "s&sos",
    "total_anomaly_score": 8
  },
  "timestamp": "2026-02-10T10:40:01"
}
```

### Log stream

- `GET /api/logs`
  - logforrasok listazasa
  - token kotelezo (`X-Log-Token` header vagy `token` query)

- `GET /api/logs/stream`
  - SSE stream (`text/event-stream`)
  - kotelezo queryk: `source`, `tail`, `token`

## Modell-validáciok (backend)

### `MutationRequest`

- `base_payload`: 1..500 karakter, null byte tiltva
- `strategies`: whitelistelt strategiak
- `max_mutations`: 1..200

### `TestRequest`

- URL mezok: csak `http` vagy `https`
- `parameter`: csak `[a-zA-Z0-9_]`
- `payloads`: maximum 100 elem
- `security_level`: `low | medium | high | impossible`

### `AnalysisRequest`

- kotelezo: `log_line`, `http_status`
- opcionlis: `timestamp`

## Frontend működes röviden

### Payload UI (`web/payload_frontend`)

- kategoriak: `/api/categories`
- kategoriabeli payloadok: `/api/payloads/{category}`
- fallback mod: ha API nem erheto el, helyi `payloads.json` olvasasa
- generalas: `/api/generate`
- futtatas: `/api/test`

### Log UI (`web/logstream_frontend`)

- forrasok: `/api/logs`
- stream: `/api/logs/stream`
- soronkenti LLM elemzes gomb: `/api/analyze`

## Docker használatnál fontos

Kontenerben az API-bol a DVWA host neve altalaban `dvwa`, ezert tipikusan:

- `target_url`: `http://dvwa/vulnerabilities/sqli/`
- `login_url`: `http://dvwa/login.php`

## Hibaelharitás

### Failed to fetch

1. Ellenorizze, fut-e az API: `http://127.0.0.1:8000/`
2. Ellenorizze a browser konzolt (CORS vagy halozati hiba)
3. Ellenorizze, hogy a frontend nem masik geprol hivja a localhost API-t

### Log stream 401 (Unauthorized)

1. Ellenorizze a tokent
2. API ujrainditas utan random token valtozhat, ha nincs fix `LOG_STREAM_TOKEN`
3. Toltse ujra a forrasokat a log UI-ban

### API dokumentáció

- Swagger: `http://127.0.0.1:8000/docs`
- ReDoc: `http://127.0.0.1:8000/redoc`

## Verziós állápot

- API verzio: `1.0.1`
- Dokumentacio frissitve: `2026-04-15`

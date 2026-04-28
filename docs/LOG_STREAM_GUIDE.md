# Log Stream Guide

## Attekintes

A log stream funkcio a backendrol olvassa az Apache/MySQL/ModSecurity logokat, majd SSE (`text/event-stream`) formatumban tovabbitja a kliensnek.

Aktualis API vegpontok:

- `GET /api/logs`
- `GET /api/logs/stream`

Mindket vegpont token vedett.

## Futasi modok

### 1. `LOG_STREAM_MODE=ssh` (alapertelett)

- A backend `asyncssh` segitsegevel tavoli hostrol olvas logfajlokat.
- SSH beallitasokat a `config.py` `SSH_CONFIG` resze adja.

### 2. `LOG_STREAM_MODE=local`

- A backend lokalis fajlokat olvas.
- Docker demoban ez a javasolt:
  - `LOG_STREAM_MODE=local`
  - `LOG_STREAM_BASE_DIR=/shared-logs`

## Token mukodes

### Token forrasa

- Ha `LOG_STREAM_TOKEN` be van allitva, azt hasznalja.
- Ha nincs beallitva, a backend futaskor random tokent general (`secrets.token_hex(24)`), es kiirja a logba.

### Ellenorzes

Token a kovetkezo helyekrol jon:

- `X-Log-Token` header
- vagy `token` query parameter

A backend idofuggetlen osszehasonlitast hasznal (`secrets.compare_digest`).

## Log forrasok

Jelenlegi whitelistelt forrasok:

- `apache2_access` -> `/var/log/apache2/access.log`
- `apache2_error` -> `/var/log/apache2/error.log`
- `mysql_error` -> `/var/log/mysql/query.log`
- `modsec_audit` -> `/var/log/apache2/modsec_audit.log`

Megjegyzes: `local` modban az eleresi utak `LOG_STREAM_BASE_DIR` ala kepzodnek at.

## Helyi hasznalat (gyors teszt)

### 1. API inditas

```bash
uvicorn app.api:app --host 127.0.0.1 --port 8000 --reload
```

### 2. Frontend inditas

```bash
cd web
python -m http.server 8001
```

Log UI: `http://127.0.0.1:8001/logstream_frontend/html/log_viewer.html`

### 3. Token bemasolasa

- Ha fix token nincs beallitva, masold ki a backend konzolbol a generalt tokent.
- Illeszd be a Log UI `Hozzaferesi token` mezobe.

### 4. Forrasok betoltese

- `Forrasok betoltese` gomb -> `/api/logs`
- Ha 401, hibas token.
- Ha 200, megjelennek a forrasok elerhetosegi allapottal.

### 5. Stream csatlakozas

- Forras kivalasztasa
- `Csatlakozas` gomb -> `/api/logs/stream?source=...&tail=...&token=...`

## Docker demo (ajanlott)

A `deploy/vps-demo/docker-compose.yml` szerint az API kontener local modban olvas logot megosztott volume-bol.

Lenyeg:

- nem kell SSH kulcs a log streamhez
- API-hoz a logok read-only mounton latszanak
- a log UI ugyanugy tokennel csatlakozik

## cURL tesztek

### Forraslista

```bash
curl -H "X-Log-Token: <TOKEN>" "http://127.0.0.1:8000/api/logs"
```

### SSE stream

```bash
curl -N "http://127.0.0.1:8000/api/logs/stream?source=apache2_access&tail=10&token=<TOKEN>"
```

## SSE eseménytipusok

- `meta` - stream metadata
- `line` - egy log sor (`historic=true/false`)
- `separator` - torteneti es elo stream hatar
- `error` - stream hiba
- `: keepalive` - kapcsolat eletben tartasa

## CORS megjegyzes

Alapertelett CORS origin lista a backendben:

- `http://localhost:8000`
- `http://127.0.0.1:8000`
- `http://localhost:8001`
- `http://127.0.0.1:8001`
- `http://localhost:5500`
- `http://127.0.0.1:5500`
- `null`

Tovabbi origin hozzaadasa:

- `CORS_EXTRA_ORIGINS` kornyezeti valtozoban, vesszovel elvalasztva.

## Gyakori hibak

### `401 Unauthorized`

- rossz token
- API ujraindult, random token megvaltozott

### Minden forras `available=false`

- API olyan gepen fut, ahol a logfajlok nem erhetoek el
- rossz `LOG_STREAM_MODE`
- rossz `LOG_STREAM_BASE_DIR`
- SSH modban hibas SSH konfiguracio

### SSE kapcsolat megszakad

- API nem erheto el
- reverse proxy timeout
- rossz token vagy rossz source

## Biztonsagi ajanlas

- Produktion / demo kornyezetben allits fix `LOG_STREAM_TOKEN` erteket.
- A log stream vegpontokat ne tedd nyilvanos internetre vedelmi reteg nelkul.
- VPN vagy SSH tunnel hasznalata javasolt (lasd: `docs/VPS_DOCKER_DEMO.md`).

## Frissitesi datum

Dokumentacio frissitve: `2026-04-25`





# LLM modul pontos leírása

## 1. Cél és szerep a rendszerben

A projektben az LLM (Large Language Model) feladata nem a payload generálás, hanem a **naplóelemzés strukturált adatkimenetté alakítása**.

Az LLM a `POST /api/analyze` végponton keresztül dolgozik, és a ModSecurity/Apache log sorokból JSON mezőket próbál kinyerni.

Fő cél:
- nyers logszövegből gépileg feldolgozható mezők előállítása,
- SQLi detekcióhoz kapcsolódó jelzők normalizálása,
- későbbi statisztikai/forenzikai elemzés támogatása.

---

## 2. Hol van implementálva

- Endpoint logika: `app/vegpontok/analyze.py`
- Input/output modellek: `app/models/analysis.py`
- LLM és analyze biztonsági konfiguráció: `config.py`
- Input tisztítás: `app/core/logging.py` (`sanitize_log_data`)
- Rate limit: `app/core/security.py` (`InMemoryRateLimiter`)

---

## 3. Adatfolyam lépésről lépésre

1. A kliens meghívja a `POST /api/analyze` végpontot.
2. A kérés tartalma:
   - `log_line` (nyers naplóbejegyzés),
   - `http_status` (HTTP státusz vagy hibatípus),
   - opcionális `timestamp`.
3. Az API kliens-IP alapján rate limitet alkalmaz.
4. A bemenetet megtisztítja (`sanitize_log_data`): hosszlimit, null byte szűrés, nem nyomtatható karakterek eltávolítása.
5. A backend OpenAI-kompatibilis `chat/completions` hívást küld az LM Studio felé.
6. A modell válaszából JSON kinyerés történik (`_extract_json`).
7. Siker esetén strukturált `analysis`, hiba esetén `success: false` válasz kerül visszaadásra.

---

## 4. Konfiguráció (jelenlegi kulcsok)

### 4.1. LLM konfiguráció (`LLM_CONFIG`)

- `LLM_PROVIDER` (default: `lmstudio`)
- `LLM_BASE_URL` (default: `http://127.0.0.1:1234/v1`)
- `LLM_MODEL` (default: `local-model`)
- `LLM_API_KEY` (default: üres)
- `LLM_TEMPERATURE` (default: `0.2`)
- `LLM_TIMEOUT_SECONDS` (default: `30`)

### 4.2. Analyze biztonsági konfiguráció (`ANALYZE_SECURITY_CONFIG`)

- `ANALYZE_RATE_LIMIT_PER_MINUTE` (default: `30`)
- `ANALYZE_MAX_LOG_LINE_CHARS` (default: `2000`)
- `ANALYZE_MAX_LLM_RESPONSE_CHARS` (default: `12000`)

---

## 5. Promptstratégia

Az endpoint két üzenetet küld a modellnek:

1. **System prompt**
   - szerep: "ModSec Analyzer",
   - cél: OWASP CRS jellegű logokból strukturált mezők kinyerése,
   - kimeneti elvárás: csak JSON.

2. **User prompt**
   - a bemeneti logot és státuszkódot XML-szerű tagek közé teszi,
   - explicit módon tiltja a Markdown választ,
   - kifejezetten SQLi nyomozási célra kér elemzést.

A várt kulcsmezők:
- `payload`
- `blocked`
- `rules_triggered`
- `libinjection_fingerprint`
- `total_anomaly_score`

---

## 6. JSON-kinyerés működése

Az `_extract_json` több lépcsőben próbál parse-olni:

1. teljes válasz közvetlen `json.loads`,
2. ```json kódblokk tartalmának parse-olása,
3. általános ``` kódblokk parse,
4. első `{` és utolsó `}` közti rész parse.

Ha minden parse lépés hibás, a kód jelenleg egy **fallback** objektumot ad vissza mintaértékekkel.

Megjegyzés:
ez a fallback biztosítja, hogy legyen visszatérő struktúra,
de kutatási/statisztikai célra torzíthatja a mérést, mert nem valódi log-kinyerés eredmény.

---

## 7. API szerződés

### 7.1. Kérés (példa)

```json
{
  "log_line": "ModSecurity: Warning. Matched Data: OR 1=1 ... [id \"942100\"] ...",
  "http_status": "403",
  "timestamp": "2026-04-11T12:00:00"
}
```

### 7.2. Sikeres válasz (példa)

```json
{
  "success": true,
  "analysis": {
    "payload": "OR 1=1",
    "blocked": true,
    "rules_triggered": [942100, 949110],
    "libinjection_fingerprint": "s&sos",
    "total_anomaly_score": 8
  },
  "timestamp": "2026-04-11T12:00:01"
}
```

### 7.3. Tipikus hiba-válaszok

- `429` ha rate limit túllépés van.
- `success: false` + "LM Studio nem elérhető..." ha kapcsolat hiba történik.
- `success: false` + "Internal Analysis Error" egyéb futási hibáknál.

---

## 8. Biztonsági és megbízhatósági szempontok

Erősségek:
- rate limit az analyze végponton,
- bemeneti méretkorlát és karaktertisztítás,
- LLM válasz hosszkorlát.

Kockázatok:
- prompt injection jellegű log tartalom részben még befolyásolhatja a választ,
- a modellnév és a modellminőség külső tényező (LM Studio oldalon),
- fallback mintaértékek félrevezethetnek, ha nincs külön jelölve, hogy parse-hiba történt.

---

## 9. Javasolt üzemeltetési beállítások

- Strukturált adatkinyeréshez alacsony hőmérséklet: `0.0` - `0.2`.
- Stabil, instruction-tuned lokális modell használata.
- Válasz validáció bevezetése (pl. kötelező mezők és típusok ellenőrzése) a parse után.
- Külön metrika vezetése:
  - parse sikerességi arány,
  - fallback arány,
  - átlagos válaszidő,
  - hibaarány státuszkód szerint.

---


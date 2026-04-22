# Web UI Útmutató - SQLi Payload Mutator

## Áttekintés

A web alapú felület modern, böngészőben futó interfészt biztosít az SQLi Payload Mutator eszközhöz. Az architektúra FastAPI backendet és tiszta HTML/CSS/JavaScript frontendet használ.

### Biztonsági Tervezés

- **Elkülönítés**: A UI nem csatlakozik közvetlenül a célrendszerhez
- **Validáció**: Kliens és szerver oldali input validáció
- **Hitelesítés**: Opcionális DVWA login támogatás
- **Korlátozás**: CORS korlátozva localhost-ra
- **Védelem**: Nincs eval(), dinamikus kódvégrehajtás vagy OS parancs futtatás

## Telepítés

### 1. Függőségek telepítése

```bash
# Navigáljon a projekt könyvtárába
cd "projekt könyvtár"

# Telepítse a szükséges csomagokat
pip install -r requirements.txt
```

### 2. Backend elindítása

```bash
# FastAPI szerver indítása
cd app 
python api.py
```

Vagy közvetlenül uvicorn-nal:

```bash
uvicorn api:app --host 127.0.0.1 --port 8000 --reload
```

A szerver a következő címen lesz elérhető: `http://127.0.0.1:8000`

### 3. Frontend megnyitása

Egyszerűen nyissa meg az `index.html` fájlt egy böngészőben:

```bash
# Windows PowerShell
start index.html

# Vagy közvetlenül a böngészőből: File -> Open
```

**Megjegyzés**: A frontend működéséhez a backend szervernek futnia kell!

## Használat

### 1. Mutation Generálás

1. **Base Payload megadása**: Írja be egy alapvető SQL injection payloadot
   - Példa: `' OR '1'='1`
   - Maximum 500 karakter

2. **Mutation Strategies kiválasztása**:
   - URL Encoding
   - Comment Injection
   - Whitespace Variation
   - Case Variation


3. **Maximum Mutations**: Állítssa be a generálandó mutációk számát (1-200)

4. Kattintson a **"Generate Mutations"** gombra

### 2. Payload Tesztelés

1. **Target URL**: Adjon meg a DVWA végpont URL-jét
   - **Docker-ben (ajánlott)**: `http://dvwa/vulnerabilities/sqli/`
   - **Localhostos fejlesztésben**: `http://localhost:8080/vulnerabilities/sqli/`
   - **VPS-en**: `http://YOUR_VPS_IP:8080/vulnerabilities/sqli/` vagy `http://YOUR_DOMAIN:8080/vulnerabilities/sqli/`

2. **Injection Parameter**: Adjon meg a paraméter nevét
   - Példa: `id`

3. **Authentication (opcionális)**:
   - **Docker-ben (ajánlott)**: `http://dvwa/login.php`
   - **Localhostos fejlesztésben**: `http://localhost:8080/login.php`
   - Username: `admin`
   - Password: `password`

4. **Security Level**: Válassza ki a DVWA biztonsági szintjét
   - Low, Medium, High, vagy Impossible

5. Kattintson a **"Execute Against Target"** gombra

### Docker-kompatibilis Konfigurálás

Ha Docker Compose-ban futtatja a rendszert, az API konténer a `vulnerable_net` hálózaton van, ezért **a `localhost:8080` helyett a `dvwa` Docker hostname-t kell használni**.

**Helyesen (Docker-ben):**
- Target URL: `http://dvwa/vulnerabilities/sqli/`
- Login URL: `http://dvwa/login.php`

**Helytelenül (nem működik Docker-ben):**
- Target URL: `http://localhost:8080/vulnerabilities/sqli/` - *Connection refused*
- Login URL: `http://localhost:8080/login.php` - *Connection refused*

### 3. Eredmények értelmezése

Az eredmények különböző kategóriákba sorolva jelennek meg:

- **Success**: A payload sikeresen végrehajtódott
- **Failed**: A payload blokkolva lett vagy hibát okozott

Minden eredmény tartalmazza:
- A payloadot
- HTTP status kódot
- Válasz méretét
- Hibaüzeneteket (ha vannak)

## API Végpontok

A backend a következő REST API végpontokat biztosítja:

### `GET /`
Health check és API információk

### `GET /api/categories`
Elérhető payload kategóriák listája

### `GET /api/payloads/{category}`
Egy adott kategória payloadjainak lekérdezése

### `POST /api/generate`
Mutation generálás

**Request Body**:
```json
{
  "base_payload": "' OR '1'='1",
  "strategies": ["encoding", "comment_injection"],
  "max_mutations": 50
}
```

**Response**:
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
  "total_count": 50,
  "timestamp": "2025-12-13T10:30:00"
}
```

### `POST /api/test`
Payloadok tesztelése cél ellen

**Request Body**:
```json
{
  "target_url": "http://yourip/DVWA/vulnerabilities/sqli/",
  "parameter": "id",
  "payloads": ["' OR '1'='1", "admin'--"],
  "login_url": "http://yourip/DVWA/login.php",
  "username": "admin",
  "password": "password",
  "security_level": "low"
}
```

**Response**:
```json
{
  "success": true,
  "results": [
    {
      "payload": "' OR '1'='1",
      "success": true,
      "status_code": 200,
      "response_length": 4567
    }
  ],
  "total_tested": 2,
  "successful_bypasses": 1,
  "timestamp": "2025-12-13T10:35:00"
}
```

## Biztonsági Szempontok

### Input Validáció

**Kliens oldal (JavaScript)**:
- URL formátum ellenőrzés
- Paraméter név validáció (csak alfanumerikus)
- Payload hossz korlátozás (max 500 karakter)
- Null byte detektálás

**Szerver oldal (FastAPI + Pydantic)**:
- Típus ellenőrzés
- Értéktartomány validáció
- URL séma ellenőrzés (csak HTTP/HTTPS)
- Whitelist alapú strategy ellenőrzés

### Védelmek

1. **XSS Védelem**: `textContent` használata `innerHTML` helyett
2. **Code Injection Védelem**: Nincs `eval()`, dinamikus kódvégrehajtás
3. **CORS Védelem**: Korlátozva localhost-ra
4. **Rate Limiting**: Mutation és request limitek
5. **Error Handling**: Globális exception handler, nincs sensitive info leakage



## Hibaelhárítás

### "Failed to fetch" hiba

**Probléma**: A frontend nem tudja elérni a backendet

**Megoldás**:
1. Ellenőrizze, hogy a backend fut-e: `http://127.0.0.1:8000`
2. Nézze meg a böngésző konzolt (F12) CORS hibákért
3. Ellenőrizze a firewall beállításokat

### CORS hiba

**Probléma**: Cross-Origin Request Blocked

**Megoldás**:
- Nyissa meg az `index.html`-t a következő URL-ről: `http://localhost:8000` vagy `http://127.0.0.1:8000`
- Vagy használja egyszerű HTTP szervert:
  ```bash
  python -m http.server 8001
  # Majd nyissa meg: http://localhost:8001/index.html
  ```

### Payloadok nem működnek

**Probléma**: Minden payload "Failed" eredményt ad

**Megoldás**:
1. Ellenőrizze a DVWA VM IP címét
2. Ellenőrizze, hogy a DVWA fut-e és elérhető
3. Ellenőrizze a security level beállítást
4. Próbálja ki először a "low" security levelet
5. Ellenőrizze az authentication credentialokat

### Backend importálási hibák

**Probléma**: `ModuleNotFoundError`

**Megoldás**:
```bash
# Telepítse újra a függőségeket
pip install -r requirements.txt

# Vagy egyenként:
pip install fastapi uvicorn pydantic
```
## API Dokumentáció

A FastAPI automatikus API dokumentációt biztosít:

- **Swagger UI**: `http://127.0.0.1:8000/docs`
- **ReDoc**: `http://127.0.0.1:8000/redoc`

## Licenc és Felelősség
Ez az eszköz egyetemi szakdolgozati projekthez készült, kizárólag oktatási és kutatási célokra.

**Verzió**: 1.0.1  
**Utolsó frissítés**: 2026-04-13  
**Szerző**: Egyetemi szakdolgozati projekt

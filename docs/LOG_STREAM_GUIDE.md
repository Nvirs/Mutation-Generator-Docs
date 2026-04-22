# Log Stream – Működés és tesztelési útmutató

## 0. Futási módok (SSH vs Docker local)

A log stream backend kétféle módban tud futni:

- `LOG_STREAM_MODE=ssh` (alapértelmezett): távoli VM logolvasás `asyncssh`-val.
- `LOG_STREAM_MODE=local`: lokális fájlrendszer olvasás (pl. Docker shared volume).

Docker demóban (`deploy/vps-demo`) a javasolt beállítás:

- `LOG_STREAM_MODE=local`
- `LOG_STREAM_BASE_DIR=/shared-logs`

Ilyenkor az API a logokat közvetlenül a csatolt volume-ból olvassa,
nem használ `SSH_KEY_PATH`-ot.

## 1. Hogyan működik a `LOG_STREAM_TOKEN`?

### A probléma, amit megold

A `/api/logs` és `/api/logs/stream` végpontok szerver-oldali naplófájlokat
stremelnek a böngészőbe. Ezek a fájlok rendkívül érzékeny adatokat tartalmazhatnak
(munkamenet-sütik, jelszótöredékek, belső IP-cím, SQL-hibák), ezért ezeket csak
egy előzetesen egyeztetett titkos tokennel rendelkező kliens érheti el.


`secrets.token_hex(24)` az operációs rendszer kriptográfiai véletlenszám-generátorát
(`/dev/urandom` Linuxon, `CryptGenRandom` Windowson) használja – brute-force-sal
gyakorlatilag feltörhetetlen (2^192 lehetséges érték).

---

### Token validálás kérésenként

Minden `/api/logs` és `/api/logs/stream` kérésnél a FastAPI meghívja a
`require_log_token` dependency-t:

```
Böngésző kérés
      │
      ├─ HTTP header:  X-Log-Token: <token>   ← fetch() hívásokhoz (Load Sources)
      │
      └─ Query param:  ?token=<token>          ← EventSource-hoz, mert a böngésző
                                                 EventSource API nem tud custom
                                                 headert küldeni

require_log_token():
  candidate = header_token OR query_token
  if secrets.compare_digest(candidate, _LOG_STREAM_TOKEN):
      → OK, kérés folytatódik
  else:
      → HTTP 401 Unauthorized
```

> **Miért `secrets.compare_digest` és nem `==`?**
> A sima `==` összehasonlítás megáll az első eltérő karakternél, vagyis minél
> több karaktert talál el egy támadó, annál gyorsabban tér vissza. Ez
> *timing side-channel* támadást tesz lehetővé. A `compare_digest` mindig fix
> idő alatt fut le, karakterszámtól függetlenül.

---

### Token átadása a böngészőből

```
UI: [Access Token] [________________________] [Load Sources]
                             ↑ ide kerül a token

1. Load Sources gomb:
   fetch("/api/logs", { headers: { "X-Log-Token": token } })
   → Ha 401: "✗ Invalid token" üzenet
   → Ha 200: dropdown feltöltve, "✓ Token accepted"

2. Connect gomb (EventSource):
   new EventSource("/api/logs/stream?source=apache2_error&tail=100&token=<token>")
   → Ha 401: az EventSource onerror-ba esik, "⚠ Connection lost" megjelenik
```

---

## 2. Hálózati architektúra – Host-Only VM

### Miért nem működik, ha az API a Windows gépen fut?

A probléma egyszerű: a log-fájlok (`/var/log/apache2/...`) **a DVWA VM-en**
léteznek, nem a Windows fejlesztői gépen. Ha az API Windows-on fut, nem találja
ezeket a fájlokat → `available: false`.

```
  HELYTELEN                         HELYES
  ─────────                         ──────

  Windows gép                       Windows gép (192.168.xx.x)
  ┌─────────────────┐               ┌─────────────────┐
  │ Böngésző        │               │ Böngésző        │
  │ API (uvicorn)   │               │ Live Server     │
  │  /var/log → ✗   │               │ config.js ←─┐  │
  └────────┬────────┘               └──────┬───── │──┘
           │ Host-Only                     │ HTTP  │
  ┌────────┴────────┐               ┌──────┴───────┴──┐
  │ DVWA VM         │               │ DVWA VM          │
  │ /var/log → ✓    │               │ /var/log → ✓     │
  │                 │               │ API (uvicorn) ←  │
  └─────────────────┘               └──────────────────┘
  Logok itt vannak, API             API közvetlenül olvassa
  nem látja őket                    a logokat a VM-en
```

> A Host-Only adapter tökéletesen megfelel: Windows ↔ VM kommunikáció működik,
> internet nem szükséges.

---

## 3. Tesztelési útmutató

### Előfeltételek

| Szükséges | Leírás |
|---|---|
| Python ≥ 3.9 | **A DVWA VM-en** telepítve |
| FastAPI + uvicorn | `pip install -r requirements.txt` a VM-en |
| Mutation-Generator kód | VM-re másolva (lásd B. pont) |
| Host-Only hálózat | VM és Windows host látják egymást |
| Port 8000 nyitva | VM tűzfalában (`sudo ufw allow 8000/tcp`) |

---

### A. Gyors teszt – fejlesztői gépen (csak token mechanizmus)

Ha **egyedül a token-mechanizmust** akarod ellenőrizni Windows-on
(`available: false` lesz mindenhol – ez normális):

```bash
cd Mutation-Generator
uvicorn app.api:app --host 127.0.0.1 --port 8000 --reload
```

Konzolra megjelenik:
```
WARNING  LOG_STREAM_TOKEN env var is not set. ...
           LOG_STREAM_TOKEN = a3f9c1d8e4b27...4f91c3
```

Másold ki a tokent, illeszd be a böngésző mezőjébe, **Load Sources** →
`available: false` – helyes, az API Windows-on fut.

---

### B. A kód átmásolása a DVWA VM-re

```powershell
# Windows PowerShellből – SSH kell a VM-en: sudo apt install openssh-server
scp -r "C:\Users\ivasy\Documents\Szakdoga\Mutation-Generator" `
    dvwa@192.168.56.101:/home/dvwa/
```

Vagy git-tel a VM-en:
```bash
git clone <repository_url> ~/Mutation-Generator
```

---

### C. Teljes teszt – DVWA VM-en (tényleges stream)

#### 1. VM IP-jének meghatározása

```bash
# A VM-en:
ip a | grep 192.168
# pl.: inet 192.168.xx.xxx/24
```

```powershell
# Windows-on (ellenőrzés):
ping 192.168.xx.xx
```

#### 2. Python és függőségek telepítése (a VM-en)

```bash
sudo apt update && sudo apt install python3-pip python3-venv -y
cd ~/Mutation-Generator
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

#### 3. Port megnyitása a VM tűzfalában

```bash
sudo ufw allow 8000/tcp
```

#### 4. Persistent token beállítása (a VM-en)

```bash
# .bashrc végéhez add hozzá:
echo 'export LOG_STREAM_TOKEN="valami-titkos-jelszo"' >> ~/.bashrc
source ~/.bashrc
```

#### 5. CORS beállítása – Windows host IP hozzáadása

A Windows host IP-je (fut a böngésző):
```powershell
ipconfig | findstr 192.168
# pl.: 192.168.56.1
```

Nyisd meg az [app/api.py](../app/api.py) fájlt, és vedd ki a megjegyzésből:
```python
"http://192.168.xx.xx:5500",   # Windows host – Live Server
"http://192.168.xx.xx",
```

#### 6. web/config.js frissítése (Windows-on)

A [web/config.js](../web/config.js) az egyetlen fájl, amit a VM IP-jére kell állítani:

```javascript
window.MUTATION_API_URL = 'http://192.168.xx.xx:8000';
```

#### 7. API indítása a VM-en

```bash
cd ~/Mutation-Generator
source venv/bin/activate
uvicorn app.api:app --host 0.0.0.0 --port 8000
```

> `--host 0.0.0.0` kötelező – nélküle az API csak a VM loopbackjén hallgat
> és a Windows-os böngésző nem éri el.

Elvárt kimenet:
```
INFO  Log-stream token loaded from environment.
INFO  Uvicorn running on http://0.0.0.0:8000
```

#### 8. Apache2 logok generálása

```bash
# A VM-en vagy Windows PowerShellbol (192.168.56.101-gyel):
curl http://localhost/DVWA/
curl "http://localhost/DVWA/vulnerabilities/sqli/?id=%27&Submit=Submit"
curl http://localhost/nemletezik    # → error.log feltöltése
```

#### 9. Stream ellenőrzése curl-lel (VM-en)

```bash
curl -N \
  -H "X-Log-Token: valami-titkos-jelszo" \
  "http://localhost:8000/api/logs/stream?source=apache2_access&tail=10"
```

Elvárt kimenet:
```
data: {"type": "meta", "source": "apache2_access", ...}

data: {"type": "line", "text": "127.0.0.1 - - [21/Feb/2026:...] \"GET /DVWA/", "historic": true}

: keepalive
```

#### 10. Böngészős live stream (Windows-ról)

1. Nyisd meg `web/index.html`-t Live Server-rel
2. Token mezőbe: `valami-titkos-jelszo`
3. **Load Sources** → most `available: true` az Apache/MySQL forrásoknál
4. Válaszd **Apache2 Access Log** → **Connect**
5. Küldj kéréseket a DVWA-nak PowerShellből – azonnal megjelennek

```powershell
curl http://192.168.xx.xx/DVWA/
```





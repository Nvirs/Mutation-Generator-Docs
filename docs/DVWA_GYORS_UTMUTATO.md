# DVWA Gyors Telepítési Útmutató (Magyar)

## Rövid összefoglaló

Ez az útmutató segít elindítani a DVWA-t Ubuntu Desktopon, az SQLi Payload Mutator teszteléséhez.

## 1. Előfeltételek

- Ubuntu 22.04 vagy újabb (Desktop vagy Server)
- Internetkapcsolat
- Root vagy sudo jogosultságok
- Terminál hozzáférés (Terminal alkalmazás)

## 2. Telepítési módszerek

### Módszer A: Automatikus script (ajánlott)

```bash
# Script futtatása
sudo bash quick_start.sh
```

Ez a script automatikusan:
- Telepíti az Apache-t, MySQL-t, PHP-t
- Letölti és telepíti a DVWA-t
- Létrehozza az adatbázist
- Beállítja a konfigurációt

### Módszer B: Manuális telepítés

Lássa a részletes útmutatót: [`DVWA_SETUP.md`](DVWA_SETUP.md)

## 3. Telepítés utáni lépések

### 3.1. DVWA inicializálása

1. Nyissa meg böngészőben: `http://localhost/setup.php`
   - Ubuntu Desktop esetén használhatja a Firefox, Chrome vagy bármely más böngészőt
   - Ha távoli szerveren van: `http://[SERVER_IP]/setup.php`

2. Kattintson a **"Create / Reset Database"** gombra

3. Várjon, amíg az adatbázis létrejön 

### 3.2. Bejelentkezés

- **URL:** `http://localhost/login.php`
- **Felhasználónév:** `admin`
- **Jelszó:** `password`

### 3.3. Security Level beállítása

1. Bejelentkezés után menjen a **"DVWA Security"** menüpontra
2. Válassza ki a **"Low"** vagy **"Medium"** security levelt
3. Kattintson a **"Submit"** gombra

**Fontos:** SQL Injection teszteléshez legalább **"Low"** security level szükséges!

## 4. Az eszközünk használata

> Megjegyzés: a CLI kivezetésre került, ezért a tesztelés API végpontokon keresztül történik.

### Alapvető tesztelés

```bash
# API indítása
uvicorn app.api:app --host 127.0.0.1 --port 8000 --reload

# Mutációk generálása
curl -X POST http://127.0.0.1:8000/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "base_payload": "'"' OR '"'1'"'='"'1",
    "strategies": ["encoding", "comment_injection"],
    "max_mutations": 20
  }'
```

### Részletes tesztelés több stratégiával

```bash
curl -X POST http://127.0.0.1:8000/api/test \
  -H "Content-Type: application/json" \
  -d '{
    "target_url": "http://localhost/vulnerabilities/sqli/",
    "parameter": "id",
    "payloads": ["'"' UNION SELECT 1,2,3 --", "'"' OR '"'1'"'='"'1"],
    "login_url": "http://localhost/login.php",
    "username": "admin",
    "password": "password",
    "security_level": "low"
  }'
```

## 5. Hibaelhárítás

### Apache nem indul el

```bash
sudo systemctl status apache2
sudo systemctl restart apache2
```

### MySQL kapcsolati hiba

```bash
sudo systemctl status mysql
sudo systemctl restart mysql
```

### Nem tudok bejelentkezni

- Ellenőrizze, hogy a setup.php-t futtattad-e
- Próbálja meg újra inicializálni az adatbázist
- Ellenőrizze a MySQL szolgáltatást: `sudo systemctl status mysql`

### Az eszköz nem tud kapcsolódni

- Ellenőrizze, hogy a DVWA elérhető-e: `curl http://localhost/login.php`
- Ellenőrizze a bejelentkezési adatokat
- Próbálja meg a teljes URL-t: `http://localhost/vulnerabilities/sqli/`

## 6. További segítség

- Részletes útmutató: [`DVWA_SETUP.md`](DVWA_SETUP.md)
- DVWA hivatalos dokumentáció: https://github.com/digininja/DVWA
- Az eszköz dokumentációja: [`README.md`](README.md)


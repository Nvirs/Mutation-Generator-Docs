# DVWA Telepítési és Konfigurációs Útmutató

Ez az útmutató részletesen bemutatja, hogyan telepítse és konfigurálja a DVWA-t (Damn Vulnerable Web Application) Ubuntu Desktopon, Apache mod_security WAF-fal együtt.

## 1. lépés: Rendszerfrissítés és alapcsomagok telepítése

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y apache2 mysql-server php php-mysql php-gd libapache2-mod-php
```

## 2. lépés: MySQL konfigurálása

```bash
# MySQL biztonsági beállítások (opcionális, de ajánlott)
sudo mysql_secure_installation

# MySQL root felhasználó beállítása
sudo mysql -u root <<EOF
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'p@ssw0rd';
FLUSH PRIVILEGES;
EOF
```

**Megjegyzés:** A jelszót (`p@ssw0rd`) változtassa meg egy biztonságosra!

## 3. lépés: DVWA letöltése és telepítése

```bash
# Web könyvtárba navigálás
cd /var/www/html

# Régi tartalom eltávolítása (ha van)
sudo rm -rf *

# DVWA letöltése GitHub-ról
sudo git clone https://github.com/digininja/DVWA.git .

# Jogosultságok beállítása
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html
```

## 4. lépés: DVWA konfiguráció

```bash
# Konfigurációs fájl másolása
sudo cp /var/www/html/config/config.inc.php.dist /var/www/html/config/config.inc.php

# Konfigurációs fájl szerkesztése
sudo nano /var/www/html/config/config.inc.php
```

A fájlban módosítsa a következő részeket:

```php
$_DVWA[ 'db_server' ]   = '127.0.0.1';
$_DVWA[ 'db_database' ] = 'dvwa';
$_DVWA[ 'db_user' ]     = 'dvwa';
$_DVWA[ 'db_password' ]  = 'p@ssw0rd';
$_DVWA[ 'db_port'] = '3306';

// reCAPTCHA kulcsok (opcionális, teszteléshez nem szükséges)
$_DVWA[ 'recaptcha_public_key' ]  = '';
$_DVWA[ 'recaptcha_private_key' ] = '';
```

## 5. lépés: MySQL adatbázis létrehozása

```bash
sudo mysql -u root -p <<EOF
CREATE DATABASE dvwa;
CREATE USER 'dvwa'@'localhost' IDENTIFIED BY 'p@ssw0rd';
GRANT ALL ON dvwa.* TO 'dvwa'@'localhost';
FLUSH PRIVILEGES;
EOF
```

## 6. lépés: PHP beállítások

```bash
# PHP konfigurációs fájl szerkesztése
sudo nano /etc/php/8.1/apache2/php.ini
```

Keressen rá és módosítsa:

```ini
allow_url_include = On
allow_url_fopen = On
display_errors = On
display_startup_errors = On
```

**Megjegyzés:** A PHP verzió lehet 7.4, 8.0, 8.1 vagy 8.2 - ellenőrzés: `php -v`

## 7. lépés: Apache újraindítása

```bash
sudo systemctl restart apache2
sudo systemctl enable apache2
```

## 8. lépés: DVWA inicializálása

1. Nyissa meg böngészőben: `http://localhost/setup.php`
   - Ubuntu Desktop esetén használhatja a Firefox, Chrome vagy bármely más böngészőt
   - Ha távoli szerveren van: `http://[SERVER_IP]/setup.php`
2. Kattintson a "Create / Reset Database" gombra
3. Várjon, amíg az adatbázis létrejön

## 9. lépés: Bejelentkezés DVWA-ba

- **URL:** `http://localhost/login.php`
  - Ha távoli szerveren van: `http://[SERVER_IP]/login.php`
- **Felhasználónév:** `admin`
- **Jelszó:** `password`

## 10. lépés: mod_security telepítése (WAF)

```bash
# mod_security telepítése
sudo apt install -y libapache2-mod-security2

# Modul aktiválása
sudo a2enmod security2

# Alapértelmezett szabályok másolása
sudo cp /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf

# ModSecurity konfiguráció szerkesztése
sudo nano /etc/modsecurity/modsecurity.conf
```

Módosítsa a következő sort:

```conf
SecRuleEngine DetectionOnly
```

erre:

```conf
SecRuleEngine On
```

## 11. lépés: OWASP Core Rule Set telepítése (opcionális, de ajánlott)

```bash
# OWASP CRS letöltése
cd /tmp
sudo git clone https://github.com/coreruleset/coreruleset.git

# Szabályok másolása
sudo cp -r coreruleset/rules /etc/modsecurity/
sudo cp coreruleset/crs-setup.conf.example /etc/modsecurity/crs-setup.conf

# Apache konfiguráció módosítása
sudo nano /etc/apache2/mods-available/security2.conf
```

Adja hozzá a fájl végéhez:

```apache
<IfModule security2_module>
    Include /etc/modsecurity/crs-setup.conf
    Include /etc/modsecurity/rules/*.conf
</IfModule>
```

## 12. lépés: Apache újraindítása

```bash
sudo systemctl restart apache2
```

## 13. lépés: DVWA Security Level beállítása

1. Bejelentkezés után menjen a "DVWA Security" menüpontra
2. Válassza ki a "Low" vagy "Medium" security levelt
3. Kattintson a "Submit" gombra

**Fontos:** SQL Injection teszteléshez legalább "Low" security level szükséges!

## 14. lépés: Az eszközünk használata DVWA-val

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

### Részletes tesztelés

```bash
# Payloadok futtatása DVWA ellen
curl -X POST http://127.0.0.1:8000/api/test \
  -H "Content-Type: application/json" \
  -d '{
    "target_url": "http://localhost/vulnerabilities/sqli/",
    "parameter": "id",
    "payloads": ["'"' OR '"'1'"'='"'1", "admin'"'--"],
    "login_url": "http://localhost/login.php",
    "username": "admin",
    "password": "password",
    "security_level": "low"
  }'
```

### Mutációk mentése fájlba

```bash
# Mutációk lekérdezése API-ból JSON válaszként
curl -X POST http://127.0.0.1:8000/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "base_payload": "'"' UNION SELECT 1,2,3 --",
    "strategies": ["encoding", "whitespace", "case"],
    "max_mutations": 50
  }' > mutations.json
```

## Hibaelhárítás

### Apache nem indul el

```bash
# Apache státusz ellenőrzése
sudo systemctl status apache2

# Hibaüzenetek ellenőrzése
sudo tail -f /var/log/apache2/error.log
```

### MySQL kapcsolati hiba

```bash
# MySQL szolgáltatás ellenőrzése
sudo systemctl status mysql

# MySQL újraindítása
sudo systemctl restart mysql
```

### PHP hibaüzenetek

```bash
# PHP hibaüzenetek ellenőrzése
sudo tail -f /var/log/apache2/error.log

# PHP verzió ellenőrzése
php -v
```

### mod_security túl szigorú

Ha a mod_security túl sok kérést blokkol, ideiglenesen kikapcsolhatja:

```bash
sudo nano /etc/modsecurity/modsecurity.conf
# Módosítsd: SecRuleEngine DetectionOnly
sudo systemctl restart apache2
```

## További információk

- **DVWA hivatalos dokumentáció:** https://github.com/digininja/DVWA
- **mod_security dokumentáció:** https://github.com/SpiderLabs/ModSecurity
- **OWASP Core Rule Set:** https://coreruleset.org/

## Tesztelés

A telepítés sikerességének ellenőrzése:

```bash
# Apache státusz
sudo systemctl status apache2

# MySQL státusz
sudo systemctl status mysql

# Weboldal elérhetőség
curl http://localhost/login.php

# Az eszközünk tesztelése
curl http://127.0.0.1:8000/api/categories
```


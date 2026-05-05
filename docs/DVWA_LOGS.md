# DVWA Támadás Ellenőrzés Útmutató

## Áttekintés

Ez az útmutató megmutatja, hogyan ellenőrizheti, hogy a DVWA (Damn Vulnerable Web Application) helyesen fogadja-e a mutation generátor által küldött SQLi támadásokat Ubuntu környezetben.

## 1. Apache/Nginx Web Szerver Logok

### Apache Access Log (alapértelmezett)
```bash
# Real-time monitoring
sudo tail -f /var/log/apache2/access.log

# Csak SQLi kérések szűrése
sudo tail -f /var/log/apache2/access.log | grep -i "sql\|union\|select\|or\|and"
```

### Apache Error Log
```bash
# Hibaüzenetek figyelése
sudo tail -f /var/log/apache2/error.log
```

**Mit keresd:**
- `200 OK` státusz: A kérés sikeres volt
- `403 Forbidden`: WAF vagy biztonság blokkolta
- `500 Internal Server Error`: SQL szintaktikai hiba (ez jó jel SQLi szempontból!)

## 2. MySQL Query Logok

### Query Log Bekapcsolása

1. MySQL konfiguráció szerkesztése:
```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

2. Addja hozzá ezeket a sorokat:
```ini
[mysqld]
general_log = 1
general_log_file = /var/log/mysql/query.log
```

3. MySQL újraindítása:
```bash
sudo systemctl restart mysql
```

### Query Log Figyelése
```bash
# Real-time SQL query monitoring
sudo tail -f /var/log/mysql/query.log

# Csak SELECT, UNION, OR stb. keresése
sudo tail -f /var/log/mysql/query.log | grep -iE "SELECT|UNION|OR 1=1"
```

**Mit keresd:**
- Az általad küldött payload megjelenik a MySQL logban
- Szintaktikai hibák: `You have an error in your SQL syntax`
- Sikeres injekciók: Extra sorok a query eredményében

## 3. DVWA Saját Logok

### PHP Error Log
```bash
# PHP hibák figyelése
sudo tail -f /var/log/apache2/error.log | grep -i "php"
```

### DVWA Session/Tmp fájlok
```bash
# DVWA telepítési könyvtár (alapértelmezett)
cd /var/www/html/dvwa

# PHP session fájlok
ls -la /var/lib/php/sessions/
```

## 4. WAF (Web Application Firewall) Monitoring

### ModSecurity Státusz Ellenőrzése

```bash
# Ellenőrizze, hogy a ModSecurity be van-e kapcsolva
sudo apachectl -M | grep security

# Várható kimenet ha aktív:
# security2_module (shared)
```

### ModSecurity Konfiguráció Ellenőrzése

```bash
# ModSecurity főkonfiguráció
sudo nano /etc/modsecurity/modsecurity.conf

# Keressen rá ezekre a sorokra:
# SecRuleEngine On  <- WAF aktív (blokkol)
# SecRuleEngine DetectionOnly  <- Csak naplóz, NEM blokkol
# SecRuleEngine Off  <- WAF ki van kapcsolva

# SecStatusEngine On  <- Monitoring/status aktív (KELL!)
# SecStatusEngine Off  <- Monitoring ki
```

**Ha nincs blokkolás vagy "SecStatusEngine" hibaüzenetet látsz:**
```bash
# 1. Kapcsolja be a Status Enginet
sudo sed -i 's/SecStatusEngine Off/SecStatusEngine On/' /etc/modsecurity/modsecurity.conf

# Ha nincs ilyen sor, addja hozzá:
echo "SecStatusEngine On" | sudo tee -a /etc/modsecurity/modsecurity.conf

# 2. Állítsd át a Rule Engine-t "DetectionOnly"-ról "On"-ra
sudo sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' /etc/modsecurity/modsecurity.conf

# 3. Apache újraindítása
sudo systemctl restart apache2

# 4. Ellenőrzés
sudo grep -E "SecRuleEngine|SecStatusEngine" /etc/modsecurity/modsecurity.conf
```

### ModSecurity Audit Log Figyelése

```bash
# ModSecurity audit log helye (itt látoja a blokkolt kéréseket)
sudo tail -f /var/log/apache2/modsec_audit.log

# Vagy ha más helyen van:
sudo find /var/log -name "*modsec*"
```

**Audit log részletes nézet:**
```bash
# Csak a blokkolt kérések (403)
sudo grep -A 10 "HTTP/1.1\" 403" /var/log/apache2/modsec_audit.log

# Melyik szabály blokkolt
sudo grep "id \"" /var/log/apache2/modsec_audit.log | tail -20
```

### Apache Error Log - ModSecurity üzenetek

```bash
# ModSecurity blokkolási üzenetek real-time
sudo tail -f /var/log/apache2/error.log | grep -i "modsecurity"

# Példa kimenet ha blokkol:
# ModSecurity: Access denied with code 403 (phase 2). 
# Pattern match "(?i:select.*from)" at ARGS:id. [id "942100"]
```

### WAF Tesztelés

**1. Gyors teszt - Egyszerű SQLi:**
```bash
# Küldjon egy egyszerű SQLi payloadot
curl "http://localhost/dvwa/vulnerabilities/sqli/?id=1' OR 1=1-- -"

# Ha WAF működik: 403 Forbidden
# Ha WAF nem működik: 200 OK + adatok
```

**2. OWASP CRS szabály teszt:**
```bash
# XSS teszt
curl "http://localhost/test.php?q=<script>alert(1)</script>"

# SQLi teszt
curl "http://localhost/test.php?id=1 UNION SELECT"

# Mindkettőnél 403-at kellene kapnia ha a WAF aktív
```

**3. Részletes teszt curl-lel:**
```bash
# Láthatja a teljes választ (státuszkód, header, stb.)
curl -v "http://localhost/dvwa/vulnerabilities/sqli/?id=1' OR '1'='1"

# Keressen rá:
# < HTTP/1.1 403 Forbidden  <- WAF blokkolt
# < HTTP/1.1 200 OK  <- WAF nem blokkolt
```

### WAF Debug Mód

**ModSecurity debug log bekapcsolása:**

1. Szerkessze a konfigurációt:
```bash
sudo nano /etc/modsecurity/modsecurity.conf
```

2. Állítssa be ezeket a beállításokat:
```ini
# Status Engine (szükséges a monitoring működéséhez)
SecStatusEngine On

# Debug log beállítások
SecDebugLog /opt/modsecurity/var/log/debug.log
SecDebugLogLevel 9

# Vagy ha az /opt/modsecurity könyvtár nem létezik:
# SecDebugLog /var/log/apache2/modsec_debug.log
```

**Fontos:** Ha ezt a hibát látja: `SecStatusEngine to On`, akkor adja hozzá:
```bash
# Keressen rá a SecStatusEngine-re és állítsd On-ra
sudo grep -n "SecStatusEngine" /etc/modsecurity/modsecurity.conf

# Ha nem található, adja hozzá a fájl elejéhez
echo "SecStatusEngine On" | sudo tee -a /etc/modsecurity/modsecurity.conf
```

3. Hozzon létre a debug log könyvtárat:
```bash
# Ha /opt/modsecurity-t használja
sudo mkdir -p /opt/modsecurity/var/log
sudo chown www-data:www-data /opt/modsecurity/var/log

# Vagy alapértelmezett Apache log könyvtár
sudo mkdir -p /var/log/apache2
```

4. Apache újraindítás:
```bash
sudo systemctl restart apache2
```

5. Debug log figyelése:
```bash
#sud -etcsu Ha /opt/modsecurity-t használja 
sudo tail -f /opt/modsecurity/var/log/debug.log

# Vagy alapértelmezett
sudo tail -f /var/log/apache2/modsec_debug.log
```

### OWASP Core Rule Set (CRS) Ellenőrzése

```bash
# CRS szabályok helye
ls -la /usr/share/modsecurity-crs/rules/

# Ellenőrizze, hogy be vannak-e töltve
grep -r "Include.*modsecurity-crs" /etc/apache2/

# SQLi szabályok
cat /usr/share/modsecurity-crs/rules/REQUEST-942-APPLICATION-ATTACK-SQLI.conf | grep SecRule | head -10
```

### Gyakori WAF Problémák

**1. WAF DetectionOnly módban van:**
```bash
# Ellenőrzés
sudo grep "SecRuleEngine" /etc/modsecurity/modsecurity.conf

# Ha "DetectionOnly" látod, változtasd "On"-ra
```

**2. WAF szabályok nincsenek betöltve:**
```bash
# Apache konfiguráció ellenőrzése
sudo cat /etc/apache2/mods-enabled/security2.conf

# Kell lennie egy ilyen sornak:
# IncludeOptional /usr/share/modsecurity-crs/owasp-crs.load
```

**3. Whitelist/Exclusions blokkolja a tesztelést:**
```bash
# Keressen fehérlistát vagy exclusion-t
sudo grep -r "SecRuleRemoveById" /etc/modsecurity/
sudo grep -r "SecRuleRemoveByMsg" /etc/modsecurity/
```

### WAF Monitoring Parancs

**Egy terminálban mindent látni:**
```bash
# Apache error.log szűrve ModSecurity-re
sudo tail -f /var/log/apache2/error.log | grep --line-buffered -iE "modsecurity|403"

# Vagy ha van modsec_audit.log:
sudo tail -f /var/log/apache2/modsec_audit.log
```

## 5. Hálózati Forgalom Monitorozás

### tcpdump HTTP forgalom
```bash
# HTTP POST kérések figyelése (ha payload POST-ban megy)
sudo tcpdump -i any -A -s 0 'tcp port 80 and (tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354)'

# Egyszerűbb verzió - minden HTTP forgalom
sudo tcpdump -i any -A -s 0 'tcp port 80'
```

### netstat - Aktív kapcsolatok
```bash
# Látható kapcsolatok a 80-as porton
sudo netstat -an | grep :80
```

## 6. Gyakorlati Ellenőrzési Munkafolyamat

### Előkészületek (egyszer)

```bash
# Terminál 1: Apache access log
sudo tail -f /var/log/apache2/access.log

# Terminál 2: WAF/ModSecurity figyelés
sudo tail -f /var/log/apache2/error.log | grep -i "modsecurity"

# Terminál 3: MySQL query log (ha be van kapcsolva)
sudo tail -f /var/log/mysql/query.log

# Terminál 4: Apache error log
sudo tail -f /var/log/apache2/error.log
```

### Támadás Indítása

1. Indítsd el a mutation generátort:
```bash
curl -X POST http://127.0.0.1:8000/api/test \
   -H "Content-Type: application/json" \
   -d '{
      "target_url": "http://localhost/dvwa/vulnerabilities/sqli/",
      "parameter": "id",
      "payloads": ["'"' OR '"'1'"'='"'1"],
      "login_url": "http://localhost/dvwa/login.php",
      "username": "admin",
      "password": "password",
      "security_level": "low"
   }'
```

2. Figyelje a logokat real-time:
   - **Access log**: Látod a bejövő HTTP kéréseket
   - **MySQL log**: Látod a végrehajtott SQL query-ket
   - **Error log**: Látod a PHP/SQL hibákat

## 6. Automatizált Log Elemzés

### Gyors statisztika script
```bash
# Hány kérés érkezett az elmúlt 1 percben?
sudo tail -n 1000 /var/log/apache2/access.log | grep "sqli" | wc -l

# Hány volt sikeres (200)?
sudo tail -n 1000 /var/log/apache2/access.log | grep "sqli" | grep " 200 " | wc -l

# Hány volt blokkolt (403)?
sudo tail -n 1000 /var/log/apache2/access.log | grep "sqli" | grep " 403 " | wc -l
```

### SQL Error keresés
```bash
# SQL hibák az Apache error logban
sudo grep -i "sql syntax" /var/log/apache2/error.log | tail -20
```

## 7. DVWA Security Level Ellenőrzés

```bash
# Ellenőrizze a DVWA biztonsági szintjét
# Böngészőben: DVWA Security -> DVWA Security Level

# Low: Nincs védelem - minden payload átmegy
# Medium: Alapvető szűrés - sok payload blokkolt
# High: Strict szűrés - nagyon kevés payload megy át
# Impossible: Paraméteres query - semmi sem megy át
```


## Gyors Összefoglaló

**Minimum amit nézzen:**
```bash
# 1 terminál ablak
sudo tail -f /var/log/apache2/access.log | grep -i "sqli"
```
**Sikeres teszt jele**: Látja a kéréseket a logban, és 200-as státuszkódot kapja.





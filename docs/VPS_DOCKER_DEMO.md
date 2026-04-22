# VPS + Docker demó útmutató 

Ez az útmutató a projekt biztonságos bemutatásához készült: DVWA + ModSecurity egy VPS-en, az API külön konténerben, hozzáférés VPN-en és SSH-n keresztül.

## Cél architektúra

- `dvwa` konténer: csak belső Docker hálózaton (`vulnerable_net`) érhető el
- `api` konténer: csak loopback portra publikálva (`127.0.0.1:8000`)
- Külső elérés: kizárólag VPN + SSH tunnel / Reverse proxy
- Menedzsment: VS Code Remote SSH

## Ajánlott biztonsági modell
- VPS Linuxon fut a Docker stack
- a `dvwa` egyáltalán nem kap publikus portot
- az `api` csak a VPS loopback interfészére publishol
- adminisztráció kizárólag SSH kulccsal
- SSH elérés vagy VPN mögött legyen, vagy IP-re szűrve tűzfalon
- VS Code a VPS-re Remote SSH-val csatlakozik

Ha a legbiztonságosabb, mégis gyorsan beüzemelhető megoldást keresed, akkor:

- `Tailscale + SSH kulcs + UFW + fail2ban`

Ha nem akar külön mesh VPN-t, akkor a második legjobb opció:

- `SSH kulcs + UFW allow csak a saját fix IP-dre + fail2ban`

## Ha már most ezt használod

Az általad leírt jelenlegi állapot:

- a VPS szolgáltatói tűzfalán az inbound csak a te IP-d felől engedett
- outbound engedett IPv4 és IPv6 felé
- SSH csak kulccsal érhető el

Ez már önmagában is erős alap.

Ebben az esetben:

- a publikus SSH elérés kockázata jelentősen csökken, mert csak a te IP-dről próbálható el
- a jelszavas brute force gyakorlatilag ki van zárva, ha `PasswordAuthentication no`
- a DVWA és az API izolációját továbbra is a Docker hálózat és a loopback bind adja

Gyakorlati következtetés:

- ha a vizsgán is ugyanarról az engedélyezett IP-ről vagy stabil VPN végpontról fogsz dolgozni, a külön VPN nem feltétlenül kötelező
- ha a vizsga helyszínén más hálózatról leszel, a csak-saját-IP szabály problémát okozhat
- emiatt demóhoz a Tailscale továbbra is jó tartalék, mert nem függ a helyszíni publikus IP-től

Röviden: a mostani setupod már biztonságos, a VPN inkább rendelkezésre állási és kényelmi réteg, nem az elsődleges védelem.

## 1) VPS előkészítése

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose-plugin git ufw fail2ban
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

Jelentkezzen ki/be, hogy a Docker csoport tagság érvényesüljön.

Hozzon létre egy külön, nem root felhasználót, ha még nincs:

```bash
sudo adduser deployer
sudo usermod -aG sudo deployer
sudo usermod -aG docker deployer
```

Ezután a további lépéseket ezzel a normál userrel végezze.

## 2) SSH kulcsos belépés beállítása Windowsról

Windows PowerShellben hozz létre egy új kulcspárt:

```powershell
ssh-keygen -t ed25519 -a 100 -f $env:USERPROFILE\.ssh\szakdoga_vps
```

Ez két fájlt hoz létre:

- privát kulcs: `C:\Users\<te>\.ssh\szakdoga_vps`
- publikus kulcs: `C:\Users\<te>\.ssh\szakdoga_vps.pub`

A publikus kulcs tartalmát másold fel a VPS-re az adott user `authorized_keys` fájljába.

Ha még van jelszavas SSH hozzáférés, legegyszerűbb így:

```powershell
type $env:USERPROFILE\.ssh\szakdoga_vps.pub | ssh deployer@<VPS_IP> "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

Teszteld le a kulcsos belépést, mielőtt letiltod a jelszavas authot:

```powershell
ssh -i $env:USERPROFILE\.ssh\szakdoga_vps deployer@<VPS_IP>
```

## 3) SSH hardening a VPS-en

Nyisd meg:

```bash
sudo nano /etc/ssh/sshd_config
```

Legalább ezek legyenek beállítva:

```conf
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
X11Forwarding no
AllowUsers deployer
```

Ezután ellenőrzés és újraindítás:

```bash
sudo sshd -t
sudo systemctl restart ssh
```

Fontos: ezt csak akkor csinálja meg, ha a kulcsos belépés már működik.

## 4) Projekt klónozása és indítás

```bash
git clone <A_TE_REPOD_URL> Mutation-Generator
cd Mutation-Generator/deploy/vps-demo
docker compose build
docker compose up -d
```

### Konténeres live log monitor (SSH nélkül)

A `deploy/vps-demo/docker-compose.yml` beállítása szerint a `dvwa` konténer `/var/log`
mappája egy közös Docker volume-ban van, amit az `api` konténer read-only módon kap meg.

Ezért a live log monitor Docker demóban **nem SSH-t használ**, hanem lokális fájlolvasást:

- `LOG_STREAM_MODE=local`
- `LOG_STREAM_BASE_DIR=/shared-logs`

Következmény: a `SSH_KEY_PATH` / `SSH_HOST` értékek nem szükségesek ehhez a demóhoz,
és a korábbi "No such file or directory: .../.ssh/..." warning eltűnik.

Fontos: az API konténerből a DVWA cél URL-ek Docker belső DNS-en mennek, ezért
teszteléshez a `target_url` / `login_url` érték maradjon:

- `http://dvwa/vulnerabilities/sqli/`
- `http://dvwa/login.php`

Tailscale itt csak a kliens (böngésző) → API eléréshez kell, nem a konténeren belüli `api` → `dvwa` híváshoz.

Ha a projektet nem gitből viszi fel, akkor VS Code Remote SSH kapcsolaton belül is megnyithatod a mappát, és onnan futtathatod ugyanezeket a parancsokat.

## 5) Állapot ellenőrzése

```bash
docker compose ps
docker compose logs -f dvwa
docker compose logs -f api
```

API health check helyben:

```bash
curl http://127.0.0.1:8000/
```

Ellenőrizheted azt is, hogy a DVWA nincs publikus portra kitéve:

```bash
docker ps
ss -tulpn | grep 8000
```

Nem szabad olyat látnod, hogy a DVWA `0.0.0.0:80` vagy hasonló publikus bindon fut.

## 6) UFW minimál firewall

Csak SSH + VPN port legyen nyitva.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
# sudo ufw allow 51820/udp
sudo ufw enable
sudo ufw status verbose
```

Ha van fix otthoni IP-d, akkor még jobb ezt használni:

```bash
sudo ufw delete allow 22/tcp
sudo ufw allow from <SAJAT_IP>/32 to any port 22 proto tcp
```

Ha Tailscale-t használsz, akkor a publikus SSH portot akár teljesen zárva is hagyhatod.
Ha már a VPS szolgáltató saját firewallján csak a te IP-d van engedve, akkor az UFW itt második védelmi vonal. Ettől még érdemes bekapcsolni.

## 7) VPN ajánlás demóhoz

- Használj WireGuardot vagy Tailscale-t.
- A vizsgán a kliens gép VPN-en csatlakozik a VPS-hez.
- A `dvwa` nem publikus, csak belső hálózati névvel (`http://dvwa/...`) érhető el az API konténerből.

### Tailscale

Egyszerű és gyors demóhoz. Telepítés után a gép kapsz egy privát Tailscale IP-t.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Előnyök:

- nem kell külön WireGuard konfigurációt szerkeszteni
- a VPS nem lesz nyitva a teljes internet felé
- a VS Code Remote SSH csatlakozhat közvetlenül a Tailscale IP-re


## 7.1) Konkrétan a te esetedre (egyetemi hálózat, változó IP)

Mivel a vizsgán valószínűleg más publikus IP-ről fogsz csatlakozni, a provider firewallban beállított fix IP whitelist önmagában nem lesz elég.

Ebben a helyzetben javasolt:

- tartsd meg a jelenlegi provider firewall szabályokat (szigorú inbound)
- építs be Tailscale VPN elérést a VPS-re
- VS Code Remote SSH-t a Tailscale IP-re irányítsd

### Gyors Tailscale beállítás VPS-en

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh
tailscale ip -4
```

A `tailscale ip -4` kimenete lesz a VPS privát VPN címe (tipikusan `100.x.y.z`).

### Helyi gépen (Windows)

1. Telepítsd a Tailscale klienst.
2. Jelentkezz be ugyanabba a Tailscale networkbe.
3. Ellenőrizd, hogy látod a VPS-t a Tailscale admin felületen.

### SSH config Tailscale címre

`C:\Users\<te>\.ssh\config`:

```sshconfig
Host szakdoga-vps-ts
  HostName <TAILSCALE_100_X_Y_Z_IP>
  User deployer
  Port 22
  IdentityFile C:/Users/<te>/.ssh/szakdoga_vps
  IdentitiesOnly yes
  ServerAliveInterval 30
  ServerAliveCountMax 3
```

### Ellenőrzés

```powershell
ssh szakdoga-vps-ts
ssh -L 8000:127.0.0.1:8000 szakdoga-vps-ts
```

Majd böngészőből:

- `http://127.0.0.1:8000/`

Ha ez működik, a helyszíni IP-változás már nem fogja megakasztani a bemutatót.

## 8) VS Code Remote SSH használat Windowsról

### 8.1 Extension telepítése

Helyi VS Code-ban telepítsd:

- `Remote - SSH`

### 8.2 SSH config létrehozása Windows alatt

Az SSH config fájl tipikusan itt van:

- `C:\Users\<te>\.ssh\config`

Példa normál publikus IP-re:

```sshconfig
Host szakdoga-vps
  HostName <VPS_IP_VAGY_DOMAIN>
  User deployer
  Port 22
  IdentityFile C:/Users/<te>/.ssh/szakdoga_vps
  IdentitiesOnly yes
  ServerAliveInterval 30
  ServerAliveCountMax 3
```

Példa Tailscale IP-re:

```sshconfig
Host szakdoga-vps-ts
  HostName <TAILSCALE_100_X_Y_Z_IP>
  User deployer
  Port 22
  IdentityFile C:/Users/<te>/.ssh/szakdoga_vps
  IdentitiesOnly yes
```

### 8.3 Első csatlakozás VS Code-ból

1. `F1` vagy `Ctrl+Shift+P`
2. `Remote-SSH: Connect to Host...`
3. válaszd: `szakdoga-vps` vagy `szakdoga-vps-ts`
4. az első csatlakozásnál fogadd el a host key-t, ha ellenőrizted a szerver fingerprintjét
5. nyisd meg a távoli mappát: `/home/deployer/Mutation-Generator`

### 8.4 Mire számíts első használatkor

- a VS Code feltelepít egy távoli server komponenst a user home könyvtárába
- ez normális működés
- a terminál már a VPS-en fog futni
- innen tudod futtatni a `docker compose` parancsokat

### 8.5 Jó gyakorlatok Remote SSH-hoz

- ne root userrel dolgozz
- ne ments jelszót plain textben
- kulcsot passphrase-szel használj
- ha lehet, Tailscale IP-re csatlakozz, ne publikus IP-re
- a szerver fingerprintet első csatlakozás előtt külön ellenőrizd

## 9) API elérés a helyi gépről (SSH tunnel)

Mivel az API csak `127.0.0.1:8000`-on hallgat a VPS-en:

```bash
ssh -L 8000:127.0.0.1:8000 <SSH_USER>@<VPS_IP>
```

Ezután a helyi böngészőből/klienstől az API: `http://127.0.0.1:8000`.

Windows PowerShell példa a fenti configgal:

```powershell
ssh -L 8000:127.0.0.1:8000 szakdoga-vps
```

Ha Tailscale-t használsz:

```powershell
ssh -L 8000:127.0.0.1:8000 szakdoga-vps-ts
```

Ha Remote SSH-val már bent van VS Code-ban, az API-t a VPS termináljából is ellenőrizhető:

```bash
curl http://127.0.0.1:8000/
```

## 10) Manuális beüzemelési sorrend összefoglalva

1. VPS létrehozása Ubuntu LTS rendszerrel.
2. Normál user létrehozása `sudo` joggal.
3. SSH kulcspár generálása a Windows gépeden.
4. Publikus kulcs felmásolása a VPS-re.
5. Kulcsos belépés tesztelése.
6. `sshd_config` hardening és jelszavas login tiltása.
7. Docker, UFW, fail2ban telepítése.
8. Opcionálisan Tailscale telepítése.
9. VS Code Remote SSH csatlakozás.
10. Projekt felmásolása vagy klónozása.
11. `docker compose build` majd `docker compose up -d`.
12. API tesztelése lokál tunnelön keresztül.

Ha a mostani VPS-ed már fut és a firewall + SSH kulcs már kész, nálad a rövidített sorrend ez:
1. Ellenőrizd, hogy a kulcsos SSH továbbra is működik.
2. Ellenőrizd, hogy a publikus inbound továbbra is csak a te IP-dre szűrt.
3. Döntsd el, kell-e Tailscale a vizsgahelyszíni mobil eléréshez.
4. Remote SSH-val lépjen be a VPS-re.
5. Nyisd meg a projektet és indítsd a Docker stack-et.
6. Tunnellel ellenőrizd az API-t.

Megjegyzés: ha a provider firewall már a te IP-dre korlátoz, akkor a `VPN aktív` pont inkább ajánlott, nem minden esetben kötelező.




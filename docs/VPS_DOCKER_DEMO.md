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

A publikus kulcs tartalmát másolja fel a VPS-re az adott user `authorized_keys` fájljába.

Ha még van jelszavas SSH hozzáférés, legegyszerűbb így:

```powershell
type $env:USERPROFILE\.ssh\szakdoga_vps.pub | ssh deployer@<VPS_IP> "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

Tesztelje a kulcsos belépést, mielőtt letiltja a jelszavas authot:

```powershell
ssh -i $env:USERPROFILE\.ssh\szakdoga_vps deployer@<VPS_IP>
```

## 3) SSH hardening a VPS-en

Nyissa meg:

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

Ha a projektet nem gitből viszi fel, akkor VS Code Remote SSH kapcsolaton belül is megnyithatja a mappát, és onnan futtathatja ugyanezeket a parancsokat.

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

Ellenőrizheti azt is, hogy a DVWA nincs publikus portra kitéve:

```bash
docker ps
ss -tulpn | grep 8000
```

Nem szabad olyat látnia, hogy a DVWA `0.0.0.0:80` vagy hasonló publikus bindon fut.

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

Ha van fix otthoni IP-ja, akkor még jobb ezt használni:

```bash
sudo ufw delete allow 22/tcp
sudo ufw allow from <SAJAT_IP>/32 to any port 22 proto tcp
```

Ha Tailscale-t használja, akkor a publikus SSH portot akár teljesen zárva is hagyhatja.
Ha már a VPS szolgáltató saját firewallján csak az on Ip-je van engedve, akkor az UFW itt második védelmi vonal. Ettől még érdemes bekapcsolni.

## 7) VPN ajánlás demóhoz

- Használja WireGuardot vagy Tailscale-t.
- A `dvwa` nem publikus, csak belső hálózati névvel (`http://dvwa/...`) érhető el az API konténerből.

### Tailscale

Egyszerű és gyors demóhoz. Telepítés után a gép kapnia kéne egy privát Tailscale IP-t.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

## 8) VS Code Remote SSH használat Windowsról

### 8.1 Extension telepítése

Helyi VS Code-ban telepítse:

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
3. válassza: `szakdoga-vps` vagy `szakdoga-vps-ts`
4. az első csatlakozásnál fogadja el a host key-t, ha ellenőrzi a szerver fingerprintjét
5. nyissa meg a távoli mappát: `/home/deployer/Mutation-Generator`

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





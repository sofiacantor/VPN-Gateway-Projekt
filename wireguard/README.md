# WireGuard-konfiguration

Denna mapp innehåller **mallar** för WireGuard-konfiguration. Riktiga nycklar genereras av Ansible på gateway-VM:n och placeras aldrig i Git.

---

## Innehållsförteckning

- [Översikt](#översikt)
- [Filer i denna mapp](#filer-i-denna-mapp)
- [WireGuard — kort introduktion](#wireguard--kort-introduktion)
- [Konfigurationshistorik](#konfigurationshistorik)
- [Manuell konfiguration (referens)](#manuell-konfiguration-referens)
- [Automatiserad konfiguration (Ansible)](#automatiserad-konfiguration-ansible)
- [Viktig konfigurationsdetalj — två MASQUERADE-regler](#viktig-konfigurationsdetalj--två-masquerade-regler)
- [Klientkonfiguration på Windows](#klientkonfiguration-på-windows)
- [Felsökning och lärdomar](#felsökning-och-lärdomar)
- [Säkerhet och Git](#säkerhet-och-git)
- [Verifiering](#verifiering)

---

## Översikt

WireGuard är den centrala säkerhetskomponenten i projektet. Det fungerar som en krypterad tunnel mellan Windows-klienten och VPN-gateway-VM:n. När tunneln är aktiv kan klienten nå det interna nätverket (`192.168.56.0/24`). Utan tunneln är det interna nätverket osynligt.

**Tekniska detaljer:**

| Aspekt | Värde |
|--------|-------|
| Protokoll | WireGuard (UDP) |
| Port | 51820 |
| VPN-nätverk | 10.0.0.0/24 |
| Server (gateway) | 10.0.0.1/24 |
| Klient (Windows) | 10.0.0.2/24 |
| Kryptering | ChaCha20 + Poly1305 (asymmetrisk nyckelutbyte) |
| Port-forward | Vagrant: host:51820 → guest:51820 |

---

## Filer i denna mapp

```
wireguard/
├── wg0.conf.template        # Mall för server-konfiguration (gateway)
├── client.conf.template     # Mall för klient-konfiguration (Windows)
└── README.md                # Denna fil
```

### `wg0.conf.template`

Mall som visar **strukturen** för WireGuard-serverns konfiguration utan riktiga nycklar. Innehåller platshållare som `<SERVER_PRIVATE_KEY>` och `<CLIENT_PUBLIC_KEY>`.

### `client.conf.template`

Mall som visar strukturen för Windows-klientens konfiguration. Innehåller platshållare för båda nycklarna och rätt Endpoint-format.

---

## WireGuard — kort introduktion

WireGuard använder **asymmetrisk kryptografi** (samma princip som SSH-nycklar):

- Varje **sida** (klient och server) har ett eget nyckelpar
- **Privat nyckel** = HEMLIG, stannar på sin enhet
- **Publik nyckel** = OFFENTLIG, delas med motparten

### Var varje nyckel ligger

| Plats | I `[Interface]` | I `[Peer]` |
|-------|-----------------|------------|
| **Gateway** (`wg0.conf`) | Server PRIVAT | Klient PUBLIK |
| **Windows-klient** (`client.conf`) | Klient PRIVAT | Server PUBLIK |

> **Tankehjälp:** `[Interface]` är "MITT visitkort" (privat nyckel). `[Peer]` är "DEN ANDRAS visitkort" (publik nyckel).

### Vad varje sektion betyder

**På gateway:**

```ini
[Interface]                           # "JAG är gatewayen"
PrivateKey = <serverns privata>       # Min hemliga nyckel
Address = 10.0.0.1/24                 # Min IP i VPN-nätet
ListenPort = 51820                    # Vilken port jag lyssnar på
PostUp/PostDown = (iptables-regler)   # Sätt upp/ta ner NAT vid start/stopp

[Peer]                                # "Klienten är detta"
PublicKey = <klientens publika>       # Vem som får ansluta
AllowedIPs = 10.0.0.2/32              # Vilken IP klienten har
```

**På Windows-klient:**

```ini
[Interface]                           # "JAG är klienten"
PrivateKey = <klientens privata>      # Min hemliga nyckel
Address = 10.0.0.2/24                 # Min IP i VPN-nätet

[Peer]                                # "Gatewayen är detta"
PublicKey = <serverns publika>        # Vem jag ansluter till
Endpoint = 127.0.0.1:51820            # Vart paketen ska
AllowedIPs = 10.0.0.0/24, 192.168.56.0/24  # Vilka nät som routas via VPN
PersistentKeepalive = 25              # Håll anslutningen aktiv
```

---

## Konfigurationshistorik

WireGuard har konfigurerats på **två sätt** under projektets utveckling:

### Fas 1: Manuell konfiguration (Del 5)

Initialt installerades WireGuard manuellt med nyckelgenerering, iptables-regler och tjänstestart. Detta dokumenterades för att förstå alla steg och bli familiär med WireGuard.

### Fas 2: Automatiserad konfiguration (Del 9)

Hela installationen flyttades till Ansible-rollen `wireguard`. Vid `vagrant up` körs nu rollen automatiskt och hela konfigurationen sätts upp på under 5 minuter.

**Mallarna i denna mapp** (`wg0.conf.template`, `client.conf.template`) behålls som referens — Ansible använder en egen Jinja2-mall (`ansible/roles/wireguard/templates/wg0.conf.j2`) med variabler.

---

## Manuell konfiguration (referens)

> ⚠️ **Detta är historisk dokumentation från Del 5.** I projektets nuvarande tillstånd sker installationen automatiskt via Ansible — se nästa sektion.

### Steg 1: Logga in på gateway och bli root

```bash
vagrant ssh gateway
sudo -i
```

**Varför `sudo -i`:** WireGuard-mappen kräver `chmod 700` (bara root kan läsa). Med vanlig sudo blir det krångligt med rättigheter — root-shell förenklar.

### Steg 2: Installera WireGuard

```bash
apt update
apt install -y wireguard
```

### Steg 3: Generera nyckelpar

```bash
cd /etc/wireguard
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client_private.key | wg pubkey > client_public.key
chmod 600 server_private.key client_private.key
```

**Vad varje kommando gör:**
- `wg genkey` — genererar slumpmässig privat nyckel
- `tee fil.key` — skriver till fil OCH skickar vidare
- `| wg pubkey > publik.key` — räknar ut motsvarande publika nyckel

### Steg 4: Aktivera IP forwarding

```bash
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

Verifiera:

```bash
cat /proc/sys/net/ipv4/ip_forward
# Förväntat: 1
```

### Steg 5: Hämta nycklarna för konfigurationen

```bash
cat server_private.key
cat client_public.key
```

Kopiera dessa till `wg0.conf`-mallen i Steg 6.

### Steg 6: Skapa wg0.conf

Kopiera `wg0.conf.template` till `/etc/wireguard/wg0.conf` och ersätt platshållare:

```bash
cp /vagrant/wireguard/wg0.conf.template /etc/wireguard/wg0.conf
nano /etc/wireguard/wg0.conf
```

**Innehåll efter redigering:**

```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <din serverns privata nyckel>

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE; iptables -t nat -A POSTROUTING -o enp0s8 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o enp0s3 -j MASQUERADE; iptables -t nat -D POSTROUTING -o enp0s8 -j MASQUERADE

[Peer]
PublicKey = <din klientens publika nyckel>
AllowedIPs = 10.0.0.2/32
```

Sätt rättigheter:

```bash
chmod 600 /etc/wireguard/wg0.conf
```

### Steg 7: Starta WireGuard

```bash
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
systemctl status wg-quick@wg0
```

Förväntat: `Active: active (exited)` (grönt).

### Steg 8: Verifiera

```bash
wg                                  # Visar interface och peer
ip addr show wg0                    # Visar wg0 med IP 10.0.0.1/24
iptables -t nat -L POSTROUTING      # Visar MASQUERADE-reglerna
```

### Steg 9: Avsluta

```bash
exit  # Lämna root-shell
exit  # Lämna gateway-VM
```

---

## Automatiserad konfiguration (Ansible)

I projektets nuvarande version utför Ansible alla manuella steg automatiskt via rollen `wireguard`. Se [`../ansible/README.md`](../ansible/README.md) för detaljer.

**Sammanfattat utför Ansible:**

1. Installerar WireGuard
2. Skapar `/etc/wireguard/` med rättigheter `0700`
3. Genererar nyckelpar (idempotent — återanvänds vid omkörning)
4. Aktiverar IP forwarding
5. Renderar `wg0.conf` från Jinja2-mall med rätt iptables-regler för **båda** interfaces
6. Startar `wg-quick@wg0`-tjänsten
7. Genererar `client.conf` för Windows-klient

**Kör med:**

```bash
vagrant up               # Första gången
vagrant provision        # Vid omkörning
```

---

## Viktig konfigurationsdetalj — två MASQUERADE-regler

> **Detta är den viktigaste lärdomen från projektet!**

`wg0.conf` måste innehålla MASQUERADE-regler för **båda** utgående interfaces:

```bash
iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE   # För internet
iptables -t nat -A POSTROUTING -o enp0s8 -j MASQUERADE   # För internt nät
```

### Varför båda behövs

**`enp0s3` (NAT-interface)** — Behövs om VPN-klienten skulle skicka trafik mot internet via gateway.

**`enp0s8` (intnet-interface)** — Behövs för att VPN-klienten ska kunna nå `192.168.56.20` (internal-VM).

### Vad händer utan `enp0s8`-regeln

Utan MASQUERADE för `enp0s8` händer följande:

```
Klient (10.0.0.2)
    ↓ ping 192.168.56.20
WireGuard-tunnel
    ↓ paket inkommer i wg0 på gateway (källa: 10.0.0.2)
IP forwarding routar vidare till enp0s8
    ↓ paket lämnar enp0s8 (källa fortfarande 10.0.0.2!)
Internal-VM (192.168.56.20) tar emot paket
    ↓ vill svara
Internal-VM letar route till 10.0.0.2 → finns INGEN!
    ↓
SVAR DROPPAS — klienten får timeout
```

### Vad fixar `enp0s8`-regeln

Med MASQUERADE:

```
Klient (10.0.0.2)
    ↓ ping 192.168.56.20
WireGuard-tunnel
    ↓ paket inkommer i wg0 (källa: 10.0.0.2)
IP forwarding routar till enp0s8
    ↓ MASQUERADE ändrar källa: 10.0.0.2 → 192.168.56.10
Internal-VM tar emot paket (källa: 192.168.56.10) 
    ↓ svarar tillbaka till 192.168.56.10
Gateway tar emot svar och "translaterar tillbaka" till 10.0.0.2 
    ↓ skickar via WireGuard-tunnel
Klient får svar!
```

### Hur detta upptäcktes

Buggen upptäcktes under felsökning med `tcpdump`:

```bash
# På gateway
sudo tcpdump -i enp0s8 -n
```

Observerade att paket lämnade `enp0s8` med källan `10.0.0.2` istället för `192.168.56.10`. Detta avslöjade att MASQUERADE-regeln saknades. Efter tillägg av `enp0s8`-regeln fungerade pingen direkt.

> **Pedagogisk lärdom:** Detta är ett klassiskt nätverksproblem som inte är intuitivt för nybörjare. Det visar varför **felsökning med rätt verktyg** (`tcpdump`, `iptables -L`, `ip route`) är fundamental för nätverksdebugging.

---

## Klientkonfiguration på Windows

### Hämta klient-konfiguration från gateway

```bash
vagrant ssh gateway
sudo cat /etc/wireguard/client/client.conf
```

Markera HELA outputen → kopiera.

### Spara på Windows

```powershell
notepad C:\WireGuard-Client\client.conf
```

Klistra in, spara.

### Importera i WireGuard-appen

1. Öppna WireGuard-appen
2. Klicka **Lägg till tunnel** → **Importera tunneln från fil**
3. Välj `C:\WireGuard-Client\client.conf`
4. Klicka **Aktivera**

### Klient-konfigurationens viktiga detaljer

| Fält | Värde | Varför |
|------|-------|--------|
| `Address` | `10.0.0.2/24` | Klientens IP i VPN-nätet |
| `Endpoint` | `127.0.0.1:51820` | Vagrant port-forward — INTE 192.168.56.10! |
| `AllowedIPs` | `10.0.0.0/24, 192.168.56.0/24` | Vilka nät routas via VPN |
| `PersistentKeepalive` | `25` | Förhindrar NAT-tabell timeout |

> ⚠️ **Vanlig miss:** `Endpoint` ska vara `127.0.0.1:51820` (lokal port som forwardas), INTE `192.168.56.10:51820` (gatewayens interna IP — Windows saknar route dit).

---

## Felsökning och lärdomar

Under projektets utveckling stötte vi på flera klassiska WireGuard-problem. Detta är dokumentationen av vad som hände och hur vi löste det.

### Problem 1: ICMP-paket timar ut

**Symptom:** `ping 192.168.56.20` från Windows ger "Request timed out" trots aktivt VPN.

**Diagnos:**

```bash
sudo wg show
# Visade: handshake hade skett, paket räknades
# Men ingen route fungerade
```

```bash
sudo tcpdump -i enp0s8 -n
# Avslöjade att paket lämnade enp0s8 med källan 10.0.0.2
# Internal-VM hade ingen route tillbaka till 10.0.0.2
```

**Lösning:** Lägg till andra MASQUERADE-regel för `enp0s8` (se sektion ovan).

### Problem 2: Handshake fungerar inte (148/92-bytes mönster)

**Symptom:** `sudo wg` visar peer men ingen `latest handshake`. `transfer` ökar men slutförs aldrig.

**Diagnos:**

```bash
sudo tcpdump -i any -n udp port 51820
# Visade kontinuerliga 148-byte paket från klient
# Och 92-byte svar från gateway
# Men handshake slutfördes aldrig
```

148-byte = WireGuard handshake initiation, 92-byte = handshake response. Att de fortsätter i loop betyder att **nycklarna inte matchar**.

**Lösning:** Verifiera att klient-publik-nyckel som gateway har som peer **matchar** den faktiska publika nyckeln som motsvarar klientens privata nyckel:

```bash
# På gateway
sudo cat /etc/wireguard/client_private.key | wg pubkey
# Detta ska matcha det som står på Windows som "Publik nyckel" i [Interface]
```

### Problem 3: "General failure" från ping

**Symptom:** `ping 10.0.0.1` på Windows ger "General failure" — inte ens timeout.

**Diagnos:**

```powershell
route print 10.0.0.*
# Verifierade att routes finns
```

```powershell
Get-NetConnectionProfile -InterfaceAlias "client"
# Visade kategori = Public
```

**Lösning:** WireGuard-adaptern var i "Public network"-kategori. Windows Defender Firewall blockerar ICMP på Public-nätverk.

```powershell
# Som administratör
Set-NetConnectionProfile -InterfaceAlias "client" -NetworkCategory Private
```

### Problem 4: Endpoint pekar fel

**Symptom:** Klient ansluter aldrig till gateway. Inga paket i `tcpdump`.

**Diagnos:**

```powershell
Get-Content C:\WireGuard-Client\client.conf
# Visade: Endpoint = 192.168.56.10:51820
```

Detta är fel — Windows har ingen route till `192.168.56.0/24`.

**Lösning:** Ändra till:

```ini
Endpoint = 127.0.0.1:51820
```

Vagrant port-forward gör att Windows ansluter till lokal port 51820, som forwardas till gateway-VM:s port 51820.

### Problem 5: Klocksynk-problem under installation

**Symptom:** `apt install` misslyckas med fel om certifikat eller "release file from future".

**Diagnos:** VM:n har gått tillbaka i tiden efter `vagrant halt` + `vagrant up`.

**Lösning:**

```bash
sudo systemctl restart systemd-timesyncd
sleep 10
sudo apt update
```

Ansible-rollen för nginx gör detta automatiskt som första steg.

---

## Säkerhet och Git

### Vad som ALDRIG ska finnas i Git

- Privata nycklar (`*.key`-filer)
- Publika nycklar med riktiga värden (`*.pub`-filer)
- Riktiga `wg0.conf` (med faktiska nycklar)
- Riktiga `client.conf` (med faktiska nycklar)

### Vad SOM får finnas i Git

- Mallar (`*.template`) — visar struktur utan riktiga värden
- Ansible-rollen som **genererar** nycklar
- Jinja2-templates med variabler

### .gitignore-skydd

```
# WireGuard-nycklar och konfigurationer
*.key
*.pub
privatekey
publickey

# WireGuard-konfigurationer (utom mallar)
wireguard/*.conf
!wireguard/*.template

# Ansible-genererade hemligheter
ansible/**/client.conf
ansible/**/*.key
```

### Verifiera att nycklar aldrig läckt

```bash
git log --all -- "**/*.key"           # Ska vara tom
git log --all -- "**/client.conf"     # Ska vara tom
git log --all -- "**/wg0.conf"        # Ska vara tom (utom .template)
```

Om något visas — be om hjälp innan du pushar mer!

---

## Verifiering

### På gateway-VM

```bash
vagrant ssh gateway

# WireGuard-tjänsten kör?
sudo systemctl status wg-quick@wg0

# wg0-interface uppe med rätt IP?
ip addr show wg0
# Förväntat: inet 10.0.0.1/24

# Lyssnar på port 51820 UDP?
sudo ss -lunp | grep 51820

# Peer konfigurerad?
sudo wg

# iptables NAT-regler korrekta?
sudo iptables -t nat -L POSTROUTING -n -v
# Förväntat: TVÅ MASQUERADE-rader (enp0s3 och enp0s8)

# IP forwarding aktiverat?
cat /proc/sys/net/ipv4/ip_forward
# Förväntat: 1
```

### På Windows-klient

```powershell
# WireGuard-adapter uppe?
ipconfig | findstr "10.0.0.2"

# Routes till VPN-nät?
route print 10.0.0.*
route print 192.168.56.*

# Ping till gateway?
ping 10.0.0.1

# Ping till internal-VM (via VPN)?
ping 192.168.56.20

# Webbsidan visas?
curl http://192.168.56.20
```

### Verifiera handshake

```bash
vagrant ssh gateway
sudo wg
```

Förväntat efter VPN-anslutning:

```
peer: <klientens publika nyckel>
  endpoint: 10.0.2.2:XXXXX
  allowed ips: 10.0.0.2/32
  latest handshake: X seconds ago         ← KRITISK
  transfer: X B received, Y B sent        ← Ska öka
```

---

## Referenser

- Huvud-README: [`../README.md`](../README.md)
- Ansible WireGuard-roll: [`../ansible/roles/wireguard/`](../ansible/roles/wireguard/)
- Arkitekturbeskrivning: [`../docs/arkitektur.md`](../docs/arkitektur.md)
- Säkerhetstester: [`../docs/tester.md`](../docs/tester.md)
- [WireGuard officiell sida](https://www.wireguard.com/)
- [WireGuard Quick Start](https://www.wireguard.com/quickstart/)
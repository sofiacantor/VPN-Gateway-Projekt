# VPN-Gateway Projekt

> En automatiserad och säker VPN-lösning där en intern webbserver endast är åtkomlig via WireGuard VPN. Hela miljön sätts upp automatiskt med Vagrant och Ansible — ett kommando räcker för att skapa två virtuella maskiner, konfigurera VPN-gateway, generera nycklar och driftsätta tjänsten.

---

## Innehållsförteckning

- [Arkitektur](#arkitektur)
- [Miljöer och IP-adresser](#miljöer-och-ip-adresser)
- [Mappstruktur](#mappstruktur)
- [Komponenter](#komponenter)
- [Krav och förutsättningar](#krav-och-förutsättningar)
- [Kom igång](#kom-igång)
- [Secrets](#secrets)
- [Säkerhetsåtgärder](#säkerhetsåtgärder)
- [Säkerhetsanalys](#säkerhetsanalys)
- [Verifiering](#verifiering)
- [Designval och motivering](#designval-och-motivering)

---

## Arkitektur

![Arkitekturdiagram](docs\diagrams\arkitektur.drawio.png)

Projektet består av tre logiska zoner:

1. **Windows-host** — användarens dator med WireGuard-klient
2. **VPN-Gateway VM** — fungerar som "dörrvakt" till det interna nätverket
3. **Internal VM** — den skyddade webbservern

WireGuard VPN-tunneln är den **enda** vägen in till det interna nätverket. Utan en aktiv VPN-anslutning är `192.168.56.20` helt onåbart från Windows-hosten.

```
Windows-host (din dator)
        |
        | WireGuard VPN-tunnel (krypterad UDP/51820)
        | via Vagrant port-forward: 127.0.0.1:51820
        |
┌───────▼──────────────────────────────────┐
│       VirtualBox-container                │
│                                           │
│  ┌─────────────────────┐                  │
│  │  Gateway VM         │                  │
│  │  wg0:  10.0.0.1/24  │                  │
│  │  e8:   192.168.56.10│                  │
│  │  WireGuard + iptables                  │
│  └────────┬────────────┘                  │
│           │ Internt nätverk               │
│           │ (intnet 192.168.56.0/24)     │
│           ▼                               │
│  ┌─────────────────────┐                  │
│  │  Internal VM        │                  │
│  │  e8: 192.168.56.20  │                  │
│  │  nginx (port 80)    │                  │
│  │  (ingen port fwd)   │                  │
│  └─────────────────────┘                  │
└──────────────────────────────────────────┘
```

För detaljerad arkitekturbeskrivning, se [docs/arkitektur.md](docs/arkitektur.md).

---

## Miljöer och IP-adresser

| VM | Roll | IP-adress | Port forwarding | Beskrivning |
|---|---|---|---|---|
| `gateway` | VPN-Gateway | 192.168.56.10 (intnet)<br>10.0.0.1/24 (wg0) | `UDP 51820 → host:51820` | WireGuard-server som krypterar/dekrypterar trafik och routar via NAT |
| `internal` | Webbserver | 192.168.56.20 (intnet) | — | nginx på port 80, ej direkt nåbar utifrån |

### Nätverkssegmentering

Tre logiskt separerade nätverk:

| Nätverk | IP-område | Funktion |
|---------|-----------|----------|
| **VirtualBox NAT** | 10.0.2.0/24 | Internet för VMs (automatiskt) |
| **VPN-nätverk** | 10.0.0.0/24 | Krypterad WireGuard-tunnel |
| **Internt nätverk** | 192.168.56.0/24 | Mellan VMs (`intnet`, isolerat från host) |

---

## Mappstruktur

```
VPN-Gateway-Projekt/
├── Vagrantfile                          # Definierar båda VMs och Ansible-provisionering
├── README.md                            # Denna fil
├── .gitignore                           # Skyddar nycklar och hemligheter från Git
│
├── ansible/                             # Ansible-automation
│   ├── ansible.cfg                      # Ansible-konfiguration (inventory, roller, mm)
│   ├── inventory.ini                    # Definierar gateway och internal som lokala targets
│   ├── site.yml                         # Master playbook — kör alla roller i ordning
│   ├── README.md                        # Beskrivning av Ansible-strukturen
│   │
│   └── roles/
│       ├── wireguard/                   # Installerar och konfigurerar WireGuard VPN
│       │   ├── tasks/
│       │   │   └── main.yml             # 18 tasks: install, nycklar, NAT, tjänster
│       │   ├── handlers/
│       │   │   └── main.yml             # Restart WireGuard vid konfigändringar
│       │   └── templates/
│       │       └── wg0.conf.j2          # Jinja2-mall för WireGuard-konfiguration
│       │
│       └── nginx/                       # Installerar och konfigurerar webbservern
│           ├── tasks/
│           │   └── main.yml             # Apt install, kopiera HTML, starta tjänst
│           ├── handlers/
│           │   └── main.yml             # Restart nginx vid filändringar
│           └── files/
│               ├── illustration.png     # Bild som visas på webbsidan
│               └── index.html           # Anpassad webbsida för intern tjänst
│
├── webserver/                           # Källfiler för webbservern
│   ├── illustration.png                 # Bild för webbsidan
│   ├── index.html                       # HTML-källfil (kopieras till nginx-rollen)
│   └── README.md                        # Beskrivning av webbserver-källan
│
├── wireguard/                           # Mallar för WireGuard-konfiguration
│   ├── client.conf.template             # Mall för klient (utan riktiga nycklar)
│   ├── wg0.conf.template                # Mall för server (utan riktiga nycklar)
│   └── README.md                        # Beskrivning av WireGuard-konfigurationen
│
└── docs/                                # All projektdokumentation
    ├── arkitektur.md                    # Detaljerad arkitekturbeskrivning
    ├── tester.md                        # Säkerhetstester och resultat
    │
    ├── diagrams/                        # Arkitekturdiagram
    │   └── arkitektur.drawio.png # Diagrammet (skapat i draw.io)
    │
    └── screenshots/                     # Skärmdumpar från säkerhetstester
        ├── test1-utan-vpn-ping-fail.png             # Test 1: ping misslyckas utan VPN
        ├── test1-med-vpn-ping-success.png           # Test 1: ping fungerar med VPN
        ├── Test1-med-vpn+ansible-ping-fail.png      # Test 1: ansible-test som misslyckas
        ├── Test1-med-vpn+ansible-ping-success.png   # Test 1: ansible-test som lyckas
        ├── test2-utan-vpn-browser-fail.png          # Test 2: browser misslyckas utan VPN
        ├── test2-med-vpn-browser-success.png        # Test 2: browser fungerar med VPN
        ├── Test2-med-vpn+ansible-browser-fail.png   # Test 2: ansible-test misslyckas
        ├── Test2-med-vpn+ansible-browser-success.png # Test 2: ansible-test lyckas
        ├── test3-utan-vpn-curl-fail.png             # Test 3: curl misslyckas utan VPN
        ├── test3-med-vpn-curl-success.png           # Test 3: curl fungerar med VPN
        ├── Test3-med-vpn+ansible-curl-fail.png      # Test 3: ansible-test misslyckas
        └── Test3-med-vpn+ansible-curl-success.png   # Test 3: ansible-test lyckas
```

---

## Komponenter

### Vagrantfile

Definierar två virtuella maskiner i VirtualBox:

- **Gateway VM** — Ubuntu 22.04, 1024 MB RAM, NAT + intnet + WireGuard port-forward (UDP 51820)
- **Internal VM** — Ubuntu 22.04, 512 MB RAM, NAT + intnet

Båda VMs provisioneras med `ansible_local` — Ansible körs **inuti** varje VM, inte på Windows-hosten. Detta gör projektet portabelt — ingen Ansible-installation krävs på värddatorn.

### ansible.cfg

Pekar på `inventory.ini`, sätter `host_key_checking = False` (lämpligt i labbmiljö) och anger att roller finns i `./roles`-katalogen. Använder `stdout_callback = yaml` för läsbar output.

### inventory.ini

Grupperar VMs i `[gateway]` och `[internal]` med `ansible_connection=local`. Detta innebär att Ansible kör tasks direkt i VM:n istället för via SSH — snabbare och säkrare.

### site.yml

Master playbook med två plays:

1. **gateway** — kör WireGuard-rollen som installerar och konfigurerar VPN
2. **internal** — kör nginx-rollen som installerar webbservern

### Rollen wireguard

Innehåller ~18 tasks som:

1. Installerar WireGuard via apt
2. Skapar `/etc/wireguard` med rättigheter `0700`
3. Genererar serverns och klientens nyckelpar (idempotent — återanvänds om de redan finns)
4. Aktiverar IP forwarding (`net.ipv4.ip_forward = 1`)
5. Renderar `wg0.conf` från Jinja2-mall med korrekta nycklar och iptables-regler för **båda** nätverkskort (NAT och intnet)
6. Startar och aktiverar `wg-quick@wg0`-tjänsten
7. Genererar `client.conf` för Windows-klienten

### Rollen nginx

Innehåller tasks som:

1. Säkerställer att `systemd-timesyncd` kör (förebygger apt-fel pga klockproblem)
2. Installerar nginx via apt
3. Kopierar anpassad `index.html` till `/var/www/html/`
4. Säkerställer att nginx kör och lyssnar på port 80

### wg0.conf (WireGuard server-konfiguration)

Genereras från mall med variabler:

- `[Interface]` med privat nyckel, IP `10.0.0.1/24`, port `51820`
- `[Peer]` med klientens publika nyckel och `AllowedIPs = 10.0.0.2/32`
- `PostUp`/`PostDown` med iptables-regler för **både** internet (NAT via `enp0s3`) och internt nätverk (MASQUERADE via `enp0s8`)

### client.conf (WireGuard klient-konfiguration)

Genereras automatiskt av Ansible på gateway. Innehåller:

- Klientens privata nyckel och IP `10.0.0.2/24`
- Serverns publika nyckel
- `Endpoint = 127.0.0.1:51820` (via Vagrant port-forward)
- `AllowedIPs = 10.0.0.0/24, 192.168.56.0/24` (vad som routas via VPN)
- `PersistentKeepalive = 25` (håller anslutningen aktiv)

---

## Krav och förutsättningar

**Programvara som måste vara installerad på Windows-hosten:**

- [VirtualBox](https://www.virtualbox.org/) — testat med version 7.x
- [Vagrant](https://www.vagrantup.com/) — testat med version 2.x
- [Git](https://git-scm.com/)
- [WireGuard for Windows](https://www.wireguard.com/install/) — VPN-klient

**Hårdvarukrav:**

- Minst 4 GB RAM (projektet använder ca 1.5 GB)
- Minst 10 GB ledigt diskutrymme

**Notering om Ansible:**

Ansible behöver **inte** installeras på Windows. Det körs `ansible_local` — d.v.s. inuti varje VM. Vagrant installerar Ansible automatiskt i VM:erna första gången.

---

## Kom igång

```bash
# 1. Klona repot
git clone https://github.com/sofiacantor/VPN-Gateway-Projekt.git
cd VPN-Gateway-Projekt

# 2. Starta alla VMs (Ansible kör automatiskt)
vagrant up

#3. Köra Ansible-playbook
vagrant ssh gateway
cd /vagrant
cd ansible
ansible-playbook -i inventory.ini site.yml

# 3. Hämta klient-konfiguration från gateway via rooten
vagrant ssh gateway
sudo -i
ls /etc/wireguard/client
cat /etc/wireguard/client/client.conf
exit

# 4. Kopiera output till C:\WireGuard-Client\client.conf på Windows

# 5. Importera i tom tunnel i WireGuard-appen och aktivera
```

**Förväntat slutresultat:**

Med VPN aktivt — öppna `http://192.168.56.20` i webbläsaren. Du ska se den anpassade webbsidan "Intern Tjänst — Endast tillgänglig via VPN".

Utan VPN — samma URL ger `ERR_CONNECTION_TIMED_OUT` (säkerheten fungerar).

---

## Secrets

I detta projekt **genereras alla hemligheter (WireGuard-nycklar) automatiskt av Ansible inuti VM:n**. Inga nycklar finns i versionshanteringen.

### Vad som skyddas i Git

Filen `.gitignore` innehåller:

```
# Vagrant
.vagrant/

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

### Mallar (templates) i Git

Två mallar är committade så strukturen syns men utan riktiga värden:

- `wireguard/wg0.conf.template` — visar formatet utan nycklar
- `wireguard/client.conf.template` — visar formatet utan nycklar

Riktiga nycklar genereras av Ansible vid `vagrant up` och finns endast inuti VM:n och på Windows-klienten — aldrig i Git.

### Kontrollera att inga nycklar läckt

```bash
git log --all -- "**/*.key"        # Ska vara tom output
git log --all -- "**/client.conf"  # Ska vara tom output
```

---

## Säkerhetsåtgärder

Följande säkerhetsåtgärder är implementerade och automatiserade via Ansible:

| Åtgärd | Var | Hur verifieras det |
|---|---|---|
| Internt nätverk isolerat från host | VirtualBox (`intnet`) | `ping 192.168.56.20` från Windows utan VPN → timeout |
| WireGuard-trafik krypterad | Gateway VM | `sudo wg show` visar handshakes och nycklar |
| Privata nycklar med `chmod 600` | Gateway VM | `ls -la /etc/wireguard/*.key` |
| `/etc/wireguard` med `chmod 700` | Gateway VM | `ls -ld /etc/wireguard` |
| IP forwarding endast på gateway | Gateway VM | `cat /proc/sys/net/ipv4/ip_forward` (1=på) |
| iptables NAT för **båda** interfaces | Gateway VM | `sudo iptables -t nat -L -n -v` |
| nginx ej direkt nåbar utifrån | Internal VM | `Test-NetConnection 192.168.56.20 -Port 80` från Windows utan VPN → fail |
| Secrets utanför Git | Alla | `git log --all -- "**/*.key"` (tom output) |

---

## Säkerhetsanalys

### Vad miljön skyddar mot

| Hot | Skydd |
|-----|-------|
| Skanning av öppna portar från internet | Internal-VM har ingen direkt internetexponering |
| Direktåtkomst till webbservern | Windows-host har ingen route till `192.168.56.0/24` utan VPN |
| Avlyssning av trafik | WireGuard krypterar all trafik mellan klient och gateway |
| Obehörig anslutning till VPN | Asymmetrisk kryptografi — anslutning kräver matchande nyckelpar |
| Sidovägar via Windows-host | `intnet` (inte `host-only`) — Windows har ingen nätverksadapter på det interna nätet |

### Kvarvarande brister

**Brist 1: HTTP istället för HTTPS mellan klient och webbserver**

Trafiken mellan WireGuard-klienten och nginx är okrypterad HTTP **inuti** den krypterade VPN-tunneln. Om någon kringgår VPN-skyddet skulle trafiken kunna avlyssnas.

*Åtgärd:* Konfigurera HTTPS på nginx med ett självsignerat certifikat (eller Let's Encrypt om publik DNS sätts upp). Detta ger "defense in depth" — flera krypteringslager.

*Accepterat i denna miljö eftersom:* VPN-tunneln redan är krypterad, och webbservern är inte exponerad mot internet. Risken bedöms som låg i labbmiljö.

---

**Brist 2: Privat WireGuard-nyckel sparas oskyddat på Windows**

Klientens privata WireGuard-nyckel sparas i klartext i `C:\WireGuard-Client\client.conf`. Om Windows-datorn komprometteras kan en angripare ansluta till VPN och nå den interna tjänsten.

*Åtgärd:* Använda hårdvarunycklar (YubiKey eller liknande) för WireGuard-autentisering, alternativt kryptera filsystemet (BitLocker).

*Accepterat i denna miljö eftersom:* Labbmiljö med personlig dator där andra säkerhetskontroller (lokal användarkonto, BitLocker) bedöms räcka.

---

**Brist 3: Ingen loggning eller intrångsdetektering**

Det finns ingen övervakning som larmar vid ovanlig aktivitet — t.ex. anslutningsförsök till stängda portar eller misslyckade WireGuard-handshakes.

*Åtgärd:* Aktivera detaljerad loggning för WireGuard och nginx, och implementera en lösning som Wazuh eller Suricata för centraliserad logganalys.

*Accepterat i denna miljö eftersom:* Projektet är dimensionerat för labbmiljö, inte produktion. Övervakning är en naturlig nästa fas.

---

**Brist 4: Single point of failure — gateway-VM:n**

Om gateway-VM:n går ner är hela det interna nätverket otillgängligt. Det finns ingen redundans.

*Åtgärd:* I produktion skulle två gateway-VMs konfigureras med failover (t.ex. via keepalived/VRRP).

*Accepterat i denna miljö eftersom:* Inte ett krav för labbmiljö.

---

### Vad som skyddar miljön

Trots ovanstående brister har miljön följande skyddslager (defense in depth):

- **Lager 1 — Nätverkssegmentering:** `intnet` isolerar internt nätverk från Windows-host
- **Lager 2 — VPN-kryptering:** WireGuard krypterar all trafik
- **Lager 3 — Autentisering:** Asymmetriska nycklar krävs för anslutning
- **Lager 4 — Brandväggsregler:** iptables på gateway begränsar vilken trafik som routas
- **Lager 5 — Inga hemligheter i Git:** Nycklar genereras dynamiskt, mallar utan värden

---

## Verifiering

Verifieringen sker i två huvudkategorier: **säkerhetstester** (att tjänsten är skyddad utan VPN) och **funktionstester** (att tjänsten fungerar med VPN).

### Säkerhetstester — utan VPN aktivt

Med WireGuard-tunneln **avaktiverad** ska följande misslyckas:

| Test | Förväntat resultat |
|------|---------------------|
| `ping 192.168.56.20` från Windows | "Request timed out" |
| `curl http://192.168.56.20` från Windows | "Failed to connect" / timeout |
| `http://192.168.56.20` i webbläsare | `ERR_CONNECTION_TIMED_OUT` |
| `Test-NetConnection 192.168.56.20 -Port 80` | `TcpTestSucceeded: False` |

### Funktionstester — med VPN aktivt

Med WireGuard-tunneln **aktiverad** ska följande lyckas:

| Test | Förväntat resultat |
|------|---------------------|
| `ping 10.0.0.1` från Windows | `Reply from 10.0.0.1` |
| `ping 192.168.56.20` från Windows | `Reply from 192.168.56.20` |
| `curl http://192.168.56.20` från Windows | HTML-innehåll returneras |
| `http://192.168.56.20` i webbläsare | "Intern Tjänst" sida visas |

### Verifiering av automation

Att hela miljön fungerar från scratch:

```bash
# 1. Destroya allt
vagrant destroy -f

# 2. Skapa om från scratch
vagrant up

# 3. Verifiera att Ansible körde igenom utan fel
# Förväntat: båda VMs visar "failed=0" i PLAY RECAP

# 4. Verifiera WireGuard på gateway
vagrant ssh gateway
sudo wg
# Ska visa: interface wg0, peer med korrekt publik nyckel

# 5. Verifiera nginx på internal (från gateway)
curl http://192.168.56.20
# Ska returnera HTML-innehållet
```

För kompletta testresultat med skärmdumpar, se [docs/screenshots] (docs\screenshots) och [docs/tester.md](docs/tester.md).

---

## Designval och motivering

### Varför VPN istället för port forwarding?

Port forwarding skulle exponera den interna tjänsten direkt på en publik port — vem som helst som hittar IP och port kan ansluta. VPN-lösningen kräver korrekta krypteringsnycklar, vilket inte kan brute-forceas på rimlig tid. Dessutom är VPN-trafiken krypterad och tjänsten är osynlig från internet — ingen kan skanna och hitta den.

### Varför `intnet` istället för `host-only`?

VirtualBox `host-only`-nätverk skapar en nätverksadapter **även på Windows-hosten**, vilket innebär att hosten kan nå den interna IP-rangen `192.168.56.0/24` **direkt** — utan VPN. Detta skulle helt kringgå säkerhetsmodellen. Med `intnet` skapas nätverket **bara mellan VMs**, så VPN blir den enda vägen in.

### Varför Ansible istället för manuell konfiguration?

Manuell konfiguration av WireGuard + iptables + nginx kräver 30+ steg som måste göras exakt rätt. Vid testperioder där VMs ofta byggs om (`vagrant destroy` + `vagrant up`) är manuell konfiguration både tidskrävande och felbenägen. Ansible:

- Automatiserar hela uppställningen — ett kommando räcker
- Är **idempotent** — kan köras många gånger utan att introducera nya fel
- Versionshanterar konfigurationen — alla ändringar syns i Git-historiken
- Möjliggör reproducerbar miljö — kompis kan klona repot och få identisk uppställning

### Varför `ansible_local` istället för Ansible på Windows?

Ansible kräver Linux för att köras. Två alternativ fanns:

1. **Ansible på Windows via WSL2** — kräver att varje teammedlem installerar WSL2 och Ansible
2. **Ansible inuti VM:n (`ansible_local`)** — Vagrant installerar Ansible automatiskt i VM:n

`ansible_local` valdes för portabilitet — projektet fungerar på vilken Windows-dator som helst med bara Vagrant och VirtualBox. Detta gör projektet enklare att dela mellan teammedlemmar.

### Varför två separata VMs?

Att köra WireGuard och nginx på samma VM hade förenklat uppställningen men eliminerat det interna nätverksskiktet. Med två separata VMs:

- WireGuard-gatewayen kan vara minimal (utan onödiga tjänster)
- Internal-VM:n innehåller endast den specifika tjänsten
- Säkerhetszonerna är tydligt separerade
- Att lägga till fler interna tjänster blir trivialt (ny VM på samma intnet)

### Varför `PersistentKeepalive = 25` i klient-konfigurationen?

WireGuard skickar `PersistentKeepalive`-paket var 25:e sekund för att hålla anslutningen aktiv genom NAT-tabeller (här Vagrants port-forward). Utan detta skulle anslutningen tappa kontakt efter en kort inaktivitetsperiod, eftersom NAT-tabellerna timeout:ar. 25 sekunder är ett standardvärde som balanserar paketöverhead mot stabilitet.

---

*Skapad av: Sofia Cantor Romero och Joséphine Montanera*  
*Kurs: Virtualiseringsteknik och Automation*  
*Datum: 2026-05-22*
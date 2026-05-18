# Arkitekturbeskrivning

Detta dokument beskriver den tekniska arkitekturen i VPN-Gateway projektet
och motiverar de designval som gjorts.

## Översikt

![Arkitektur](C:\Projekt\VPN-Gateway-Projekt\docs\diagrams\arkitektur.drawio.png)

## Komponenter

### Windows-host (användarens dator)

**Roll:** Slutanvändarens miljö där VPN-klienten körs.

**Programvara:**
- WireGuard-klient (Windows PowerShell)
- VirtualBox (virtualiseringsmotor)
- Vagrant (VM-orkestrering)
- Git + GitHub (versionshantering)

**Nätverk:**
- WireGuard-tunnel: IP `10.0.0.2/24`
- Vanlig internetanslutning (via WiFi/Ethernet)

### VPN-Gateway VM

**Roll:** Den centrala säkerhetskomponenten — fungerar som "dörrvakt" för
det interna nätverket.

**OS:** Ubuntu 22.04 LTS

**Tjänster:**
- WireGuard VPN-server (port 51820/UDP)
- IP forwarding aktivt
- iptables NAT-regler för båda interfaces

**Nätverkskort:**

| Interface | IP | Funktion |
|-----------|-----|----------|
| `enp0s3` | 10.0.2.15 | NAT (VirtualBox internet-access) |
| `enp0s8` | 192.168.56.10 | Internt nätverk (intnet) |
| `wg0` | 10.0.0.1 | WireGuard VPN-interface |

### Internal VM

**Roll:** Den skyddade interna tjänsten.

**OS:** Ubuntu 22.04 LTS

**Tjänst:** Nginx-webbserver på port 80

**Nätverk:**
- `enp0s8`: 192.168.56.20 (internt nätverk)
- Ingen direkt internetexponering

## Nätverkstopologi

### Tre separata nätverk

Projektet använder tre logiskt separerade nätverk:

1. **VirtualBox NAT** (10.0.2.0/24)
   - Varje VM har sin egen "parallella" NAT-värld
   - Används endast för internetåtkomst (apt-paket etc.)

2. **VPN-nätverk** (10.0.0.0/24)
   - Krypterad WireGuard-tunnel mellan Windows-host och gateway
   - Bara två peers: host (10.0.0.2) och gateway (10.0.0.1)

3. **Internt nätverk** (192.168.56.0/24)
   - VirtualBox `intnet` — helt isolerat från Windows-host
   - Endast åtkomligt från VMs som är medlemmar
   - Detta är **kärnan** i säkerhetsmodellen

### Trafikflöde — med VPN

```
Windows (10.0.0.2)
    ↓ ICMP/HTTP
WireGuard-tunnel (krypterad)
    ↓ UDP 51820
Gateway wg0 (10.0.0.1)
    ↓ IP forwarding + NAT
Gateway enp0s8 (192.168.56.10)
    ↓ Internt nätverk
Internal enp0s8 (192.168.56.20)
    ↓ Port 80
Nginx-webbserver
```

### Trafikflöde — utan VPN

```
Windows
    ↓ försök ping 192.168.56.20
Windows route-tabell: ingen route till 192.168.56.0/24
    ↓
PAKET DROPPAS — säkerhet fungerar!
```

## Designval och motivering

### Varför VPN istället för port forwarding?

**Port forwarding** skulle exponera tjänsten direkt på en port:
- Vem som helst som hittar IP+port kan ansluta
- Inget skydd mot scanning
- Kräver att tjänsten själv har autentisering

**VPN-lösning** kräver:
- Korrekta krypteringsnycklar (ej brute-forceable)
- Hela tunneln är krypterad
- Tjänsten är osynlig från internet
- "Defense in depth" — flera säkerhetslager

### Varför `intnet` istället för `host-only`?

`host-only` (VirtualBox-standard) skapar en nätverksadapter **även på Windows-host**:
- Windows kan då nå 192.168.56.x **direkt** utan VPN
- Detta **kringgår** hela säkerhetsmodellen

`intnet` (det vi valde) skapar nätverket **bara mellan VMs**:
- Windows har ingen adapter på det nätverket
- VPN är den **enda** vägen in
- Korrekt säkerhetsmodell

### Varför Ansible för konfiguration?

Manuell konfiguration har problem:
- 30+ steg som måste göras exakt rätt
- Lätt att glömma något (typ NAT-regel för enp0s8)
- Kan inte reproduceras på en ny dator utan dokumentation

Ansible-automation:
- ETT kommando (`vagrant up`) sätter upp allt
- Idempotent (kan köras många gånger)
- Hela konfigurationen är **kod** och versionshanteras
- Kompis kan klona repot och få identisk miljö

## Säkerhetsanalys

### Vad är skyddat?

✅ **Tjänsten är osynlig utan VPN** — Windows har ingen route
✅ **Trafik krypteras** mellan klient och gateway (WireGuard)
✅ **Nycklar krävs** för anslutning — kan ej brute-forceas
✅ **Nätverkssegmentering** — internt nät isolerat
✅ **Defense in depth** — flera lager (intnet + iptables + WireGuard)

### Kvarvarande risker

⚠️ **HTTP istället för HTTPS** — om någon kringgår VPN är trafiken okrypterad
⚠️ **Privat nyckel på Windows** — om datorn komprometteras kan en angripare ansluta
⚠️ **Single point of failure** — om gateway går ner är tjänsten otillgänglig
⚠️ **Inga loggar/övervakning** — kan ej upptäcka attacker

### Förbättringar för produktion

1. Implementera HTTPS med Let's Encrypt
2. Använd hårdvarunycklar (YubiKey) för VPN-autentisering
3. Lägg till loggning och övervakning (typ Suricata IDS)
4. Konfigurera failover för gateway
5. Använd separata VPN-nycklar för varje användare
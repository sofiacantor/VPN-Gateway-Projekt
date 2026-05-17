# Sakerhetstester - VPN Gateway

Detta dokument samlar alla sakerhetsverifieringar for VPN-gateway-projektet.
Syftet ar att bevisa att den interna tjansten (192.168.56.20) endast ar
tillgänglig via VPN och inte fran ovriga natverk.

---

## Test 1: Tillgänglighet UTAN VPN

**Datum:** [2026-05-10]  
**Testat av:** Sofia och Joséphine

### Forutsattningar
- WireGuard-tunnel: **Inactive** (avstangd)
- Windows-host pa vanligt natverk
- Gateway och internal VMs kor

### Test 1.1: Ping 192.168.56.20

**Kommando:**
```
ping 192.168.56.20
```

**Forvantat resultat:** 100% paketforlust (host onabar)  
**Faktiskt resultat:** ✅ 100% paketforlust - Request timed out

**Skarmdump:** `screenshots/test1-utan-vpn-ping-fail.png`

### Test 1.2: Webblaesare

**URL:** `http://192.168.56.20`  
**Forvantat resultat:** Anslutning misslyckas  
**Faktiskt resultat:** ✅ ERR_CONNECTION_TIMED_OUT

**Skarmdump:** `screenshots/test2-utan-vpn-browser-fail.png`

### Test 1.3: curl

**Kommando:**
```
curl http://192.168.56.20 --connect-timeout 5
```

**Forvantat resultat:** Connection timeout  
**Faktiskt resultat:** ✅ Failed to connect / timeout

**Skarmdump:** `screenshots/test3-utan-vpn-curl-fail.png`

### Slutsats Test 1

Tjansten ar **INTE** tillganglig utan VPN. Detta bekraftar att internal-VM
ar isolerad pa det privata natverket och endast kan nas via gatewayen.

---

## Test 2: Tillganglighet MED VPN

**Datum:** [2016-05-10]  
**Testat av:** Sofia och Joséphine

### Forutsattningar
- WireGuard-tunnel: **Active** (aktiverad)
- Windows-host ansluten via VPN (10.0.0.2)
- Gateway och internal VMs kor

### Test 2.1: Ping 192.168.56.20

**Kommando:**
```
ping 192.168.56.20
```

**Forvantat resultat:** Reply fran 192.168.56.20  
**Faktiskt resultat:** ✅ Paket gar igenom (ca 5ms latency)

**Skarmdump:** `screenshots/test1-med-vpn-ping-ok.png`

### Test 2.2: Webblaesare

**URL:** `http://192.168.56.20`  
**Forvantat resultat:** Webbsidan laddas  
**Faktiskt resultat:** ✅ Sidan "Intern Tjanst" visas korrekt

**Skarmdump:** `screenshots/test2-med-vpn-browser-ok.png`

### Test 2.3: curl

**Kommando:**
```
curl http://192.168.56.20
```

**Forvantat resultat:** HTML-respons  
**Faktiskt resultat:** ✅ Komplett HTML returneras

**Skarmdump:** `screenshots/test3-med-vpn-curl-ok.png`

### Slutsats Test 2

Tjansten ar tillganglig nar VPN ar aktivt. VPN-tunneln routar trafik
korrekt fran Windows-host till internal-VM via gatewayen.

---

## Sammanfattning

| Test | Utan VPN | Med VPN |
|------|----------|---------|
| ping 192.168.56.20 | ❌ Timeout | ✅ Reply |
| http://192.168.56.20 | ❌ Refused | ✅ Sida laddas |
| curl http://192.168.56.20 | ❌ Failed | ✅ HTML |

**Slutsats:** Sakerhetsmodellen fungerar som tankt.

- **Utan VPN:** Tjansten ar isolerad och onabar
- **Med VPN:** Tjansten ar tillganglig for autentiserade VPN-anvandare

---

## Sakerhetsforbattring: Natverksisolering

**Datum:** [2026-05-10]  
**Genomfort av:** Sofia och Joséphine

### Bakgrund

Vid initial uppsattning anvandes `private_network` (host-only) i Vagrantfile.
Detta gjorde att Windows-host hade direkt access till 192.168.56.0/24 via en
VirtualBox host-only-adapter, vilket gjorde VPN:n meningslos for sakerhet.

### Atgard

Andrade Vagrantfile till `virtualbox__intnet: "internal-net"` for bada VMs.
Detta gor natverket till en **akta** intern VirtualBox-natverk som ar isolerat
fran Windows-host.

### Resultat

- **Innan:** Windows kunde pinga 192.168.56.20 utan VPN (sakerhetshal)
- **Efter:** Windows kan ENDAST pinga 192.168.56.20 nar VPN ar aktivt

Detta gor att VPN nu ar **enda vagen in** till det interna natverket.

---

## Felsokningsanteckning: NAT-konfiguration

**Datum:** [2026-05-10]  
**Genomfort av:** Sofia och Joséphine

### Problem

Efter natverksisolering med `intnet` kunde Windows-host inte na 192.168.56.20
trots aktiv VPN. Felsokning med `tcpdump` pa gateway visade att ICMP-paket
kom in pa wg0 och gick ut pa enp0s8 — men inga svar kom tillbaka.

### Diagnos

NAT-regeln (MASQUERADE) i wg0.conf var bara konfigurerad for utgaende
trafik via enp0s3 (internet). Trafik via enp0s8 (interna natet) NAT:ades
inte. Detta gjorde att internal-VM sag klientens VPN-IP (10.0.0.2) som
kalla, och saknade route tillbaka.

### Atgard

Lagt till en andra MASQUERADE-regel for enp0s8 i PostUp/PostDown:

```
PostUp = iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE; iptables -t nat -A POSTROUTING -o enp0s8 -j MASQUERADE
```

### Verifiering

Efter andring:

- `ping 192.168.56.20` fran Windows: ✅ Reply
- `curl http://192.168.56.20` fran Windows: ✅ HTML-respons
- Webblaesare visar tjansten korrekt

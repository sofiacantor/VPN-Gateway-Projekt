# Sakerhetstester - VPN Gateway

Detta dokument samlar alla säkerhetsverifieringar för VPN-gateway-projektet.
Syftet är att bevisa att den interna tjänsten (192.168.56.20) endast är
tillgänglig via VPN och inte från övriga nätverk.

---

## Test 1: Tillgänglighet UTAN VPN

**Datum:** [2026-05-10]  
**Testat av:** Sofia och Joséphine

### Forutsättningar
- WireGuard-tunnel: **Inactive** (avstängd)
- Windows-host pa vanligt nätverk
- Gateway och internal VMs kör

### Test 1.1: Ping 192.168.56.20

**Kommando:**
```
ping 192.168.56.20
```

**Förväntat resultat:** 100% paketförlust (host onåbar)  
**Faktiskt resultat:** ✅ 100% paketförlust - Request timed out

**Skärmdump:** `screenshots/test1-utan-vpn-ping-fail.png`

### Test 1.2: Webbläsare

**URL:** `http://192.168.56.20`  
**Förvantat resultat:** Anslutning misslyckas  
**Faktiskt resultat:** ✅ ERR_CONNECTION_TIMED_OUT

**Skärmdump:** `screenshots/test2-utan-vpn-browser-fail.png`

### Test 1.3: curl

**Kommando:**
```
curl http://192.168.56.20 --connect-timeout 5
```

**Förvantat resultat:** Connection timeout  
**Faktiskt resultat:** ✅ Failed to connect / timeout

**Skärmdump:** `screenshots/test3-utan-vpn-curl-fail.png`

### Slutsats Test 1

Tjänsten är **INTE** tillgänglig utan VPN. Detta bekräftar att internal-VM
är isolerad på det privata nätverket och endast kan nås via gatewayen.

---

## Test 2: Tillganglighet MED VPN

**Datum:** [2016-05-10]  
**Testat av:** Sofia och Joséphine

### Forutsattningar
- WireGuard-tunnel: **Active** (aktiverad)
- Windows-host ansluten via VPN (10.0.0.2)
- Gateway och internal VMs kör

### Test 2.1: Ping 192.168.56.20

**Kommando:**
```
ping 192.168.56.20
```

**Förvantat resultat:** Reply fran 192.168.56.20  
**Faktiskt resultat:** ✅ Paket går igenom (ca 5ms latency)

**Skärmdump:** `screenshots/test1-med-vpn-ping-ok.png`

### Test 2.2: Webbläsare

**URL:** `http://192.168.56.20`  
**Förvantat resultat:** Webbsidan laddas  
**Faktiskt resultat:** ✅ Sidan "Intern Tjänst" visas korrekt

**Skärmdump:** `screenshots/test2-med-vpn-browser-ok.png`

### Test 2.3: curl

**Kommando:**
```
curl http://192.168.56.20
```

**Förvantat resultat:** HTML-respons  
**Faktiskt resultat:** ✅ Komplett HTML returneras

**Skärmdump:** `screenshots/test3-med-vpn-curl-ok.png`

### Slutsats Test 2

Tjänsten är tillgänglig när VPN är aktivt. VPN-tunneln routar trafik
korrekt från Windows-host till internal-VM via gatewayen.

---

## Sammanfattning

| Test | Utan VPN | Med VPN |
|------|----------|---------|
| ping 192.168.56.20 | ❌ Timeout | ✅ Reply |
| http://192.168.56.20 | ❌ Refused | ✅ Sida laddas |
| curl http://192.168.56.20 | ❌ Failed | ✅ HTML |

**Slutsats:** Säkerhetsmodellen fungerar som tänkt.

- **Utan VPN:** Tjänsten är isolerad och onårbar
- **Med VPN:** Tjänsten är tillgänglig for autentiserade VPN-användare

---

## Säkerhetsforbättring: Nätverksisolering

**Datum:** [2026-05-10]  
**Genomfört av:** Sofia och Joséphine

### Bakgrund

Vid initial uppsättning användes `private_network` (host-only) i Vagrantfile.
Detta gjorde att Windows-host hade direkt access till 192.168.56.0/24 via en
VirtualBox host-only-adapter, vilket gjorde VPN:n meningslös for säkerhet.

### Åtgärd

Ändrade Vagrantfile till `virtualbox__intnet: "internal-net"` for båda VMs.
Detta gör nätverket till en **äkta** intern VirtualBox-natverk som är isolerat
fran Windows-host.

### Resultat

- **Innan:** Windows kunde pinga 192.168.56.20 utan VPN (säkerhetshål)
- **Efter:** Windows kan ENDAST pinga 192.168.56.20 när VPN är aktivt

Detta gör att VPN nu är **enda vägen in** till det interna nätverket.

---

## Felsökningsanteckning: NAT-konfiguration

**Datum:** [2026-05-10]  
**Genomfört av:** Sofia och Joséphine

### Problem

Efter nätverksisolering med `intnet` kunde Windows-host inte nå 192.168.56.20
trots aktiv VPN. Felsokning med `tcpdump` på gateway visade att ICMP-paket
kom in på wg0 och gick ut på enp0s8 — men inga svar kom tillbaka.

### Diagnos

NAT-regeln (MASQUERADE) i wg0.conf var bara konfigurerad for utgaende
trafik via enp0s3 (internet). Trafik via enp0s8 (interna natet) NAT:ades
inte. Detta gjorde att internal-VM sag klientens VPN-IP (10.0.0.2) som
kalla, och saknade route tillbaka.

### Åtgard

Lagt till en andra MASQUERADE-regel for enp0s8 i PostUp/PostDown:

```
PostUp = iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE; iptables -t nat -A POSTROUTING -o enp0s8 -j MASQUERADE
```

### Verifiering

Efter ändring:

- `ping 192.168.56.20` fran Windows: Reply
- `curl http://192.168.56.20` fran Windows: HTML-respons
- Webbläsare visar tjänsten korrekt

---

## Felsokningsanteckning: NAT-konfiguration

**Datum:** [Fyll i dagens datum]
**Genomfort av:** Person A

### Problem
Efter natverksisolering med `intnet` kunde Windows-host inte nas
192.168.56.20 trots aktiv VPN. Felsokning med `tcpdump` pa gateway
visade att ICMP-paket kom in pa wg0 och gick ut pa enp0s8 — men inga
svar kom tillbaka.

### Diagnos
NAT-regeln (MASQUERADE) i wg0.conf var bara konfigurerad for utgaende
trafik via enp0s3 (internet). Trafik via enp0s8 (interna natet) NAT:ades
inte. Detta gjorde att internal-VM sag klientens VPN-IP (10.0.0.2) som
kalla, och saknade route tillbaka.

### Atgard
Lagt till en andra MASQUERADE-regel for enp0s8 i PostUp/PostDown.

### Verifiering
Efter andring: 
- `ping 192.168.56.20` fran Windows: ✅ Reply
- `curl http://192.168.56.20` fran Windows: ✅ HTML-respons
- Webblaesare visar tjansten korrekt

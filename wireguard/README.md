# WireGuard-konfiguration

Mallar för WireGuard-konfiguration. Riktiga nycklar genereras manuellt och 
placeras lokalt — **aldrig** i Git.

## Generera nycklar (på gatewayen)

Logga in på gateway-VM:en och kör:

```bash
sudo -i
cd /etc/wireguard
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client_private.key | wg pubkey > client_public.key
chmod 600 server_private.key client_private.key
exit
```

## Använd mallarna

### Server (gateway)

1. Kopiera `wg0.conf.template` till `/etc/wireguard/wg0.conf` på gatewayen
2. Ersätt `<SERVER_PRIVATE_KEY>` med din serverns privata nyckel
3. Ersätt `<CLIENT_PUBLIC_KEY>` med din klients publika nyckel
4. Starta tjänsten: `sudo systemctl enable --now wg-quick@wg0`

### Klient (Windows)

1. Kopiera `client.conf.template` till en **lokal mapp utanför Git** 
   (t.ex. `C:\WireGuard-Client\client.conf`)
2. Ersätt `<CLIENT_PRIVATE_KEY>` med din klients privata nyckel
3. Ersätt `<SERVER_PUBLIC_KEY>` med din serverns publika nyckel
4. Importera i WireGuard-appen och aktivera

## Säkerhet

- **Privata nycklar får ALDRIG committas till Git**
- `.gitignore` filtrerar bort `*.key` och `.conf` (utom `.template`)
- Varje teammedlem genererar egna nyckelpar

## Viktig konfigurationsdetalj: Två MASQUERADE-regler

PostUp/PostDown maste innehalla MASQUERADE-regler for **bada** utgaende
interface:
- `enp0s3` — for ev. internet-trafik via NAT
- `enp0s8` — for trafik mot 192.168.56.0/24 (interna natverket)

Utan `enp0s8`-regeln NAT:as inte paketen pa vag till internal-VM, vilket
gor att internal-VM ser klientens VPN-IP (10.0.0.2) som kalla — och 
saknar route tillbaka. Resultat: ICMP-pingar timar ut.

Detta upptacktes med `tcpdump` under felsokning.

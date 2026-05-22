# Webbserver — Intern Tjänst

Denna mapp innehåller källfilerna för den **interna webbtjänsten** som körs på `internal-VM` (`192.168.56.20`) bakom VPN-gatewayen. Webbservern är **endast** åtkomlig med en aktiv VPN-anslutning till gatewayen.

---

## Innehållsförteckning

- [Översikt](#översikt)
- [Filer i denna mapp](#filer-i-denna-mapp)
- [Hur webbservern fungerar](#hur-webbservern-fungerar)
- [Konfigurationshistorik](#konfigurationshistorik)
- [Manuell installation (referens)](#manuell-installation-referens)
- [Automatiserad installation (Ansible)](#automatiserad-installation-ansible)
- [Verifiering](#verifiering)
- [Säkerhetsmodell](#säkerhetsmodell)
- [Felsökning](#felsökning)

---

## Översikt

Webbservern är en nginx-server som visar en anpassad HTML-sida ("Intern Tjänst — Endast tillgänglig via VPN"). Den fungerar som **representant** för en skyddad intern resurs som man i en verklig miljö skulle vilja begränsa åtkomsten till — t.ex. ett admin-panel, intern wiki eller företagsapplikation.

**Tekniska detaljer:**

| Aspekt | Värde |
|--------|-------|
| Webbserver | nginx |
| Port | 80 (HTTP) |
| Filer serveras från | `/var/www/html/` |
| IP-adress | 192.168.56.20 (internt nätverk) |
| Internetexponering | **Ingen** |
| Åtkomst | Endast via WireGuard VPN |

---

## Filer i denna mapp

```
webserver/
├── index.html           # Webbsidan som visas för användare
├── illustration.png     # Bild som visas på sidan
└── README.md            # Denna fil
```

### `index.html`

Den faktiska webbsidan som visas när man besöker `http://192.168.56.20`. Innehåller:

- Rubrik som tydligt anger "Sofies & Josies Intern Tjänst"
- Förklaring att sidan endast är åtkomlig via VPN
- Bild (`illustration.png`)
- Styling för en proffsig presentation

### `illustration.png`

Bild som visas på sidan. Inkluderas eftersom den distribueras till webbservern via Ansible.

---

## Hur webbservern fungerar

```
Användare med VPN aktivt
        ↓
Windows-klient (10.0.0.2)
        ↓ HTTP GET http://192.168.56.20
WireGuard-tunnel (krypterad)
        ↓ UDP 51820
Gateway VM (10.0.0.1 → 192.168.56.10)
        ↓ NAT MASQUERADE + IP forwarding
Internal VM (192.168.56.20)
        ↓ Port 80
Nginx serverar /var/www/html/index.html
        ↓
HTML-svar tillbaka samma väg
```

**Utan VPN:** Windows har ingen route till `192.168.56.0/24` → anslutning timeoutar → tjänsten är **osynlig**.

---

## Konfigurationshistorik

Webbservern har konfigurerats på **två sätt** under projektets utveckling:

### Fas 1: Manuell konfiguration

Initialt installerades nginx manuellt på `internal-VM` för att verifiera konceptet och förstå alla steg. Dokumenteras nedan för pedagogisk referens.

### Fas 2: Automatiserad konfiguration

Hela installationen flyttades till Ansible-rollen `nginx`. Filerna i denna mapp (`index.html`, `illustration.png`) **kopieras till** Ansible-rollens `files/`-mapp och distribueras automatiskt vid `vagrant up`.

**Sökväg i Ansible:** [`../ansible/roles/nginx/files/`](../ansible/roles/nginx/files/)

> Källfilerna behålls här i `webserver/`-mappen som "original" för referens och eventuell vidareutveckling.

---

## Manuell installation (referens)

> **Detta är historisk dokumentation.** I projektets nuvarande tillstånd sker installationen automatiskt via Ansible — se nästa sektion.

För att förstå vad Ansible-rollen gör, kan dessa manuella steg utföras:

### Steg 1: Logga in på internal-VM

```bash
vagrant ssh internal
```

### Steg 2: Säkerställ tidssynk

```bash
sudo systemctl restart systemd-timesyncd
```

**Varför:** apt kan misslyckas om VM:n har felaktig klocka (känt problem efter `vagrant up`).

### Steg 3: Uppdatera apt och installera nginx

```bash
sudo apt update
sudo apt install -y nginx
```

### Steg 4: Verifiera att nginx kör

```bash
sudo systemctl status nginx
```

Förväntat: `Active: active (running)` (grönt).

### Steg 5: Kopiera anpassad index.html

Eftersom filen ligger på Windows-host måste den först överföras. Antingen via `scp`:

```bash
# Från Windows-host (i annat PowerShell-fönster)
scp -P 2222 -o StrictHostKeyChecking=no -i .vagrant/machines/internal/virtualbox/private_key webserver/index.html vagrant@127.0.0.1:/tmp/
```

Sedan inuti VM:n:

```bash
sudo cp /tmp/index.html /var/www/html/index.html
sudo chown www-data:www-data /var/www/html/index.html
```

### Steg 6: Verifiera lokalt

```bash
curl http://localhost
```

Förväntat: HTML-koden från `index.html` returneras.

### Steg 7: Verifiera från gateway

```bash
exit
vagrant ssh gateway
curl http://192.168.56.20
```

Förväntat: Samma HTML returneras (visar att internt nätverk fungerar).

---

## Automatiserad installation (Ansible)

I projektets nuvarande version sker hela installationen automatiskt. Ansible-rollen `nginx` ([`../ansible/roles/nginx/`](../ansible/roles/nginx/)) utför följande tasks i ordning:

| Steg | Vad Ansible gör |
|------|-----------------|
| 1 | Startar `systemd-timesyncd` (förebygger apt-fel) |
| 2 | Uppdaterar apt-cache |
| 3 | Installerar nginx via apt |
| 4 | Kopierar `illustration.png` till `/var/www/html/` |
| 5 | Kopierar `index.html` till `/var/www/html/` |
| 6 | Säkerställer att nginx kör och är aktiverat vid boot |
| 7 | Väntar tills port 80 svarar (verifiering) |

**Källfilerna kommer från:** `ansible/roles/nginx/files/`

**Hela rollen körs med:**

```bash
vagrant provision internal
```

Eller automatiskt vid:

```bash
vagrant up
```

---

## Verifiering

### Med VPN aktivt — ska FUNGERA

| Test | Förväntat resultat |
|------|---------------------|
| `ping 192.168.56.20` från Windows | `Reply from 192.168.56.20` |
| `curl http://192.168.56.20` från Windows | HTML-innehållet returneras |
| `http://192.168.56.20` i webbläsare | Webbsidan visas |

### Utan VPN — ska INTE fungera

| Test | Förväntat resultat |
|------|---------------------|
| `ping 192.168.56.20` från Windows | `Request timed out` |
| `curl http://192.168.56.20` från Windows | `Failed to connect` |
| `http://192.168.56.20` i webbläsare | `ERR_CONNECTION_TIMED_OUT` |

### Lokala test inuti VMs (för felsökning)

```bash
# Från internal-VM (lokalt)
vagrant ssh internal
curl http://localhost              # Ska alltid fungera

# Från gateway-VM (via internt nätverk)
vagrant ssh gateway
curl http://192.168.56.20          # Ska alltid fungera om automation kört
```

Komplett test-dokumentation med skärmdumpar finns i [`../docs/tester.md`](../docs/tester.md).

---

## Säkerhetsmodell

### Varför är denna webbserver säker?

Webbservern är **inte säker i sig själv** — den kör vanlig okrypterad HTTP utan autentisering. **Säkerheten kommer från nätverksarkitekturen:**

1. **Internal-VM har ingen direkt internetexponering**
   - Bara på `intnet` (192.168.56.0/24) — isolerat från Windows-host
   - Ingen port-forwarding från värdmaskinen

2. **VPN är enda vägen in**
   - Windows-host måste först ansluta till VPN-gatewayen
   - Endast efter VPN-anslutning får Windows route till `192.168.56.0/24`
   - WireGuard kräver krypteringsnycklar för anslutning

3. **Defense in depth**
   - Lager 1: Nätverkssegmentering (`intnet` istället för `host-only`)
   - Lager 2: VPN-kryptering (WireGuard)
   - Lager 3: Autentisering (asymmetriska nycklar)
   - Lager 4: iptables-regler på gateway

### Kvarvarande risker

- **HTTP istället för HTTPS** — trafik mellan VPN-klient och webbserver är okrypterad inuti tunneln
- **Ingen autentisering på webbservern** — vem som helst med VPN-åtkomst ser sidan
- **nginx-version inte härdad** — använder default-konfiguration

**Förbättringar för produktion:**

- Konfigurera HTTPS med självsignerat certifikat eller Let's Encrypt
- Lägg till basic auth eller OAuth på sidan
- Härda nginx (dölj version, begränsa methods, rate limiting)

---

## Felsökning

### Problem: Sidan visas inte med VPN aktivt

**Diagnos:**

```bash
# Verifiera att nginx kör på internal
vagrant ssh internal
sudo systemctl status nginx

# Verifiera att port 80 lyssnar
sudo ss -tlnp | grep :80
```

Om nginx inte kör:

```bash
sudo systemctl start nginx
```

Eller kör om Ansible:

```bash
# På Windows
vagrant provision internal
```

### Problem: 403 Forbidden istället för sidan

Filrättigheter är fel. Kör:

```bash
vagrant ssh internal
sudo chown -R www-data:www-data /var/www/html/
sudo chmod 644 /var/www/html/index.html
```

### Problem: Default nginx-sida visas istället för anpassad

Anpassad `index.html` blev inte kopierad. Verifiera:

```bash
vagrant ssh internal
cat /var/www/html/index.html
```

Om standardsidan visas:

```bash
# På Windows
vagrant provision internal
```

### Problem: "Connection refused" från gateway

nginx kör inte eller lyssnar på fel interface. Verifiera:

```bash
vagrant ssh internal
sudo ss -tlnp | grep :80
```

Förväntat: `0.0.0.0:80` (lyssnar på alla interfaces).

---

## Referenser

- Huvud-README: [`../README.md`](../README.md)
- Ansible nginx-roll: [`../ansible/roles/nginx/`](../ansible/roles/nginx/)
- Arkitekturbeskrivning: [`../docs/arkitektur.md`](../docs/arkitektur.md)
- Säkerhetstester med skärmdumpar: [`../docs/tester.md`](../docs/tester.md)
- [Nginx officiell dokumentation](https://nginx.org/en/docs/)
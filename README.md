# VPN-Gateway Projekt

Projekt i kursen Virtualiseringsteknik och automation.

## Syfte
Bygga en VPN-gateway (WireGuard) som skyddar en intern tjänst.
Endast användare anslutna via VPN kan nå tjänsten.

## Arkitektur

![VPN-Gateway Arkitektur](docs/diagrams/vpn-gateway-architecture.png)

Diagrammet visar hela arkitekturen:

- **Windows-host** — användarens dator med WireGuard-klient (10.0.0.2)
- **VPN-Gateway VM** — Ubuntu med WireGuard-server (10.0.0.1)
- **Internal VM** — Ubuntu med nginx-webbserver (192.168.56.20)

Tjänsten på `192.168.56.20:80` är **endast** nåbar via VPN.

För detaljerad arkitekturbeskrivning, se [docs/arkitektur.md](docs/arkitektur.md).

## Teknik
- **VirtualBox + Vagrant** — virtualisering
- **WireGuard** — VPN
- **Ansible** — automation (inuti VMs via `ansible_local`)
- **Git + GitHub** — versionshantering

## Teammedlemmar
- Sofia Cantor Romero
- Joséphine Montanera

## Snabbstart

Förutsättningar: VirtualBox, Vagrant och Git installerade.

```bash
git clone https://github.com/sofiacantor/VPN-Gateway-Projekt.git
cd VPN-Gateway-Projekt
vagrant up
```

Detta sätter upp HELA miljön automatiskt:
- Två VMs (gateway och internal) på isolerat nätverk
- WireGuard installerat och konfigurerat med unika nycklar
- Nginx installerat med anpassad index.html
- IP forwarding och NAT-regler

Tid: 5-15 minuter (första gången).

## Anslut via VPN

1. Hämta klient-konfigurationen från gateway-VM:
```bash
   vagrant ssh gateway
   sudo cat /etc/wireguard/client/client.conf
```
2. Spara som `C:\WireGuard-Client\client.conf` på Windows
3. Importera i WireGuard-appen och aktivera

## Verifiera

**Med VPN aktivt:**
- `ping 10.0.0.1` → svar från gateway
- `ping 192.168.56.20` → svar från internal-VM (via gateway)
- `http://192.168.56.20` → webbsida visas

**Utan VPN:**
- `ping 192.168.56.20` → timeout (säkerhet fungerar)

## Nätverkstopologi

Projektet använder tre logiskt separerade nätverk:

| Nätverk | IP-område | Funktion |
|---------|-----------|----------|
| **VirtualBox NAT** | 10.0.2.0/24 | Internet för VMs (automatiskt) |
| **VPN-nätverk** | 10.0.0.0/24 | Krypterad WireGuard-tunnel |
| **Internt nätverk** | 192.168.56.0/24 | Mellan VMs (VirtualBox `intnet`) |

## Säkerhetsmodell

- ✅ Internal-VM **osynlig** utan VPN (ingen route från Windows-host)
- ✅ Krypterad WireGuard-tunnel mellan klient och gateway
- ✅ Internt nätverk **isolerat** via VirtualBox `intnet`
- ✅ NAT-regler på gateway för både internet och intnet
- ✅ "Defense in depth" — flera säkerhetslager

## Struktur

```
VPN-Gateway-Projekt/
├── Vagrantfile                    # VM-definitioner
├── ansible/                       # Ansible-playbooks och roller
│   ├── ansible.cfg
│   ├── inventory.ini
│   ├── site.yml
│   └── roles/
│       ├── wireguard/             # WireGuard-roll
│       └── nginx/                 # Nginx-roll
├── webserver/                     # HTML-fil för intern tjänst
├── wireguard/                     # Konfigurationsmallar
├── docs/                          # Dokumentation
│   ├── arkitektur.md              # Detaljerad arkitekturbeskrivning
│   ├── diagrams/                  # Arkitekturdiagram
│   └── screenshots/               # Test-skärmdumpar
└── README.md                      # Denna fil
```

## Status

✅ Komplett och automatiserad
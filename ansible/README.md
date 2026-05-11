# Ansible-automation

Detta directory innehåller Ansible-playbooks som automatiserar 
konfigurationen av VPN-gateway och intern webbserver.

## Struktur

```
ansible/
├── ansible.cfg          # Globala inställningar
├── inventory.ini        # Lista över maskiner
├── site.yml             # Master-playbook
└── roles/
    ├── wireguard/       # Installerar och konfigurerar WireGuard
    └── nginx/           # Installerar och konfigurerar nginx
```

## Användning

Ansible körs automatiskt av Vagrant när du startar VMs:

```
vagrant up
```

För att köra om konfigurationen utan att starta om VMs:

```
vagrant provision
```

För att köra Ansible manuellt (kräver Ansible installerat lokalt):

```
cd ansible
ansible-playbook site.yml
```

## Roller

### wireguard
Installerar och konfigurerar WireGuard på gateway-VM:
- Genererar nyckelpar
- Skapar wg0.conf med korrekt NAT (enp0s3 + enp0s8)
- Aktiverar IP forwarding
- Startar wg-quick@wg0 tjänsten

### nginx
Installerar och konfigurerar nginx på internal-VM:
- Installerar nginx-paketet
- Kopierar anpassad index.html
- Säkerställer att tjänsten kör

## Säkerhetsnoteringar

- Privata WireGuard-nycklar genereras lokalt på gateway-VM
- Nycklar lagras i `/etc/wireguard/` med 600-rättigheter
- Inga nycklar committas till Git

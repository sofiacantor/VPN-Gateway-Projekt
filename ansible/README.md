# Ansible-automation

Denna mapp innehåller all Ansible-konfiguration som automatiserar uppställningen av VPN-gateway och intern webbserver. Vid `vagrant up` kör Vagrant Ansible **inuti** varje VM (`ansible_local`) som applicerar rollerna automatiskt.

---

## Innehållsförteckning

- [Översikt](#översikt)
- [Struktur](#struktur)
- [Filer i ansible-mappen](#filer-i-ansible-mappen)
- [Roller](#roller)
- [Användning](#användning)
- [Variabler](#variabler)
- [Hur Ansible körs](#hur-ansible-körs)
- [Idempotens](#idempotens)
- [Säkerhetsnoteringar](#säkerhetsnoteringar)
- [Felsökning](#felsökning)

---

## Översikt

Ansible-automationen ersätter ca 30 manuella konfigurationssteg med ett enda kommando. Hela uppställningen av VPN-gateway och webbserver tar ca 5-15 minuter och kan reproduceras identiskt på vilken Windows-dator som helst med VirtualBox + Vagrant installerat.

**Designprincip:** Ansible körs **inuti** VM:n (`ansible_local`) istället för från Windows-hosten. Detta innebär:

- Ingen Ansible-installation behövs på Windows
- Ingen WSL2 krävs
- Samma Ansible-version körs i varje VM oavsett värddator
- Projektet är 100% portabelt mellan teammedlemmar

---

## Struktur

```
ansible/
├── ansible.cfg                          # Globala Ansible-inställningar
├── inventory.ini                        # Definierar vilka VMs som hanteras
├── site.yml                             # Master-playbook (kör alla roller)
├── README.md                            # Denna fil
│
└── roles/
    ├── wireguard/                       # Installerar och konfigurerar VPN
    │   ├── tasks/
    │   │   └── main.yml                 # 18 tasks som bygger upp WireGuard
    │   ├── handlers/
    │   │   └── main.yml                 # Restart vid konfigändringar
    │   └── templates/
    │       └── wg0.conf.j2              # Jinja2-mall för wg0.conf
    │
    └── nginx/                           # Installerar och konfigurerar webbservern
        ├── tasks/
        │   └── main.yml                 # Apt install, kopiera HTML, starta tjänst
        ├── handlers/
        │   └── main.yml                 # Restart nginx vid ändringar
        └── files/
            ├── illustration.png         # Bild som visas på webbsidan
            └── index.html               # Anpassad webbsida
```

---

## Filer i ansible-mappen

### `ansible.cfg`

Globala inställningar för Ansible:

```ini
[defaults]
inventory = inventory.ini             # Var inventory finns
host_key_checking = False             # Skippa SSH fingerprint-frågor
roles_path = ./roles                  # Var roller finns
retry_files_enabled = False           # Skapa inte .retry-filer
stdout_callback = yaml                # YAML-output istället för rörig JSON
deprecation_warnings = False          # Mindre brus i loggar

[ssh_connection]
pipelining = True                     # Snabbare anslutningar
```

### `inventory.ini`

Definierar vilka maskiner som Ansible hanterar:

```ini
[gateway]
gateway ansible_connection=local

[internal]
internal ansible_connection=local

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

**Viktigt:** `ansible_connection=local` betyder att tasks körs direkt i VM:n utan SSH. Detta är snabbare och säkrare än SSH inuti samma VM.

### `site.yml`

Master-playbook som binder ihop roller med VMs:

```yaml
---
- name: Konfigurera VPN Gateway
  hosts: gateway
  become: yes
  roles:
    - wireguard

- name: Konfigurera Intern Webbserver
  hosts: internal
  become: yes
  roles:
    - nginx
```

`become: yes` motsvarar `sudo` — Ansible kör som root vid behov.

---

## Roller

### Rollen `wireguard`

**Vad rollen gör (18 tasks i ordning):**

1. Sätter WireGuard-variabler (IP-adresser, portar, interface-namn)
2. Uppdaterar apt-cache
3. Installerar WireGuard via apt
4. Skapar `/etc/wireguard/` med rättigheter `0700`
5. Kontrollerar om server-nyckelpar finns (idempotent check)
6. Genererar server-nyckelpar (`wg genkey | wg pubkey`)
7. Kontrollerar om klient-nyckelpar finns
8. Genererar klient-nyckelpar
9. Läser in alla 4 nycklar som Ansible-fakta (variabler)
10. Aktiverar IP forwarding (`net.ipv4.ip_forward = 1`)
11. Renderar `wg0.conf` från `wg0.conf.j2`-mall
12. Sätter rättigheter `0600` på wg0.conf
13. Startar `wg-quick@wg0`-tjänsten
14. Aktiverar tjänsten vid boot
15. Skapar `/etc/wireguard/client/`-mappen
16. Genererar `client.conf` för Windows-klienten
17. Sätter rättigheter `0600` på client.conf
18. Visar instruktioner för hur klienten används

**Variabler som sätts:**

| Variabel | Värde | Förklaring |
|----------|-------|-----------|
| `wg_server_address` | `10.0.0.1/24` | Gatewayens IP i VPN-nätet |
| `wg_client_address` | `10.0.0.2` | Klientens IP i VPN-nätet |
| `wg_listen_port` | `51820` | WireGuards UDP-port |
| `wg_internet_iface` | `enp0s3` | NAT-interface (för internet) |
| `wg_internal_iface` | `enp0s8` | Internt nätverk-interface |
| `wg_config_dir` | `/etc/wireguard` | Konfigurationsmapp |

**Mall `wg0.conf.j2`:**

Genererar `wg0.conf` med:
- Serverns privata nyckel (från fakta)
- Klientens publika nyckel (från fakta)
- iptables NAT-regler för **både** internet (enp0s3) och internt nätverk (enp0s8)
- IP forwarding-regler

---

### Rollen `nginx`

**Vad rollen gör:**

1. Säkerställer att `systemd-timesyncd` kör (förebygger apt-fel pga klockproblem)
2. Uppdaterar apt-cache
3. Installerar nginx
4. Kopierar `illustration.png` till `/var/www/html/`
5. Kopierar `index.html` till `/var/www/html/`
6. Säkerställer att nginx kör och är aktiverat vid boot
7. Väntar tills port 80 svarar (verifiering)

**Files som distribueras:**

| Fil | Mål på VM | Funktion |
|-----|-----------|----------|
| `index.html` | `/var/www/html/index.html` | Webbsidan användaren ser |
| `illustration.png` | `/var/www/html/illustration.png` | Bild på sidan |

---

## Användning

### Automatisk körning via Vagrant (rekommenderat)

Ansible körs automatiskt när du skapar VMs:

```bash
vagrant up
```

### Köra om konfigurationen utan att starta om VMs

Om du ändrar Ansible-filer men inte vill destroya VMs:

```bash
vagrant provision
```

Detta kör Ansible igen mot redan existerande VMs. Tack vare **idempotens** påverkas bara det som faktiskt ändrats.

### Köra om bara en VM

```bash
vagrant provision gateway      # Bara gateway
vagrant provision internal     # Bara internal
```

### Köra Ansible manuellt inuti en VM

```bash
vagrant ssh gateway
cd /vagrant/ansible
sudo ansible-playbook site.yml --limit gateway
exit
```

---

## Variabler

Variabler sätts på två sätt i detta projekt:

### 1. Hardkodade i tasks

I `roles/wireguard/tasks/main.yml`:

```yaml
- name: Sätt WireGuard-variabler
  ansible.builtin.set_fact:
    wg_server_address: "10.0.0.1/24"
    wg_client_address: "10.0.0.2"
    wg_listen_port: 51820
    ...
```

Detta gör att variablerna är tydligt synliga i rollens kod.

### 2. Dynamiskt genererade fakta

WireGuard-nycklar **genereras** på VM:n och **läses in** som fakta:

```yaml
- name: Läs server private key
  ansible.builtin.slurp:
    src: "{{ wg_config_dir }}/server_private.key"
  register: server_priv_slurp

- name: Spara nycklar som fakta
  ansible.builtin.set_fact:
    server_private_key: "{{ server_priv_slurp.content | b64decode | trim }}"
```

Nycklarna används sedan i Jinja2-mallar (`wg0.conf.j2`).

---

## Hur Ansible körs

Detta projekt använder `ansible_local` — Vagrant installerar Ansible **inuti** VM:n och kör playbooken därifrån:

```ruby
gw.vm.provision "ansible_local" do |ansible|
  ansible.playbook = "ansible/site.yml"
  ansible.limit = "gateway"
  ansible.inventory_path = "ansible/inventory.ini"
  ansible.compatibility_mode = "2.0"
  ansible.install_mode = "default"
end
```

**Flöde:**

```
Windows-host kör "vagrant up"
        ↓
Vagrant skapar VMs i VirtualBox
        ↓
Vagrant SSH:ar in i varje VM
        ↓
Inuti varje VM:
  1. Vagrant installerar Ansible (apt)
  2. Vagrant kör: ansible-playbook site.yml --limit <vm>
  3. Ansible utför sina tasks som root
        ↓
VM:n är konfigurerad
```

---

## Idempotens

En av Ansibles viktigaste egenskaper är **idempotens** — playbooken kan köras **många gånger** utan att introducera fel eller dubblera arbete.

**Exempel:**

Om du kör `vagrant provision` upprepade gånger:

| Task | Första körningen | Andra körningen |
|------|------------------|-----------------|
| Installera WireGuard | `changed` (installerar) | `ok` (redan installerat) |
| Generera nycklar | `changed` (skapar) | `ok` (finns redan, hoppar över) |
| Rendera wg0.conf | `changed` (skapar fil) | `ok` (om innehåll är samma) |
| Starta wg-quick | `changed` (startar) | `ok` (kör redan) |

**Hur idempotens uppnås i wireguard-rollen:**

```yaml
- name: Kontrollera om server-nyckel finns
  ansible.builtin.stat:
    path: "{{ wg_config_dir }}/server_private.key"
  register: server_key_check

- name: Generera server-nycklar
  ansible.builtin.shell: |
    wg genkey | tee {{ wg_config_dir }}/server_private.key | wg pubkey > {{ wg_config_dir }}/server_public.key
  when: not server_key_check.stat.exists       # ← Kör bara om filen INTE finns
```

**Pedagogisk poäng:** Genom `when: not server_key_check.stat.exists` ser vi till att nycklar **bara genereras en gång**. Vid omkörning återanvänds de — vilket är kritiskt för att Windows-klienten inte ska tappa kontakten.

---

## Säkerhetsnoteringar

### Vad som skyddas

- Privata WireGuard-nycklar genereras **lokalt på gateway-VM**, aldrig på Windows-host
- Nycklar lagras i `/etc/wireguard/` med rättigheter `0700` (mapp) och `0600` (filer)
- Endast root kan läsa nycklarna inuti VM:n
- Inga nycklar finns någonsin i Git (skyddat av `.gitignore`)
- Mallar (`*.template`) i `wireguard/`-mappen visar struktur men innehåller inga riktiga värden

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

### Hur klienten får sin konfiguration

1. Ansible genererar `client.conf` i `/etc/wireguard/client/` på gateway-VM
2. Användaren hämtar manuellt med:
```bash
   vagrant ssh gateway
   sudo cat /etc/wireguard/client/client.conf
```
3. Användaren klistrar in i `C:\WireGuard-Client\client.conf` på Windows
4. WireGuard-appen importerar filen

**Detta innebär att klient-konfigurationen aldrig passerar Git eller publika nätverk.**

---

## Felsökning

### Problem: "ansible: command not found"

Detta händer om du försöker köra Ansible på Windows-host. Men du behöver inte! Ansible körs **inuti** VM. Använd istället:

```bash
vagrant ssh gateway
ansible --version       # Verifierar att Ansible är installerat i VM
```

### Problem: Playbook misslyckas med "Unable to lock the administration directory"

Detta är ett tillfälligt apt-lås. Vänta 30 sekunder och kör om:

```bash
vagrant provision
```

### Problem: WireGuard-tjänsten startar inte

Verifiera konfigurationen:

```bash
vagrant ssh gateway
sudo systemctl status wg-quick@wg0
sudo journalctl -u wg-quick@wg0 -n 50
```

Vanliga orsaker:
- IP forwarding inte aktiverat → kontrollera `/proc/sys/net/ipv4/ip_forward`
- iptables-konflikt → kontrollera `sudo iptables -L -n -v`

### Problem: Vill regenerera nycklar från scratch

Eftersom nycklarna är idempotenta måste de tas bort manuellt:

```bash
vagrant ssh gateway
sudo rm /etc/wireguard/*.key
sudo rm /etc/wireguard/*.pub
sudo rm /etc/wireguard/client/client.conf
exit

# Sedan kör om Ansible
vagrant provision gateway
```

### Problem: "Tasks ran for the wrong host"

Kontrollera att `--limit` används korrekt:

```bash
vagrant provision gateway      # Korrekt — bara gateway
vagrant provision              # Båda VMs
```

---

## Referenser

- [Ansible-dokumentation](https://docs.ansible.com/)
- [WireGuard officiell sida](https://www.wireguard.com/)
- [Vagrant Ansible Local Provisioner](https://developer.hashicorp.com/vagrant/docs/provisioning/ansible_local)
- Huvud-README: [../README.md](../README.md)
- Arkitekturbeskrivning: [../docs/arkitektur.md](../docs/arkitektur.md)
- Säkerhetstester: [../docs/tester.md](../docs/tester.md)
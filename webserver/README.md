# Webbserver-konfiguration

HTML-fil for nginx pa internal-VM (192.168.56.20).

## Manuell installation

1. SSH in: `vagrant ssh internal`
2. Installera nginx: `sudo apt install -y nginx`
3. Kopiera `index.html` till `/var/www/html/index.html`
4. Verifiera: `curl http://localhost`

Detta automatiseras senare med Ansible

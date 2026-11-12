# Partie B — Installation

[← Partie A — Contexte et choix techniques](../Partie%20A%20%E2%80%94%20Contexte%20et%20choix%20techniques/contexte-et-choix-techniques.md) · [Retour au README](../README.md) · [Partie C — Mise en production →](../Partie%20C%20%E2%80%94%20Mise%20en%20production/mise-en-production.md)

---

L'installation suit la [documentation officielle de NetBox](https://docs.netbox.dev/en/stable/installation/), adaptée au contexte du service.

## Sommaire

1. [Préparation de la VM](#1-préparation-de-la-vm)
2. [Installation d'Ubuntu Server](#2-installation-dubuntu-server)
3. [Synchronisation de l'heure](#3-synchronisation-de-lheure)
4. [PostgreSQL](#4-postgresql)
5. [Redis](#5-redis)
6. [NetBox](#6-netbox)
7. [Gunicorn et services systemd](#7-gunicorn-et-services-systemd)
8. [Nginx et HTTPS](#8-nginx-et-https)
9. [Sauvegardes de la base](#9-sauvegardes-de-la-base)
10. [Supervision](#10-supervision)

---

## 1. Préparation de la VM

On vérifie l'intégrité de l'ISO deux fois, sur le poste puis sur l'hyperviseur, en comparant son empreinte avec le fichier `SHA256SUMS` publié par Ubuntu :

```powershell
Get-FileHash .\ubuntu-24.04.5-live-server-amd64.iso -Algorithm SHA256
```

Paramètres Hyper-V :

- **démarrage sécurisé** conservé, avec le modèle « Autorité de certification UEFI Microsoft » (obligatoire pour démarrer Ubuntu) ;
- lecteur DVD en premier dans l'ordre de démarrage pendant l'installation, puis disque dur remis en premier.

## 2. Installation d'Ubuntu Server

| Paramètre | Choix | Raison |
|---|---|---|
| Langue | English | messages d'erreur faciles à rechercher |
| Clavier | French | — |
| Type | Ubuntu Server standard (pas « minimized ») | outils d'administration présents |
| Réseau | IPv4 manuel : 192.0.2.23, passerelle et DNS 192.0.2.254 | IP fixe de serveur |
| Disque | LVM sans chiffrement LUKS, volume `/` agrandi à ~57 Go | LUKS imposerait une phrase secrète à chaque démarrage ; l'installateur ne proposait que la moitié du disque |
| Services | OpenSSH, aucun snap | administration à distance, système minimal |

Après l'installation :

```bash
passwd                                   # remplacer le mot de passe temporaire
sudo -k && sudo -v                       # vérifier les droits sudo
sudo timedatectl set-timezone Europe/Paris
sudo apt update && sudo apt upgrade
ls /var/run/reboot-required 2>/dev/null && echo "Redémarrage nécessaire" || echo "Pas de redémarrage nécessaire"
```

On vérifie la sortie Internet, nécessaire pour apt, GitHub et PyPI :

```bash
curl -sI https://github.com | head -n 1
curl -sI https://pypi.org/simple/ | head -n 1
```

## 3. Synchronisation de l'heure

Le serveur se synchronise sur les contrôleurs de domaine, référence de temps du réseau. On utilise un **fichier séparé** : le fichier principal n'est pas modifié, et le retour arrière consiste simplement à supprimer ce fichier.

```bash
sudo mkdir -p /etc/systemd/timesyncd.conf.d
sudo nano /etc/systemd/timesyncd.conf.d/ntp-ad.conf
```

```ini
[Time]
NTP=SRV-DC2.corp.example.org SRV-DC1.corp.example.org
```

Le contrôleur principal du domaine (PDC) est placé en premier, car il est la référence de temps du domaine.

```bash
sudo systemctl restart systemd-timesyncd
timedatectl timesync-status
```

## 4. PostgreSQL

```bash
sudo apt install -y postgresql
sudo -u postgres psql
```

```sql
CREATE DATABASE netbox;
CREATE USER netbox;
\password netbox                 -- saisie masquée, pas d'historique
ALTER DATABASE netbox OWNER TO netbox;
\connect netbox;
GRANT CREATE ON SCHEMA public TO netbox;
\l netbox                        -- propriétaire netbox, encodage UTF8
\q
```

> [!TIP]
> `\password` permet de saisir le mot de passe sans qu'il apparaisse à l'écran ni dans l'historique, contrairement à `CREATE USER … PASSWORD '…'`.

Test de connexion avec le compte applicatif :

```bash
psql --username netbox --password --host localhost netbox   # puis \conninfo
```

## 5. Redis

```bash
sudo apt install -y redis-server
redis-cli ping                     # PONG
sudo ss -tlnp | grep 6379          # 127.0.0.1:6379 et [::1]:6379 uniquement
```

Redis n'écoute que sur l'interface locale : il n'est pas joignable depuis le réseau.

## 6. NetBox

Dépendances et récupération du code à la version choisie :

```bash
sudo apt install -y python3 python3-pip python3-venv python3-dev build-essential \
  libxml2-dev libxslt1-dev libffi-dev libpq-dev libssl-dev zlib1g-dev

sudo mkdir -p /opt/netbox/ && cd /opt/netbox/
sudo git clone https://github.com/netbox-community/netbox.git .
sudo git checkout v4.7.3
sudo git describe --tags           # v4.7.3
```

Compte système dédié, propriétaire des dossiers où NetBox écrit :

```bash
sudo adduser --system --group netbox
sudo chown --recursive netbox /opt/netbox/netbox/media/ /opt/netbox/netbox/reports/ /opt/netbox/netbox/scripts/
```

Configuration :

```bash
cd /opt/netbox/netbox/netbox/
sudo cp configuration_example.py configuration.py
python3 ../generate_secret_key.py      # SECRET_KEY
python3 ../generate_secret_key.py      # pepper des jetons d'API
sudo nano -l configuration.py
```

Lignes modifiées dans `configuration.py` :

```python
ALLOWED_HOSTS = ['ipam.example.org', '192.0.2.23']

DATABASES = {
    'default': {
        'NAME': 'netbox',
        'USER': 'netbox',
        'PASSWORD': '<SECRET>',
        'HOST': 'localhost',
        'PORT': '',
        'CONN_MAX_AGE': 300,
    }
}

SECRET_KEY = '<SECRET>'

API_TOKEN_PEPPERS = {
    1: '<SECRET>',        # obligatoire pour les jetons d'API v2 (NetBox 4.5+), 50 caractères minimum
}

TIME_ZONE = 'Europe/Paris'
```

Les paramètres Redis par défaut (localhost, bases 0 et 1) sont conservés.

On protège le fichier, qui contient les secrets :

```bash
sudo chown root:netbox configuration.py
sudo chmod 640 configuration.py         # -rw-r----- root netbox
```

Les secrets sont contrôlés **sans être affichés** : longueur (50 caractères minimum), syntaxe Python valide, aucune valeur vide.

Installation, migrations et création du super-utilisateur :

```bash
sudo /opt/netbox/upgrade.sh             # "Upgrade complete!"
sudo /opt/netbox/venv/bin/python /opt/netbox/netbox/manage.py showmigrations --plan | grep -c "\[ \]"   # 0 migration en attente
sudo /opt/netbox/venv/bin/python /opt/netbox/netbox/manage.py createsuperuser
```

Test du compte sans afficher le mot de passe :

```bash
sudo /opt/netbox/venv/bin/python /opt/netbox/netbox/manage.py shell -c \
"import getpass; from django.contrib.auth import authenticate; print('MOT DE PASSE OK' if authenticate(username='admin', password=getpass.getpass()) else 'ECHEC')"
```

Test avec le serveur de développement. Comme le pare-feu ne laisse passer que le port 443, le test se fait depuis la VM elle-même, avec deux sessions SSH :

```bash
# session SSH 1
sudo /opt/netbox/venv/bin/python /opt/netbox/netbox/manage.py runserver 192.0.2.23:8000 --insecure
# session SSH 2
curl -sI http://192.0.2.23:8000/login/ | head -n 1     # HTTP/1.1 200 OK
```

## 7. Gunicorn et services systemd

```bash
sudo cp /opt/netbox/contrib/gunicorn.py /opt/netbox/gunicorn.py
grep -n "^bind" /opt/netbox/gunicorn.py              # bind = '127.0.0.1:8001'
echo "control_socket_disable = True" | sudo tee -a /opt/netbox/gunicorn.py
```

> [!NOTE]
> Gunicorn 26 crée par défaut une prise de contrôle dans le dossier personnel de l'utilisateur. Le compte système `netbox` n'en a pas, ce qui provoquait l'erreur `Control server error: Permission denied: '/nonexistent'`. La directive `control_socket_disable = True` désactive cette fonction, inutile ici.

```bash
sudo cp -v /opt/netbox/contrib/*.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now netbox netbox-rq
curl -sI -H "Host: 192.0.2.23" http://127.0.0.1:8001/login/ | head -n 1   # 200 OK
```

L'en-tête `Host` est nécessaire, car NetBox refuse les requêtes dont le nom ne figure pas dans `ALLOWED_HOSTS`.

## 8. Nginx et HTTPS

```bash
sudo apt install -y nginx
sudo cp /opt/netbox/contrib/nginx.conf /etc/nginx/sites-available/netbox
sudo rm /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/netbox /etc/nginx/sites-enabled/netbox
```

La configuration part du modèle officiel `contrib/nginx.conf`. Seuls le nom du serveur et les chemins du certificat sont modifiés (lignes signalées par `# modifié`) :

```nginx
server {
    listen [::]:443 ssl ipv6only=off;

    server_name ipam.example.org 192.0.2.23;                                   # modifié

    ssl_certificate     /etc/ssl/certs/wildcard.example.org-fullchain.crt;    # modifié
    ssl_certificate_key /etc/ssl/private/wildcard.example.org.key;            # modifié

    client_max_body_size 25m;

    location /static/ {
        alias /opt/netbox/netbox/static/;
    }

    location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    # Redirection HTTP -> HTTPS
    listen [::]:80 ipv6only=off;
    server_name _;
    return 301 https://$host$request_uri;
}
```

Nginx gère le chiffrement TLS, sert directement les fichiers statiques (`/static/`) et transmet le reste des requêtes à Gunicorn, qui n'écoute qu'en local. Le second bloc redirige tout accès HTTP vers HTTPS.

Installation du certificat wildcard, à partir des trois fichiers copiés depuis le poste d'administration (certificat, certificat intermédiaire, clé privée) :

```bash
# depuis le poste d'administration (PowerShell)
scp -r .\cert-netbox <admin>@192.0.2.23:~/

# sur le serveur
chmod 600 ~/cert-netbox/wildcard.example.org.key
grep -c ENCRYPTED ~/cert-netbox/wildcard.example.org.key        # 0 = clé sans phrase secrète

# chaîne complète : certificat + intermédiaire
{ cat ~/cert-netbox/wildcard.example.org.cer; echo; cat ~/cert-netbox/<intermediaire>.cer; } \
  | sudo tee /etc/ssl/certs/wildcard.example.org-fullchain.crt > /dev/null
grep -c "BEGIN CERTIFICATE" /etc/ssl/certs/wildcard.example.org-fullchain.crt   # 2

sudo install -m 600 -o root -g root ~/cert-netbox/wildcard.example.org.key /etc/ssl/private/wildcard.example.org.key
sudo nginx -t && sudo systemctl reload nginx
rm -r ~/cert-netbox                    # suppression des copies (également sur le poste)
```

Vérification du certificat réellement servi, **depuis le serveur** :

```bash
echo | openssl s_client -connect 192.0.2.23:443 -servername ipam.example.org 2>/dev/null \
  | openssl x509 -noout -subject -issuer -enddate
echo | openssl s_client -connect 192.0.2.23:443 -servername ipam.example.org 2>/dev/null \
  | grep "Verify return code"          # 0 (ok)
```

> [!IMPORTANT]
> La vérification se fait depuis le serveur et non depuis un navigateur : l'antivirus du poste filtre le trafic TLS et re-signe les certificats, ce qui fausse l'empreinte affichée.

## 9. Sauvegardes de la base

```bash
sudo install -d -m 700 -o postgres -g postgres /var/backups/netbox
```

Script `/usr/local/sbin/netbox-backup.sh` (droits 755), exécuté par le compte `postgres` :

```bash
#!/bin/bash
# Sauvegarde quotidienne de la base NetBox (format custom pg_dump)
set -euo pipefail
umask 077
cd /                              # le compte postgres n'a pas accès au dossier courant de l'appelant

DOSSIER=/var/backups/netbox
RETENTION_JOURS=14
FICHIER="$DOSSIER/netbox_$(date +%F_%H%M).dump"

pg_dump -Fc -f "$FICHIER" netbox
find "$DOSSIER" -name 'netbox_*.dump' -type f -mtime +"$RETENTION_JOURS" -delete
echo "Sauvegarde OK : $FICHIER"
```

Planification dans `/etc/cron.d/netbox-backup`, tous les jours à 1 h 30, avec le résultat envoyé dans le journal système :

```cron
30 1 * * * postgres /usr/local/sbin/netbox-backup.sh 2>&1 | logger -t netbox-backup
```

Contrôles :

```bash
journalctl -t netbox-backup --since yesterday      # "Sauvegarde OK : ..."
sudo ls -l /var/backups/netbox
sudo -u postgres pg_restore --list /var/backups/netbox/<fichier>.dump | head -n 12
```

**Test de restauration** dans une base séparée, sans toucher à la production :

```bash
sudo -u postgres createdb netbox_test_restauration
sudo -u postgres pg_restore -d netbox_test_restauration /var/backups/netbox/<fichier>.dump
sudo -u postgres psql -d netbox_test_restauration -c "SELECT username FROM users_user;"
sudo -u postgres dropdb netbox_test_restauration
```

Procédure de restauration réelle (documentée) :

```bash
sudo -u postgres /usr/local/sbin/netbox-backup.sh   # sauvegarde de l'état actuel avant toute opération
sudo systemctl stop netbox netbox-rq
sudo -u postgres dropdb netbox
sudo -u postgres createdb -O netbox netbox
sudo -u postgres pg_restore -d netbox /var/backups/netbox/<fichier>.dump
sudo systemctl start netbox netbox-rq
```

> [!WARNING]
> En cas de restauration sur un nouveau serveur, il faut reprendre à l'identique le mot de passe PostgreSQL, la `SECRET_KEY` et le pepper des jetons : sans eux, les jetons d'API existants deviennent invalides.

## 10. Supervision

Le serveur est supervisé par CheckMK. Les flux nécessaires sont ouverts par l'administrateur réseau :

| Sens | Port | Usage |
|---|---|---|
| SRV-IPAM → supervision | TCP 80 | téléchargement de l'agent |
| SRV-IPAM → supervision | TCP 8000 | enregistrement de l'agent (Agent Receiver) |
| supervision → SRV-IPAM | TCP 6556 | interrogation de l'agent |

```bash
nc -zv -w 3 198.51.100.21 80
nc -zv -w 3 198.51.100.21 8000
wget http://198.51.100.21/<site>/check_mk/agents/check-mk-agent_<version>_all.deb
sudo dpkg -i check-mk-agent_<version>_all.deb
systemctl is-active check-mk-agent.socket cmk-agent-ctl-daemon
sudo cmk-agent-ctl register --hostname SRV-IPAM --server 198.51.100.21 --site <site> --user agent_registration
# sans --password : le mot de passe est saisi à l'écran et n'apparaît pas dans l'historique
```

Résultat : l'hôte est UP dans la supervision, avec les services CPU, disque, uptime et agent.

---

[← Partie A — Contexte et choix techniques](../Partie%20A%20%E2%80%94%20Contexte%20et%20choix%20techniques/contexte-et-choix-techniques.md) · [Retour au README](../README.md) · [Partie C — Mise en production →](../Partie%20C%20%E2%80%94%20Mise%20en%20production/mise-en-production.md)

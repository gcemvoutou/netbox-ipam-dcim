# Partie C — Mise en production

[← Partie B — Installation](../Partie%20B%20%E2%80%94%20Installation/procedure-installation.md) · [Retour au README](../README.md) · [Partie D — IPAM, DCIM et source de vérité →](../Partie%20D%20%E2%80%94%20IPAM%2C%20DCIM%20et%20source%20de%20v%C3%A9rit%C3%A9/ipam-dcim-source-de-verite.md)

---

## Chronologie

La partie technique a été réalisée en cinq jours ouvrés :

| Jour | Étapes |
|---|---|
| J1 | Vérification de l'ISO, préparation de la VM |
| J2 | Ubuntu Server, NTP, mises à jour, PostgreSQL, Redis |
| J3 | NetBox, Gunicorn, Nginx (avec un certificat temporaire) |
| J4 | DNS, certificat wildcard, sauvegardes, comptes, redémarrage de contrôle → **mise en production** |
| J5 et suite | Supervision (après ouverture des flux), puis alimentation des données |

Un point de contrôle Hyper-V a été pris avant l'installation de NetBox, VM arrêtée proprement pour garantir un état cohérent. Il a été supprimé une fois la production validée.

## Validation finale

On redémarre le serveur pour vérifier que tout repart seul :

```bash
sudo reboot
systemctl is-active postgresql redis-server netbox netbox-rq nginx cron     # active x6
curl -sI https://ipam.example.org/login/ | head -n 1                        # HTTP/1.1 200 OK
df -h /                                                                      # 16 % utilisé
```

Les six services sont actifs, la page de connexion répond en HTTPS par son nom DNS, et l'espace disque est largement suffisant.

## Checklist de mise en production

- [x] HTTPS sans avertissement, chaîne de certification vérifiée
- [x] services relancés automatiquement au démarrage
- [x] sauvegarde automatique vérifiée le lendemain
- [x] restauration testée dans une base séparée
- [x] secrets rangés dans le gestionnaire de mots de passe, `configuration.py` protégé
- [x] comptes nominatifs et autorisations en place
- [x] serveur supervisé
- [x] points de contrôle Hyper-V supprimés
- [x] administrateur réseau informé de la fin de l'installation (intégration de la VM à la sauvegarde)

## Documentation d'exploitation

Une fiche technique interne décrit les opérations courantes : redémarrage, vérification des services, consultation des journaux, mises à jour du système, renouvellement du certificat, sauvegarde et restauration, gestion des comptes.

Commandes d'exploitation principales :

```bash
systemctl is-active postgresql redis-server netbox netbox-rq nginx cron   # état des services
journalctl -u netbox -n 30 --no-pager                                     # journaux de NetBox
journalctl -t netbox-backup --since yesterday                             # résultat de la dernière sauvegarde
sudo -u postgres /usr/local/sbin/netbox-backup.sh                         # sauvegarde manuelle (avant une mise à jour ou un gros import)
sudo systemctl restart netbox netbox-rq                                   # après une modification de configuration
sudo nginx -t && sudo systemctl reload nginx                              # après une modification de Nginx
df -h /                                                                   # espace disque
```

Mise à jour de NetBox (méthode prévue par l'installation Git) :

```bash
sudo -u postgres /usr/local/sbin/netbox-backup.sh   # sauvegarde avant mise à jour
cd /opt/netbox
sudo git fetch --tags
sudo git checkout vX.Y.Z                            # après lecture des notes de version
sudo ./upgrade.sh
sudo systemctl restart netbox netbox-rq
```

> [!IMPORTANT]
> Pour arrêter la VM, on utilise `sudo shutdown -h now`. L'option « Éteindre » de l'hyperviseur revient à débrancher la prise et peut corrompre la base.

---

[← Partie B — Installation](../Partie%20B%20%E2%80%94%20Installation/procedure-installation.md) · [Retour au README](../README.md) · [Partie D — IPAM, DCIM et source de vérité →](../Partie%20D%20%E2%80%94%20IPAM%2C%20DCIM%20et%20source%20de%20v%C3%A9rit%C3%A9/ipam-dcim-source-de-verite.md)

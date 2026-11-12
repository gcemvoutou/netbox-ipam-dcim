# Partie A — Contexte et choix techniques

[← Retour au README](../README.md) · [Partie B — Installation →](../Partie%20B%20%E2%80%94%20Installation/procedure-installation.md)

---

## Contexte

Le projet est mené dans le service informatique d'une collectivité territoriale, composé d'une petite équipe : un responsable, un administrateur réseau (également RSSI), des techniciens et une alternante. Il couvre tout le cycle, de l'installation du serveur à l'alimentation des données.

Avant la mise en production, une **preuve de concept** (POC) avait été réalisée sous Docker (`netbox-docker`). Elle a permis de prendre en main NetBox et de préparer les premiers fichiers d'import.

## Installation native, sans Docker

L'équipe a choisi de ne pas utiliser Docker en production, pour plusieurs raisons :

- l'équipe ne maîtrise pas Docker : l'exploitation et le dépannage sont plus simples avec des **services systemd classiques** ;
- NetBox documente pas à pas cette méthode d'installation officielle ;
- chaque composant (PostgreSQL, Redis, Gunicorn, Nginx) reste **visible et administrable séparément** ;
- c'est la même logique que pour les autres serveurs Linux du service.

La contrepartie : chaque brique, que Docker fournissait toute prête dans le POC, doit être installée et configurée à la main.

## Choix des versions

| Choix | Justification |
|---|---|
| **NetBox v4.7.3** plutôt que v4.7.1 | Les notes de version montrent que la 4.7.2 corrige une faille de sécurité (jetons d'API visibles en clair dans les résultats de tâches) et que la 4.7.3 corrige le contrôle d'accès. Même série 4.7 que le POC, donc fichiers CSV compatibles. |
| **Ubuntu Server 24.04 LTS** (et non une version plus récente) | Seule version citée comme testée par la documentation d'installation native. |
| **Téléchargement par Git** (et non par archive) | Mises à jour plus simples : `git checkout <nouvelle version>` puis `upgrade.sh`. |

> [!TIP]
> Lire les notes de version avant de choisir une version : c'est ce qui a permis d'éviter une version comportant une faille connue.

## HTTPS avec le certificat wildcard

La structure dispose déjà d'un certificat wildcard (`*.example.org`) délivré par une autorité publique. Il est utilisé pour NetBox :

- **aucun avertissement** dans les navigateurs et aucun certificat à déployer sur les postes ;
- **pas de Let's Encrypt** : le serveur est interne et non joignable depuis Internet, et un certificat wildcard géré par l'équipe existait déjà ;
- **pas de certificat auto-signé** en production : il provoque un avertissement et habitue les utilisateurs à l'ignorer. Il n'a servi qu'une journée, en attendant le vrai certificat.

Nginx sert la **chaîne complète** (certificat + intermédiaire) et la clé privée est protégée en `600 root`.

> [!WARNING]
> La durée de vie des certificats publics raccourcit progressivement (200 jours, puis 100 jours, puis 47 jours d'ici 2029). Le renouvellement est donc décrit dans une procédure écrite, et le certificat est inscrit à l'inventaire des certificats du service.

## Sauvegardes sur deux niveaux

- **Niveau infrastructure** : sauvegarde de la VM complète par le prestataire.
- **Niveau application** : `pg_dump` quotidien de la base sur la VM, qui permet une restauration fine de NetBox sans restaurer toute la VM.

Les points de contrôle Hyper-V ne sont pas utilisés en production (politique de l'équipe, qui les juge instables). Ils n'ont servi que pendant l'installation, puis ont été supprimés.

## Gestion des secrets

- Les secrets sont rangés dans le **gestionnaire de mots de passe de l'équipe** (KeePassXC), en deux entrées : serveur et application web.
- Secrets concernés : mot de passe du compte système, mot de passe PostgreSQL, `SECRET_KEY`, `API_TOKEN_PEPPERS`, mot de passe du compte web `admin`.
- `configuration.py` est en `640 root:netbox` : seuls root et le service NetBox peuvent le lire.
- Aucun secret n'apparaît dans un mail, un wiki ou une capture d'écran.

## Comptes locaux, sans annuaire

Les comptes NetBox sont des **comptes locaux**, sans liaison LDAP/Active Directory :

- l'équipe est petite (quelques comptes) ;
- cela limite les dépendances : pas de compte de service AD à créer ni de flux LDAP à ouvrir.

L'accès réseau reste filtré par un **groupe AD au niveau du pare-feu** : seuls les agents autorisés peuvent joindre le port 443 du serveur.

---

[← Retour au README](../README.md) · [Partie B — Installation →](../Partie%20B%20%E2%80%94%20Installation/procedure-installation.md)

# Partie G — Difficultés et leçons retenues

[← Partie F — Utilisation au quotidien](../Partie%20F%20%E2%80%94%20Utilisation%20au%20quotidien/utilisation-au-quotidien.md) · [Retour au README](../README.md)

---

## Difficultés techniques

| Problème | Cause | Solution |
|---|---|---|
| Lettres inversées pendant l'installation | console de l'installateur restée en QWERTY malgré le choix French | mot de passe temporaire composé de caractères identiques en QWERTY et AZERTY, changé ensuite en SSH |
| Le pavé numérique ne tape pas de chiffres dans la console | état de Verr Num inversé entre le poste et la console Hyper-V | désactiver Verr Num sur le poste |
| Volume `/` trop petit | l'installateur n'attribue que la moitié du disque au volume LVM | volume agrandi pendant l'installation |
| Mauvaise version cible | lecture incomplète de la page des versions sur GitHub | vérification dans les notes de version officielles → v4.7.3 (correctifs de sécurité) |
| Mot de passe PostgreSQL d'exemple appliqué | copier-coller de la documentation avec `CREATE USER … PASSWORD` | correction avec `\password` (saisie masquée), retenue comme bonne pratique |
| `Control server error: Permission denied: '/nonexistent'` | Gunicorn 26 crée une prise de contrôle dans le dossier personnel, inexistant pour le compte système | `control_socket_disable = True` dans `gunicorn.py` |
| Test impossible depuis le poste | seul le port 443 est ouvert par le pare-feu | tests depuis la VM elle-même (deux sessions SSH, `curl`) |
| `curl` refusé sur 127.0.0.1:8001 | nom absent de `ALLOWED_HOSTS` | ajout de l'en-tête `-H "Host: 192.0.2.23"` |
| Mot-clé en double dans la configuration Nginx | erreur de collage dans nano | correction avec `sed`, puis validation avec `nginx -t` |
| Empreinte du certificat différente entre le navigateur et le serveur | l'antivirus du poste filtre le trafic TLS et re-signe les certificats | vérification systématique depuis le serveur avec `openssl s_client` |
| Première sauvegarde en échec : `find: Failed to restore initial working directory` | `sudo -u postgres` hérite du dossier courant, inaccessible au compte postgres | `cd /` en début de script |
| Supervision : connexions refusées sur les ports 80 et 8000 | flux non ouverts sur le pare-feu | demande à l'administrateur réseau, nouveau test avec `nc -zv` |
| Confusion de mots de passe à l'enregistrement de l'agent | les invites `sudo` et `agent_registration` se suivent | identifier chaque invite avant de taper |

## Qualité des données

C'est la partie la plus longue du projet.

- **Sources contradictoires** (deux IP différentes pour un même équipement) : rien n'est inventé, l'équipement est mis de côté dans un fichier « à valider » jusqu'à vérification.
- **IP obsolètes encore supervisées** : des équipements avaient changé d'adresse en DHCP sans que la supervision soit mise à jour.
- **Plages DHCP inconnues** sur le réseau des écoles, géré par un prestataire : demande écrite au prestataire.
- **Tableur des VLAN contredit par la configuration réelle** (masque, présence d'un DHCP) : la configuration du pare-feu fait foi pour la partie réseau.
- **Réseaux « à supprimer » encore actifs** : ils sont enregistrés avec le statut Obsolète plutôt qu'ignorés.
- **Même plage d'IP privées sur deux sites distants** : c'est la limite d'un IPAM sans VRF, d'où l'évolution prévue vers une VRF par site distant.
- **Noms d'équipements codés** (codes de services) : validation par le référent métier avant l'import.
- **Équipements en IP dynamique mais supervisés par leur IP** : signalés à l'administrateur réseau, qui prévoit de leur attribuer une IP statique.
- **Nettoyage de la supervision pendant le projet** : les listes ont été sauvegardées avant chaque suppression d'hôtes, sans aucune perte d'information.

## Prise en main de NetBox

- Un import CSV contenant **une seule erreur n'importe rien** : tout renommage d'objet est annoncé avant l'import.
- Les jauges d'utilisation sont peu parlantes sur un /16 : une plage DHCP de 250 adresses n'y représente que 0,4 %.
- Plages d'adresses : les cases **« peuplé »** (les IP de la plage sont considérées occupées et ne peuvent pas être créées une par une) et **« marquer comme utilisé »** se cochent manuellement.
- Préfixes : la case « utilisé » marque le préfixe plein à 100 %, elle ne doit donc pas être cochée.
- Une IP située dans une plage n'est pas comptée deux fois dans l'utilisation du préfixe.
- Les préférences d'affichage des tableaux sont propres à chaque utilisateur. Une présentation par défaut commune à l'équipe se définit avec `DEFAULT_USER_PREFERENCES`.

## Leçons retenues

> [!TIP]
> - **Lire les notes de version** avant de choisir une version.
> - **Toujours tester depuis le serveur**, et non depuis un poste dont le trafic peut être filtré.
> - Utiliser `\password` plutôt que `CREATE USER … PASSWORD '…'`.
> - **Sauvegarder la base avant chaque lot d'import.**
> - **Annoncer le résultat attendu avant un import**, puis le vérifier.
> - L'alimentation des données prend beaucoup plus de temps que l'installation technique.

## Ce que le projet m'a apporté

- **Gestion de projet** : objectif et échéance, découpage en étapes, journal de bord tenu à chaque étape.
- **Rigueur en sécurité** : moindre privilège, secrets jamais exposés, accès au pare-feu en lecture seule, aucun scan réseau sans l'accord du RSSI.
- **Travail en équipe** avec plusieurs interlocuteurs : responsable du service, RSSI, référents métier, prestataire.
- **Qualité des données** : croisement des sources, fichiers « à valider », vérification après chaque import.
- **Documentation pour deux publics** : une fiche utilisateur (navigateur) et une fiche d'exploitation (serveur).

---

[← Partie F — Utilisation au quotidien](../Partie%20F%20%E2%80%94%20Utilisation%20au%20quotidien/utilisation-au-quotidien.md) · [Retour au README](../README.md)

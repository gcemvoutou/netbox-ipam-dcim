# Partie E — Comptes et droits

[← Partie D — IPAM, DCIM et source de vérité](../Partie%20D%20%E2%80%94%20IPAM%2C%20DCIM%20et%20source%20de%20v%C3%A9rit%C3%A9/ipam-dcim-source-de-verite.md) · [Retour au README](../README.md) · [Partie F — Utilisation au quotidien →](../Partie%20F%20%E2%80%94%20Utilisation%20au%20quotidien/utilisation-au-quotidien.md)

---

## Modèle retenu

Comptes **locaux et nominatifs**, selon le **principe du moindre privilège** : chacun ne dispose que des droits nécessaires à son rôle.

| Compte | Rôle | Droits |
|---|---|---|
| `admin` | administration (comptes, configuration) et import initial | super-utilisateur ; mot de passe dans le gestionnaire de l'équipe |
| administrateur réseau | saisie et mise à jour des données | autorisation **Gestion des données** (view, add, change, delete) |
| référent copieurs | mise à jour des copieurs | ajout et modification sur les objets concernés |
| autres agents | consultation de l'IPAM et du DCIM | autorisation **Lecture seule** (view) |

Le modèle ne repose pas sur des groupes : avec une petite équipe, les autorisations sont attribuées directement aux comptes.

## Les deux autorisations

Elles sont créées dans *Administration → Authentification → Autorisations* :

- **Lecture seule** : action `view` sur tous les types d'objets, sauf les exclusions ci-dessous ;
- **Gestion des données** : actions `view`, `add`, `change` et `delete` (sans `render_config` ni `sync`), avec les mêmes exclusions.

### Objets exclus et pourquoi

| Catégorie | Objets | Raison de l'exclusion |
|---|---|---|
| **Users** | utilisateurs, groupes, autorisations, jetons | la gestion des comptes est réservée à `admin` |
| **Core** | sources de données, fichiers, tâches, révisions de configuration | configuration de NetBox ; les résultats de tâches peuvent contenir des informations sensibles |
| **Extras** | scripts, modules de scripts, webhooks, règles d'événement | un script exécute du code sur le serveur ; un webhook envoie des données vers l'extérieur |

## Règles de gestion

- **Identifiant** : nom de famille en minuscules.
- **Mot de passe provisoire**, à changer à la première connexion (testé : l'ancien est refusé, le nouveau accepté). Il ne réutilise jamais un mot de passe d'administration existant, dont l'empreinte se retrouverait sinon dans la base et ses sauvegardes.
- **Besoin d'un droit supplémentaire** : on l'ajoute au compte nominatif, plutôt que de prêter le compte `admin`. La traçabilité est ainsi conservée dans le journal des modifications.
- **Départ d'un agent** : le compte est **désactivé** (case *Actif*) et non supprimé, pour que l'historique de ses modifications reste attribué.
- **Jetons d'API** : un utilisateur peut en créer, mais jamais avec plus de droits que les siens.
- **Accès au serveur (SSH)** : distinct des comptes NetBox, et réservé à l'administration du système.

## Tests réalisés

- Connexion avec un compte de test en lecture seule : les menus sont visibles, aucun bouton d'ajout ni de modification n'apparaît.
- Changement du mot de passe provisoire à la première connexion.

---

[← Partie D — IPAM, DCIM et source de vérité](../Partie%20D%20%E2%80%94%20IPAM%2C%20DCIM%20et%20source%20de%20v%C3%A9rit%C3%A9/ipam-dcim-source-de-verite.md) · [Retour au README](../README.md) · [Partie F — Utilisation au quotidien →](../Partie%20F%20%E2%80%94%20Utilisation%20au%20quotidien/utilisation-au-quotidien.md)

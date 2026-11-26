---
icon: material/clipboard-text-outline
title: Nommage et documentation
tags: [bonnes pratiques, mémo, nommage, documentation, modèle]
---

# 📋 Nommage et documentation

!!! abstract "Usage"
    Des conventions de noms prêtes à l'emploi et des **modèles à copier-coller** : fiche serveur, journal des changements, plan d'adressage. Le pourquoi : 🧠 [Les réflexes de l'admin](concept-reflexes-admin.md).

## Conventions de nommage

La meilleure convention est celle qu'on **applique partout**. Un nom doit dire ce qu'est la chose sans avoir à l'ouvrir.

| Objet | Convention | Exemples | À éviter |
|---|---|---|---|
| Serveur | `srv-<rôle><numéro>` | `srv-dc01`, `srv-files01`, `srv-web01` | `serveur`, `test2`, `gandalf` |
| Modèle (template) | `tpl-<système><numéro>` | `tpl-debian01` | `debian-ok-final` |
| Hôte de virtualisation | `pve<numéro>` | `pve01`, `pve02` | `proxmox` |
| Poste client | `PC-<numéro>` (ou `PC-<site>-<numéro>`) | `PC-01`, `PC-02` | le prénom de l'utilisateur |
| Compte utilisateur | `prénom` ou `p.nom` (une seule règle) | `alice`, `a.martin` | `user1`, comptes partagés |
| Compte d'admin | compte nominatif séparé | `admin`, `adm-alice` | travailler avec le compte d'admin intégré |
| Compte de service | `svc-<application>` | `svc-backup`, `svc-zabbix` | utiliser un compte de personne |
| Groupe AD | `GG_<service>` (global) · `DL_<ressource>_<droit>` (domaine local) | `GG_Compta`, `DL_Partage-Compta_RW` | `groupe1` |
| Partage | `<service>` ou `<usage>` | `Compta`, `Commun` | `partage`, `nouveau dossier` |
| VLAN | `<numéro> <NOM>` | `10 ADMIN`, `20 COMPTA`, `99 INVITES` | numéros sans nom |
| Snapshot | `avant-<modification>` | `avant-maj`, `avant-durcissement-ssh` | `snap1` |
| Copie de conf | `<fichier>.bak-<AAAA-MM-JJ>` | `sshd_config.bak-2026-11-26` | `sshd_config.old` |

!!! tip "Règles générales"
    - Minuscules, sans accents ni espaces, tiret `-` comme séparateur (les noms d'hôtes n'acceptent pas `_`)
    - Numéroter dès le premier (`01`) : le jour où arrive le deuxième, rien à renommer
    - Les dates toujours au format **AAAA-MM-JJ** : elles se trient dans l'ordre

## Plan d'adressage

Un tableau tenu à jour, avant d'attribuer la moindre IP fixe :

| IP | Nom | Rôle | Type | Remarque |
|---|---|---|---|---|
| `10.0.0.1` | `pve01` | Hôte Proxmox, passerelle | Fixe | |
| `10.0.0.10` | `srv-dc01` | AD, DNS, DHCP | Fixe | |
| `10.0.0.20` | `srv-files01` | Serveur de fichiers | Fixe | |
| `10.0.0.31` | `srv-web01` | Serveur web | Fixe | |
| `10.0.0.100`–`10.0.0.199` | — | Plage DHCP des postes | DHCP | |
| `10.0.0.200` | `tpl-debian01` | Modèle Debian | Fixe | éteint |

## Modèle : fiche serveur

Une fiche par machine, à créer dès sa mise en service.

```markdown
# srv-web01

| Champ | Valeur |
|---|---|
| Rôle | Serveur web (site vitrine) |
| Système | Debian 13 |
| IP / DNS | 10.0.0.31 / srv-web01.lab.local |
| Hébergement | VM 100 sur pve01 (2 vCPU, 2 Go, 20 Go) |
| Services | nginx (80, 443), SSH (22) |
| Accès | SSH par clé, compte admin ; console Proxmox en secours |
| Sauvegarde | vzdump chaque nuit à 21 h, rétention 7 jours |
| Supervision | ping + HTTP dans Uptime Kuma |
| Dépend de | srv-dc01 (DNS) |
| Utilisé par | srv-lb01 |
| Responsable | admin |

## Procédures
- Redémarrer le site : `sudo systemctl restart nginx`
- Conf : /etc/nginx/sites-available/site

## Historique
Voir le journal des changements.
```

## Modèle : journal des changements

Une entrée par intervention, la plus récente en haut.

```markdown
## AAAA-MM-JJ — <machine> — <titre court>
- Qui :
- Quoi : (fichiers, commandes, paramètres)
- Pourquoi :
- Vérifié : (tests faits après)
- Retour arrière : (comment annuler)
- Snapshot / sauvegarde : (nom, date de suppression prévue)
```

Exemple rempli dans 🛠️ [Faire une intervention propre de A à Z](tuto-intervention-propre.md#etape-9-documenter).

## Modèle : retour d'expérience après incident

```markdown
## AAAA-MM-JJ — Incident : <titre>
- Début / fin : HH:MM → HH:MM
- Impact : (qui, quoi, combien de temps)
- Chronologie : HH:MM constat… HH:MM action… HH:MM rétabli
- Cause : 
- Correction appliquée :
- Pour éviter que ça recommence :
```

## Où ranger la documentation

| Option | Pour qui | Avantages |
|---|---|---|
| Dossier de fichiers Markdown dans Git | Homelab, petite équipe | Historique des versions, lisible partout |
| Wiki (BookStack, Wiki.js) | Équipe | Recherche, droits, mise en page |
| Outil de gestion de parc (GLPI) | Entreprise | Inventaire, tickets et docs liés |

!!! warning "Pièges fréquents"
    - Écrire des **mots de passe** dans la doc : ils vont dans un **gestionnaire de mots de passe** (KeePassXC, Bitwarden, Vaultwarden), la doc dit seulement où les trouver.
    - Une doc stockée **uniquement** sur le serveur qu'elle décrit : le jour de la panne, elle est inaccessible.
    - Des noms différents pour la même machine dans le DNS, Proxmox et la doc.

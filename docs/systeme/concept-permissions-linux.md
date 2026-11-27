---
icon: material/book-open-variant
title: Comptes, droits et permissions Linux
tags: [système, bases, concept, linux, utilisateurs, groupes, permissions, sudo]
---

# 🧠 Comptes, droits et permissions Linux

!!! abstract "En une phrase"
    Sous Linux, chaque fichier appartient à **un utilisateur** et à **un groupe**, et porte trois jeux de droits (**lire, écrire, exécuter**) : pour le **propriétaire**, pour le **groupe** et pour **tous les autres**.

## L'analogie

Un immeuble de bureaux. Chaque pièce (fichier ou dossier) a **un responsable** (le propriétaire) et **un service** attitré (le groupe). Sur la porte, une affiche dit ce que peuvent faire le responsable, les collègues du service et les visiteurs : entrer, consulter les dossiers, les modifier. **root**, c'est le gardien avec le passe-partout ; **sudo**, c'est demander le passe-partout à l'accueil pour une seule intervention, en signant le registre.

## Comptes et groupes

| Élément | Rôle | Où c'est stocké |
|---|---|---|
| **Utilisateur** | Un compte, identifié par un **UID** (`root` = 0, humains à partir de 1000) | `/etc/passwd` |
| **Mot de passe** | Haché, lisible seulement par root | `/etc/shadow` |
| **Groupe** | Ensemble d'utilisateurs, identifié par un **GID** | `/etc/group` |
| **Groupe principal** | Groupe attribué aux fichiers que l'utilisateur crée (souvent du même nom que lui) | 4ᵉ champ de `/etc/passwd` |
| **Groupes secondaires** | Groupes supplémentaires qui donnent des droits en plus (`sudo`, `www-data`…) | `/etc/group` |
| **Compte système** | Compte de service sans connexion (`www-data`, `postgres`), UID < 1000 | `/etc/passwd` |

## Lire les droits d'un fichier

```text
$ ls -l rapport.txt
-rw-r----- 1 alice compta 1204 nov. 26 10:12 rapport.txt
│└┬┘└┬┘└┬┘   └──┬──┘ └──┬─┘
│ │  │  │       │       └─ groupe
│ │  │  │       └───────── propriétaire
│ │  │  └───────────────── autres : aucun droit
│ │  └──────────────────── groupe : lecture
│ └─────────────────────── propriétaire : lecture + écriture
└───────────────────────── type : - fichier, d dossier, l lien
```

| Droit | Lettre | Valeur | Sur un **fichier** | Sur un **dossier** |
|---|---|---|---|---|
| Lecture | `r` | 4 | Lire le contenu | Lister les fichiers (`ls`) |
| Écriture | `w` | 2 | Modifier le contenu | Créer, renommer, **supprimer** des fichiers dedans |
| Exécution | `x` | 1 | Lancer comme programme | **Entrer** dedans (`cd`) et accéder aux fichiers |

On additionne les valeurs pour chaque catégorie : `rw-r-----` = **6** (4+2) **4** (4) **0** → `640`.

| Notation | Droits | Usage typique |
|---|---|---|
| `644` | `rw-r--r--` | Fichier lisible par tous, modifiable par son propriétaire |
| `600` | `rw-------` | Fichier privé (clé SSH, fichier de mots de passe) |
| `755` | `rwxr-xr-x` | Script ou dossier accessible à tous |
| `750` | `rwxr-x---` | Dossier d'équipe : le groupe entre, les autres non |
| `770` | `rwxrwx---` | Dossier partagé où le groupe écrit aussi |

## Les commandes clés

| Besoin | Commande |
|---|---|
| Changer les droits | `chmod 750 dossier` ou `chmod g+w fichier` |
| Changer le propriétaire / le groupe | `chown alice:compta fichier` |
| Créer un utilisateur | `adduser alice` (Debian) / `useradd -m alice` |
| Ajouter à un groupe | `usermod -aG compta alice` |
| Voir ses identités | `id`, `groups` |
| Agir en administrateur | `sudo <commande>` |

## Les droits spéciaux (à connaître)

| Droit | Notation | Effet |
|---|---|---|
| **setgid** sur un dossier | `chmod g+s` (2xxx) | Les fichiers créés dedans héritent du **groupe du dossier** : indispensable pour un dossier partagé |
| **sticky bit** sur un dossier | `chmod +t` (1xxx) | Chacun ne peut supprimer que **ses** fichiers (c'est le cas de `/tmp`) |
| **setuid** sur un exécutable | `chmod u+s` (4xxx) | Le programme s'exécute avec les droits de son propriétaire (ex. `passwd`) : à éviter sur ses propres fichiers |

## root et sudo

| | Se connecter en **root** | Utiliser **sudo** |
|---|---|---|
| Portée | Tous les droits, tout le temps | Droits d'admin pour une seule commande |
| Traçabilité | On ne sait pas qui a fait quoi | Chaque commande est journalisée avec le nom de l'utilisateur |
| Risque d'erreur | Élevé | Réduit : on réfléchit avant de taper `sudo` |
| Qui y a droit | Celui qui connaît le mot de passe root | Les membres du groupe `sudo` (Debian/Ubuntu) ou `wheel` (RHEL/Rocky), ou les règles de `/etc/sudoers` |

## À quoi ça sert concrètement

- Créer un dossier partagé pour un service (compta, RH) sans que les autres le lisent.
- Donner les droits d'admin à un technicien sans lui confier le mot de passe root.
- Comprendre un **« Permission denied »** et le corriger sans tout passer en `777`.
- Sécuriser les fichiers sensibles (`~/.ssh`, fichiers de config avec mots de passe).

!!! warning "Pièges fréquents"
    - **`chmod 777`** pour « régler » un problème : tout le monde peut tout modifier. Chercher plutôt le bon propriétaire ou le bon groupe.
    - **`usermod -G` sans `-a`** : remplace tous les groupes secondaires au lieu d'en ajouter un. L'utilisateur peut perdre `sudo`.
    - **Ajout à un groupe sans effet** : il faut **se déconnecter et se reconnecter** (ou `newgrp`) pour que le nouveau groupe soit pris en compte.
    - **Dossier sans `x`** : on voit le nom des fichiers mais on ne peut pas les ouvrir. Un dossier a besoin de `x` pour être traversé.
    - **Modifier `/etc/sudoers` directement** : une faute de syntaxe bloque sudo. Toujours passer par `visudo`.

## Pour aller plus loin

- 🛠️ **[Gérer utilisateurs, groupes et permissions](tuto-permissions-linux.md)** : la mise en pratique
- 🛠️ [Préparer une VM clonée depuis un template](tuto-preparer-clone-vm.md) (une VM Debian pour t'entraîner)
- 🧠 [SSH](../reseau/concept-ssh.md) (pourquoi `~/.ssh` doit être en `700` et la clé en `600`)

??? quote "Sources"
    - [Page de manuel chmod(1)](https://man7.org/linux/man-pages/man1/chmod.1.html)
    - [Page de manuel credentials(7) (UID, GID)](https://man7.org/linux/man-pages/man7/credentials.7.html)
    - [Debian Reference : utilisateurs et permissions](https://www.debian.org/doc/manuals/debian-reference/ch01.en.html)

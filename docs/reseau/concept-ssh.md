---
icon: material/book-open-variant
title: SSH
tags: [réseau, bases, concept, ssh, accès distant]
---

# 🧠 SSH

!!! abstract "En une phrase"
    **SSH** (*Secure Shell*) permet d'ouvrir un terminal sur une machine distante à travers le réseau, avec une connexion **chiffrée** et **authentifiée**.

## L'analogie

SSH, c'est un **tunnel blindé** entre ton clavier et le serveur : tout ce que tu tapes passe dedans, et personne sur le trajet ne peut lire ni modifier ce qui circule. À l'entrée du tunnel, le serveur vérifie qui tu es (mot de passe ou clé), et toi tu vérifies que c'est bien le bon serveur (empreinte).

## Les éléments à connaître

| Élément | Rôle |
|---|---|
| **Client SSH** | Sur ta machine : la commande `ssh` (intégrée à Windows 10/11, Linux, macOS) |
| **Serveur SSH** | Sur la machine distante : le service `sshd` (paquet `openssh-server`) |
| **Port** | **22/TCP** par défaut |
| **Empreinte du serveur** | Identité du serveur, mémorisée dans `~/.ssh/known_hosts` à la 1ʳᵉ connexion |
| **Paire de clés** | Une clé **privée** (reste chez toi, secrète) + une clé **publique** (déposée sur le serveur) |
| `~/.ssh/config` | Ton carnet de raccourcis : nom court, utilisateur, port, clé |

## Mot de passe ou clé ?

| | Mot de passe | Clé |
|---|---|---|
| Sécurité | Vulnérable au bruteforce | Quasi impossible à deviner |
| Confort | À taper à chaque fois | Connexion directe |
| Usage | Dépannage, premier accès | **Recommandé** au quotidien |

## Se connecter avec un nom plutôt qu'une IP

`ssh` a besoin de savoir **quelle adresse** se cache derrière un nom. Trois façons de le lui dire :

| Méthode | Où ça se configure | Portée |
|---|---|---|
| Fichier **hosts** | Sur ton poste | Ton poste uniquement |
| Fichier **`~/.ssh/config`** | Sur ton poste | SSH uniquement, avec utilisateur et clé en prime |
| **mDNS** (`nom.local`) ou **DNS** | Sur la VM ou sur le serveur DNS | Tout le réseau |

## Joindre une machine qu'on ne voit pas directement

Quand les VM sont sur un **réseau interne** que ton PC ne joint pas, SSH peut **rebondir** par une machine qui voit les deux réseaux (hôte Proxmox, serveur « bastion ») : c'est l'option **ProxyJump** (`ssh -J`). La connexion reste chiffrée de bout en bout.

## À quoi ça sert concrètement

- Administrer des serveurs Linux, des switchs, des pare-feux sans être devant.
- Copier des fichiers (`scp`, `sftp`) de façon chiffrée.
- Gagner du temps : `ssh vm-debian` au lieu de `ssh admin@192.168.56.101`.

!!! warning "Pièges fréquents"
    - **VM en mode NAT** : elle n'est pas joignable depuis ton poste. Utiliser le mode **pont** (*bridged*) ou **réseau privé hôte** (*host-only*).
    - **L'IP de la VM change** (DHCP) : le nom pointe vers une ancienne adresse. Fixer l'IP ou faire une réservation DHCP.
    - Message **« REMOTE HOST IDENTIFICATION HAS CHANGED »** : la VM a été réinstallée. Supprimer l'ancienne empreinte avec `ssh-keygen -R <nom>`.
    - Ne **jamais** partager ni copier sa **clé privée** (fichier sans `.pub`).

## Pour aller plus loin

- 🛠️ **[Se connecter en SSH à une VM par son nom](tuto-ssh-nom-vm.md)** : la mise en pratique
- 🛠️ **[SSH par rebond vers un réseau interne](tuto-ssh-rebond.md)** : quand les VM ne sont pas joignables directement
- 🧠 [DHCP et DNS](concept-dhcp-dns.md) (la méthode « DNS » à l'échelle d'un réseau)

??? quote "Sources"
    - [RFC 4251 : architecture du protocole SSH](https://www.rfc-editor.org/rfc/rfc4251)
    - [RFC 4253 : couche transport SSH](https://www.rfc-editor.org/rfc/rfc4253)
    - [Manuels OpenSSH](https://www.openssh.com/manual.html)

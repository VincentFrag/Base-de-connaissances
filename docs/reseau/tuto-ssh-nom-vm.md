---
icon: material/wrench-outline
title: Se connecter en SSH à une VM par son nom
tags: [réseau, tuto, ssh, accès distant, linux, windows, vm]
---

# 🛠️ Se connecter en SSH à une VM par son nom

!!! abstract "Objectif"
    Taper `ssh vm-debian` au lieu de `ssh admin@192.168.56.101`, sans mot de passe, depuis Windows ou Linux.

**Niveau :** 🟢 Bases · **Durée :** 15 à 20 min

## Prérequis

- Comprendre SSH, les clés et les façons de résoudre un nom → 🧠 [SSH](concept-ssh.md)
- Une VM Linux (Debian/Ubuntu) dans VirtualBox, VMware ou Proxmox
- La VM en carte réseau **pont** (*bridged*) ou **réseau privé hôte** (*host-only*), **pas en NAT**

## Les trois méthodes

| Méthode | Avantage | Limite | Pour qui |
|---|---|---|---|
| **A. Fichier hosts** | Simple, marche pour tout (ping, navigateur…) | Droits admin, IP à mettre à jour à la main | Une ou deux VM |
| **B. `~/.ssh/config`** ⭐ | Nom + utilisateur + clé en un seul alias | SSH uniquement | **Le quotidien** |
| **C. mDNS (`.local`)** | Automatique, suit l'IP même si elle change | Réseau local uniquement | Lab avec IP en DHCP |

!!! info "Les VM sont sur un réseau interne que ton PC ne joint pas ?"
    Ces méthodes ne suffisent pas : il faut passer par un hôte de rebond → 🛠️ [SSH par rebond vers un réseau interne](tuto-ssh-rebond.md).

👉 La combinaison la plus confortable : **C + B** (la VM s'annonce toute seule, et l'alias SSH ajoute l'utilisateur et la clé).

---

## Étape 1 — Préparer la VM

**Sur la VM :**

```bash
# Installer et démarrer le serveur SSH
sudo apt update && sudo apt install -y openssh-server
sudo systemctl enable --now ssh

# Donner un nom clair à la VM
sudo hostnamectl set-hostname vm-debian

# Relever son adresse IP
ip -4 a
```

Teste une première fois **avec l'IP**, depuis ton poste :

```bash
ssh admin@192.168.56.101
```

Accepte l'empreinte (`yes`), entre le mot de passe, puis `exit`.

## Étape 2 — Créer une clé et la déposer sur la VM

**Sur ton poste :**

=== "Windows (PowerShell)"
    ```powershell
    # Créer la paire de clés (Entrée pour accepter le chemin ; une phrase de passe est conseillée)
    ssh-keygen -t ed25519

    # Envoyer la clé publique sur la VM
    type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh admin@192.168.56.101 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
    ```

=== "Linux"
    ```bash
    ssh-keygen -t ed25519
    ssh-copy-id admin@192.168.56.101
    ```

Vérifie : `ssh admin@192.168.56.101` doit se connecter **sans mot de passe** (ou seulement avec la phrase de passe de la clé).

---

## Étape 3 — Choisir une méthode pour le nom

=== "A. Fichier hosts"

    **Sur ton poste**, ajoute une ligne `IP  nom` :

    === "Windows"
        Ouvre le Bloc-notes **en administrateur**, puis le fichier :
        ```text
        C:\Windows\System32\drivers\etc\hosts
        ```
        Ajoute en bas :
        ```text
        192.168.56.101   vm-debian
        ```
        Ou en une commande (PowerShell **administrateur**) :
        ```powershell
        Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value "192.168.56.101   vm-debian"
        ```

    === "Linux"
        ```bash
        echo "192.168.56.101   vm-debian" | sudo tee -a /etc/hosts
        ```

    Connexion :
    ```bash
    ssh admin@vm-debian
    ```

=== "B. ~/.ssh/config ⭐"

    **Sur ton poste**, crée ou complète le fichier de configuration SSH :

    - Windows : `C:\Users\<toi>\.ssh\config` (sans extension)
    - Linux : `~/.ssh/config`

    ```text
    Host vm-debian
        HostName 192.168.56.101
        User admin
        IdentityFile ~/.ssh/id_ed25519

    # Autre exemple : un serveur sur un port différent
    Host srv01
        HostName 192.168.20.10
        User admin
        Port 2222
    ```

    !!! tip "Combiner avec la méthode C"
        Remplace `HostName 192.168.56.101` par `HostName vm-debian.local` : l'alias suivra l'IP même si elle change.

    Connexion :
    ```bash
    ssh vm-debian
    ```

    Sous Linux, pense aux droits : `chmod 600 ~/.ssh/config`.

=== "C. mDNS (.local)"

    **Sur la VM**, installe Avahi, qui annonce le nom de la machine sur le réseau local :

    ```bash
    sudo apt install -y avahi-daemon
    sudo systemctl enable --now avahi-daemon
    ```

    La VM est désormais joignable sous **`<hostname>.local`**, ici `vm-debian.local`.

    **Sur ton poste :**

    - **Windows 10/11** : rien à installer, la résolution `.local` est intégrée.
    - **Linux** : installer `avahi-daemon` et `libnss-mdns` s'ils sont absents.

    Connexion :
    ```bash
    ssh admin@vm-debian.local
    ```

    !!! note "Et à l'échelle d'une entreprise ?"
        On ne gère ni fichiers hosts ni mDNS : on crée un **enregistrement A** dans le DNS interne → 🛠️ [Installer et configurer DHCP et DNS](tuto-dhcp-dns.md).

---

## ✅ Vérification

```bash
# Le nom se résout-il vers la bonne IP ?
ping vm-debian            # méthode A
ping vm-debian.local      # méthode C

# Voir la configuration réellement appliquée par SSH (méthode B)
ssh -G vm-debian | findstr /i "hostname user identityfile"   # Windows
ssh -G vm-debian | grep -Ei "^(hostname|user|identityfile)"  # Linux

# La connexion finale
ssh vm-debian
```

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `Could not resolve hostname` | Le nom n'est connu nulle part | Vérifier l'orthographe dans hosts / config, ou Avahi sur la VM |
| `Connection timed out` | Le nom est résolu, mais la VM est injoignable | Mode réseau de la VM (pas NAT), pare-feu, VM démarrée ? |
| `Connection refused` | La VM répond, mais pas de SSH | `sudo systemctl status ssh` sur la VM |
| `Permission denied (publickey)` | La clé n'est pas acceptée | Refaire l'étape 2, vérifier `User` dans la config |
| Le mot de passe est encore demandé | La clé n'est pas utilisée | `IdentityFile` dans la config, droits `600` sur `authorized_keys` |

## 🔧 Dépannage courant

!!! question "« REMOTE HOST IDENTIFICATION HAS CHANGED » ?"
    La VM a été réinstallée ou une autre machine a repris son IP. Supprime l'ancienne empreinte, puis reconnecte-toi :
    `ssh-keygen -R vm-debian` (et `ssh-keygen -R 192.168.56.101`).

!!! question "Ça marchait hier, plus aujourd'hui ?"
    L'IP de la VM a changé (DHCP). Passe à la méthode **C**, fixe l'IP de la VM, ou fais une réservation DHCP.

!!! question "Windows : le fichier hosts refuse d'enregistrer ?"
    Le Bloc-notes doit être lancé **en administrateur**. Vérifie aussi qu'il n'a pas ajouté d'extension `.txt`.

!!! question "Windows : `vm-debian.local` ne répond pas ?"
    Le profil réseau Windows est peut-être en **Public**, ce qui bloque la découverte réseau. Passe-le en **Privé**.

## Voir aussi

- 🧠 [SSH](concept-ssh.md)
- 🛠️ [Diagnostiquer un problème de connexion](tuto-diagnostic-connexion.md)

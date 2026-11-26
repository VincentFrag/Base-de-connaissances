---
icon: material/wrench-outline
title: Se connecter en SSH aux VM d'un réseau interne (rebond)
tags: [réseau, tuto, ssh, proxyjump, rebond, bastion, proxmox, accès distant]
---

# 🛠️ Se connecter en SSH aux VM d'un réseau interne (rebond)

!!! abstract "Objectif"
    Taper simplement `ssh srv-lb01` ou `ssh srv-files01` depuis ton PC, alors que les VM sont sur un **réseau interne** qu'il ne peut pas joindre directement, en **rebondissant** par l'hôte de virtualisation (Proxmox ou tout serveur « bastion »).

**Niveau :** 🟡 Intermédiaire · **Durée :** 15 min

## Prérequis

- Comprendre SSH, les clés et `~/.ssh/config` → 🧠 [SSH](concept-ssh.md)
- Un PC avec le client OpenSSH (intégré à Windows 10/11 et à Linux)
- Un accès SSH à l'hôte de rebond (ici l'hôte Proxmox)
- Le serveur SSH installé sur les VM (`openssh-server`)

## Le principe

Ton PC ne voit pas le réseau interne des VM, mais l'hôte Proxmox, lui, est branché sur les deux. SSH passe donc **par lui** :

```
[PC Windows] ──SSH──▶ [Hôte Proxmox « pve01 »] ──SSH──▶ [VM srv-lb01]
 192.168.1.x           192.168.1.20                     10.0.0.30
 réseau du PC          ┆ patte sur les 2 réseaux ┆       réseau interne 10.0.0.0/24
```

Tout reste chiffré de bout en bout : l'hôte de rebond transporte la connexion mais ne la déchiffre pas.

## Le lab d'exemple

| Machine | Rôle | IP |
|---|---|---|
| `pve01` | Hôte Proxmox (rebond) | `192.168.1.20` |
| `srv-dc01` | Contrôleur de domaine / DNS (`lab.local`) | `10.0.0.10` |
| `srv-files01` | Serveur de fichiers | `10.0.0.20` |
| `srv-lb01` | Répartiteur de charge (HAProxy) | `10.0.0.30` |
| `srv-web01` | Serveur web Apache | `10.0.0.31` |
| `srv-web02` | Serveur web Nginx | `10.0.0.32` |

Réseau interne : `10.0.0.0/24` · passerelle `10.0.0.1` · DNS `10.0.0.10`

---

## Étape 1 — Vérifier les deux sauts séparément

**Saut 1** : le PC joint l'hôte Proxmox.

```powershell
ssh root@192.168.1.20
```

**Saut 2** : depuis l'hôte Proxmox, la VM est joignable.

```bash
ping -c 2 10.0.0.30
ssh admin@10.0.0.30
```

Si ces deux sauts marchent, le rebond marchera.

## Étape 2 — Rebondir en une commande (ponctuel)

L'option `-J` (*jump*) indique par où passer :

```powershell
ssh -J root@192.168.1.20 admin@10.0.0.30
```

SSH te demande d'abord le mot de passe de l'hôte, puis celui de la VM.

## Étape 3 — Rendre ça permanent avec `~/.ssh/config`

Crée ou complète le fichier :

- Windows : `C:\Users\<toi>\.ssh\config` (sans extension)
- Linux : `~/.ssh/config`

```text
# L'hôte de rebond
Host pve01
    HostName 192.168.1.20
    User root

# Les VM, joignables en passant par pve01
Host srv-dc01
    HostName 10.0.0.10
    User admin
    ProxyJump pve01

Host srv-files01
    HostName 10.0.0.20
    User admin
    ProxyJump pve01

Host srv-lb01
    HostName 10.0.0.30
    User admin
    ProxyJump pve01

Host srv-web01
    HostName 10.0.0.31
    User admin
    ProxyJump pve01

Host srv-web02
    HostName 10.0.0.32
    User admin
    ProxyJump pve01
```

Il suffit ensuite de taper :

```powershell
ssh srv-lb01
ssh srv-files01
```

!!! tip "Raccourci : une règle pour tout le réseau interne"
    Au lieu de répéter `ProxyJump pve01`, une seule règle couvre toutes les adresses du réseau interne :
    ```text
    Host 10.0.0.*
        User admin
        ProxyJump pve01
    ```
    Avec cette règle, `ssh 10.0.0.60` passe aussi par le rebond, même pour une VM que tu n'as pas encore déclarée.

## Étape 4 — Ne plus taper de mot de passe (recommandé)

Crée une clé si tu n'en as pas, puis dépose-la sur l'hôte **et** sur chaque VM. Grâce à la config, la commande passe automatiquement par le rebond.

=== "Windows (PowerShell)"
    ```powershell
    ssh-keygen -t ed25519

    # Sur l'hôte de rebond
    type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh pve01 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

    # Sur une VM (à répéter pour chacune)
    type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh srv-lb01 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
    ```

=== "Linux"
    ```bash
    ssh-keygen -t ed25519
    ssh-copy-id pve01
    ssh-copy-id srv-lb01        # passe tout seul par pve01 grâce à la config
    ```

## Étape 5 — Passer root sur la VM

Sur Debian, **root ne peut pas se connecter en SSH avec un mot de passe** (réglage par défaut, et c'est une bonne pratique). On se connecte avec son utilisateur, puis on change d'identité :

```bash
ssh srv-lb01
su -          # mot de passe root
# ou, si l'utilisateur est dans le groupe sudo :
sudo -i
```

---

## ✅ Vérification

```powershell
# La config appliquée à « srv-lb01 » : on doit voir proxyjump pve01
ssh -G srv-lb01 | findstr /i "hostname user proxyjump"      # Windows
ssh -G srv-lb01 | grep -Ei "^(hostname|user|proxyjump)"     # Linux

# La connexion, en mode bavard pour voir les deux sauts
ssh -v srv-lb01
```

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `port 22: Permission denied` **avant** toute demande de mot de passe, en direct | Le PC ne joint pas la VM : souvent une **autre carte réseau** du PC sur la même plage | Passer par le rebond, et voir le dépannage ci-dessous |
| `Connection refused` | La VM répond, mais pas de serveur SSH | Sur la VM : `apt install -y openssh-server` |
| `Connection timed out` pendant le saut vers la VM | L'hôte de rebond ne joint pas la VM | Depuis `pve01` : `ping` de la VM, vérifier son bridge réseau |
| `Could not resolve hostname pve01` | L'alias `pve01` n'existe pas dans la config | Vérifier le bloc `Host pve01` et l'orthographe |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | VM clonée, réinstallée ou IP réattribuée | `ssh-keygen -R 10.0.0.30` puis reconnexion |

## 🔧 Dépannage courant

!!! question "Pourquoi le PC ne joint-il pas la VM directement ?"
    Le plus souvent, une **ancienne carte virtuelle** (ex. VMware *VMnet8*, VirtualBox *Host-Only*) est restée configurée sur la **même plage d'adresses** que le réseau interne : Windows envoie alors le trafic vers cette carte, et non vers le bon chemin. Pour vérifier :
    ```powershell
    Get-NetIPAddress -AddressFamily IPv4 | Select InterfaceAlias, IPAddress
    ```
    Si l'hyperviseur correspondant n'est plus utilisé, désactive la carte : ++win+r++ → `ncpa.cpl` → clic droit → **Désactiver**.

!!! question "Une VM clonée ne répond pas, ou reprend l'IP d'une autre ?"
    Un clone reprend l'IP, le nom et les clés SSH de son modèle. Voir 🛠️ [Préparer une VM clonée depuis un template](../systeme/tuto-preparer-clone-vm.md).

!!! question "`ProxyJump` n'est pas reconnu ?"
    Client OpenSSH trop ancien (avant la version 7.3). Vérifie avec `ssh -V`, et mets à jour.

## Voir aussi

- 🧠 [SSH](concept-ssh.md)
- 🛠️ [Se connecter en SSH à une VM par son nom](tuto-ssh-nom-vm.md) (VM joignables directement)

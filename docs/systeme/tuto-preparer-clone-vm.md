---
icon: material/wrench-outline
title: Préparer une VM clonée depuis un template (Proxmox)
tags: [système, tuto, proxmox, virtualisation, template, clone, debian]
---

# 🛠️ Préparer une VM clonée depuis un template (Proxmox)

!!! abstract "Objectif"
    Créer une nouvelle VM Debian à partir d'un **template**, puis lui donner sa **propre identité** (nom, IP, identifiant machine, clés SSH) pour éviter les conflits avec le modèle et les autres clones.

**Niveau :** 🟢 Bases → 🟡 Intermédiaire · **Durée :** 10 min par VM

## Prérequis

- Un hôte Proxmox avec un **template** Debian (VM convertie en modèle : clic droit → *Convert to template*)
- Le template configuré en IP fixe (ici `10.0.0.200`, nom `tpl-debian01`)

## Pourquoi « personnaliser » un clone ?

Un clone est une **copie exacte** du modèle. Sans retouche, chaque clone a :

| Élément copié | Problème |
|---|---|
| Même **IP** | Conflit d'adresses, une seule VM répond |
| Même **nom d'hôte** | Confusion dans les logs, le DNS, l'annuaire |
| Même **machine-id** | Même bail DHCP, mêmes identifiants système |
| Mêmes **clés d'hôte SSH** | Toutes les VM ont la même identité SSH (faille de sécurité) |

## Le plan d'exemple

| VM | IP |
|---|---|
| `tpl-debian01` (modèle) | `10.0.0.200` |
| `srv-files01` | `10.0.0.20` |
| `srv-lb01` | `10.0.0.30` |
| `srv-web01` | `10.0.0.31` |

Réseau `10.0.0.0/24` · passerelle `10.0.0.1` · DNS `10.0.0.10` (`srv-dc01.lab.local`)

---

## Étape 1 — Cloner dans Proxmox

1. Clic droit sur le template → **Clone**.
2. *Mode* : **Full Clone** (copie indépendante du modèle).
3. *Name* : par exemple `srv-files01`, **sans espace ni parenthèse**.
4. *Target Storage* : le stockage des VM (ex. `local-zfs`).

## Étape 2 — Vérifier la carte réseau

Dans la VM clonée : **Hardware → Network Device** (double-clic) :

- la case **Disconnect** doit être **décochée** ;
- le **Bridge** doit être le même que celui des autres VM du lab (ex. `vmbr2`).

!!! warning "Un clone à la fois"
    Démarre les clones **un par un** : tant qu'ils ne sont pas personnalisés, ils ont tous l'IP du template (`10.0.0.200`).

## Étape 3 — Personnaliser la VM

Démarre la VM, ouvre sa **Console** dans Proxmox et connecte-toi en root :

```bash
# 1. Nom d'hôte
hostnamectl set-hostname srv-files01
sed -i 's/tpl-debian01/srv-files01/g' /etc/hosts

# 2. Adresse IP : modifier la ligne « address »
nano /etc/network/interfaces
```

Exemple du bloc à obtenir dans `/etc/network/interfaces` :

```text
auto ens18
iface ens18 inet static
    address 10.0.0.20/24
    gateway 10.0.0.1
```

```bash
# 3. Nouvel identifiant machine
rm -f /etc/machine-id /var/lib/dbus/machine-id
systemd-machine-id-setup

# 4. Nouvelles clés d'hôte SSH
rm -f /etc/ssh/ssh_host_*
dpkg-reconfigure openssh-server

# 5. Redémarrer pour tout appliquer
reboot
```

---

## ✅ Vérification

```bash
hostname                        # srv-files01
ip a                            # 10.0.0.20/24, interface « UP »
cat /etc/resolv.conf            # nameserver 10.0.0.10
ping -c 2 10.0.0.1            # la passerelle
ping -c 2 srv-dc01.lab.local      # le DNS interne
```

| Ce que tu vois | Ce que ça veut dire | Action |
|---|---|---|
| `NO-CARRIER` / `state DOWN` dans `ip a` | La carte réseau est débranchée | Proxmox : décocher **Disconnect** (étape 2) |
| L'IP est encore `10.0.0.200` | Fichier non modifié ou pas de redémarrage | Revoir `/etc/network/interfaces`, puis `reboot` |
| Passerelle OK, nom DNS KO | Mauvais DNS | Vérifier `nameserver` dans `/etc/resolv.conf` |
| Rien ne répond, même la passerelle | Mauvais bridge | Proxmox : même **Bridge** que les autres VM |

## 🔧 Dépannage courant

!!! question "« REMOTE HOST IDENTIFICATION HAS CHANGED » en me connectant en SSH ?"
    C'est normal après l'étape 3 : les clés d'hôte ont changé. Sur ton PC : `ssh-keygen -R 10.0.0.20`.

!!! question "Deux VM se « volent » la même adresse ?"
    Un clone a été démarré avant que le précédent soit personnalisé. Éteins-en un, corrige l'IP de l'autre, puis redémarre.

!!! question "Le nom de l'interface n'est pas `ens18` ?"
    Relève le bon nom avec `ip a` et utilise-le dans `/etc/network/interfaces`.

## Voir aussi

- 🛠️ [Se connecter en SSH aux VM d'un réseau interne (rebond)](../reseau/tuto-ssh-rebond.md)

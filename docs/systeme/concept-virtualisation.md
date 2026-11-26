---
icon: material/book-open-variant
title: Virtualisation et hyperviseurs
tags: [système, bases, concept, virtualisation, proxmox, hyperviseur, lxc, template, clone]
---

# 🧠 Virtualisation et hyperviseurs

!!! abstract "En une phrase"
    La **virtualisation** fait tourner plusieurs ordinateurs « virtuels » (des **VM**) sur une seule machine physique, grâce à un logiciel chef d'orchestre : l'**hyperviseur**.

**Niveau :** 🟢 Bases

## L'analogie

Un serveur physique, c'est un **immeuble**. L'hyperviseur est le **syndic** : il découpe l'immeuble en appartements (les VM), donne à chacun sa part d'eau et d'électricité (CPU, RAM, disque, réseau) et s'assure que les voisins ne se marchent pas dessus. Chaque locataire se croit seul chez lui.

## Les mots à connaître

| Mot | Ce que ça désigne |
|---|---|
| **Hôte** | La machine physique (ex. le serveur Proxmox `pve`) |
| **Invité** / **VM** | Le système qui tourne dans la machine virtuelle (ex. une Debian) |
| **Hyperviseur** | Le logiciel qui crée les VM et leur partage le matériel |
| **vCPU, vRAM, disque virtuel** | Les ressources « prêtées » à la VM, prises sur celles de l'hôte |
| **Bridge** (ex. `vmbr0`) | Le switch virtuel qui relie les VM entre elles et au réseau |
| **Snapshot** | Une photo de la VM à un instant T, pour revenir en arrière |

## Type 1 ou type 2 ?

| | **Type 1** (« bare metal ») | **Type 2** (« hébergé ») |
|---|---|---|
| S'installe | Directement sur le matériel | Comme une application, sur Windows/Linux/macOS |
| Performances | Excellentes (pas d'OS intermédiaire) | Correctes, l'OS hôte consomme sa part |
| Usage | Serveurs, datacenter, **homelab** | Poste de travail, tests, cours |
| Exemples | **Proxmox VE** (KVM), VMware ESXi, Hyper-V, XCP-ng | VirtualBox, VMware Workstation |

## VM ou conteneur LXC ?

Proxmox sait faire les deux. La différence tient en une question : **qui fournit le noyau ?**

| | **VM** (KVM) | **Conteneur LXC** |
|---|---|---|
| Noyau | Le sien (OS complet) | **Partagé avec l'hôte** |
| Systèmes possibles | Linux, Windows, pfSense, BSD… | **Linux uniquement** |
| Démarrage | ~30 s | ~2 s |
| Poids | Lourd (RAM et disque réservés) | Très léger |
| Isolation | Forte | Plus faible (même noyau) |
| Idéal pour | Windows Server / AD, pare-feu, tout ce qui touche au noyau | Petits services Linux : web, DNS, supervision |

!!! tip "Règle simple"
    Windows ou besoin d'un noyau à soi → **VM**. Petit service Linux → **LXC**.

## Template et clones

Un **template** est une VM « modèle » figée : déjà installée et configurée, on ne la démarre plus, on s'en sert seulement pour en fabriquer d'autres. Dans Proxmox : clic droit sur la VM → *Convert to template*.

À partir d'un template, on crée des **clones**, de deux façons :

| | **Clone complet** (*Full Clone*) | **Clone lié** (*Linked Clone*) |
|---|---|---|
| Disque | Copie intégrale du disque du modèle | Ne stocke que les **différences** avec le modèle |
| Création | Lente (copie complète) | Quasi instantanée |
| Espace disque | Autant que le modèle | Très peu au départ |
| Dépend du template ? | **Non**, VM autonome | **Oui** : le template ne peut plus être supprimé |
| Idéal pour | VM durables, production | Labs jetables, tests rapides |

!!! info "Un clone reste une copie"
    Complet ou lié, le clone démarre avec le **même nom, la même IP, le même machine-id et les mêmes clés SSH** que le modèle. Il faut le personnaliser : c'est tout l'objet du [tuto associé](tuto-preparer-clone-vm.md).

## À quoi ça sert concrètement

- Monter un **lab complet** (AD, pare-feu, serveurs Linux) sur un seul PC ou serveur.
- **Tester sans risque** : un snapshot avant la manip, un retour arrière si ça casse.
- Déployer une nouvelle VM en quelques minutes grâce aux **templates**.
- Mieux utiliser le matériel : un serveur physique héberge des dizaines de services.

!!! warning "Pièges fréquents"
    - **Confondre l'hôte et la VM** : une commande tapée dans le *Shell* du nœud `pve` modifie l'hyperviseur, pas la VM. Vérifie toujours l'invite (`root@pve` ≠ `root@srv-fic`).
    - **Démarrer plusieurs clones non personnalisés** en même temps : ils ont tous la même IP.
    - **Supprimer un template** utilisé par des clones liés : Proxmox refuse, et c'est tant mieux.
    - **Un snapshot n'est pas une sauvegarde** : il vit sur le même disque que la VM. Si le disque meurt, tout part avec.
    - **Allouer plus de RAM que l'hôte n'en a** : les VM ralentissent ou se font tuer par l'hôte.
    - Oublier la **virtualisation matérielle** (Intel VT-x / AMD-V) dans le BIOS : l'hyperviseur refuse de lancer les VM ou les rend très lentes.

## Pour aller plus loin

- 🛠️ **[Préparer une VM clonée depuis un template (Proxmox)](tuto-preparer-clone-vm.md)** : la mise en pratique
- 🛠️ [SSH par rebond vers un réseau interne](../reseau/tuto-ssh-rebond.md) : joindre les VM d'un réseau isolé

---
icon: material/book-open-variant
title: Le réseau dans Azure (VNet, sous-réseaux, NSG, appairage)
tags: [cloud, intermédiaire, concept, azure, vnet, nsg, peering, hub and spoke, nat]
---

# 🧠 Le réseau dans Azure : VNet, NSG et appairage

!!! abstract "En une phrase"
    Dans Azure, chaque VM vit dans un **réseau virtuel (VNet)** découpé en **sous-réseaux**, filtré par des **NSG** (pare-feu à règles numérotées) et relié à d'autres VNet par **appairage** ; tout est logiciel, il n'y a ni switch ni câble à gérer.

## L'analogie

Un campus d'entreprise :

- le **VNet** = le terrain clôturé du campus, avec sa plage d'adresses ;
- les **sous-réseaux** = les bâtiments (accueil, bureaux, salle serveurs) ;
- le **NSG** = le badge à l'entrée de chaque bâtiment, avec la liste de qui a le droit d'entrer ;
- l'**appairage** = une passerelle couverte entre deux campus voisins ;
- l'**IP publique** = la porte qui donne sur la rue.

---

## Les briques à connaître

| Brique | Ce que c'est | Équivalent dans un homelab Proxmox |
|---|---|---|
| **VNet** | Réseau privé isolé, avec un espace d'adresses (ex. `10.20.0.0/16`) | Un pont `vmbr` avec son plan d'adressage |
| **Sous-réseau** | Une tranche du VNet (ex. `10.20.1.0/24`) | Un VLAN |
| **Carte réseau (NIC)** | L'interface de la VM, avec une IP privée et éventuellement une IP publique | La carte `net0` de la VM |
| **NSG** | Liste de règles autoriser/refuser, posée sur un sous-réseau ou une carte | Pare-feu Proxmox ou pfSense |
| **IP publique** | Adresse joignable depuis Internet, rattachée à une carte ou un service | La redirection de port sur la box |
| **Table de routage** (UDR) | Routes personnalisées qui remplacent celles d'Azure | Les routes statiques de pfSense |
| **Appairage** (*peering*) | Lien privé entre deux VNet, sans passer par Internet | Un routeur entre deux réseaux |

### Ce qu'Azure fait à ta place

- **Le DHCP** : l'IP privée est attribuée par Azure. On la fixe **dans Azure** (*statique* sur la carte), **jamais dans l'OS**, sinon la VM perd le réseau.
- **Le routage** entre sous-réseaux d'un même VNet : automatique, sans rien configurer.
- **5 adresses réservées** par sous-réseau : la première, les trois suivantes (passerelle et DNS d'Azure) et la dernière. Un `/24` offre donc 251 adresses utilisables, un `/29` seulement 3.

## Les NSG : comment une règle est choisie

Chaque règle a une **priorité** (de 100 à 4096) : la plus **petite** est lue en premier, et la première qui correspond **décide**. Les autres ne sont pas lues.

| Priorité | Nom | Source | Port | Action |
|---|---|---|---|---|
| 100 | `ssh-admin` | `203.0.113.25/32` (IP de l'admin) | 22 | ✅ Autoriser |
| 200 | `web` | `Internet` | 80, 443 | ✅ Autoriser |
| 65000 | `AllowVnetInBound` *(par défaut)* | `VirtualNetwork` | tous | ✅ Autoriser |
| 65001 | `AllowAzureLoadBalancerInBound` *(par défaut)* | `AzureLoadBalancer` | tous | ✅ Autoriser |
| 65500 | `DenyAllInBound` *(par défaut)* | tous | tous | ❌ Refuser |

- Les NSG sont **à état** (*stateful*) : la réponse à une connexion autorisée repasse toute seule, inutile d'écrire la règle retour.
- Un NSG sur le **sous-réseau** et un autre sur la **carte** : le trafic entrant doit être autorisé **par les deux**.
- Des **étiquettes de service** (`Internet`, `VirtualNetwork`, `Storage`…) évitent de saisir des listes d'adresses.

## Sortir sur Internet

Depuis 2026, les sous-réseaux des **nouveaux** VNet sont **privés par défaut** : une VM sans IP publique **n'a pas Internet** (pas de `apt update`). Quatre façons de lui en donner :

| Solution | Coût | Quand l'utiliser |
|---|---|---|
| IP publique sur la VM | Faible | Une VM de test isolée |
| **NAT Gateway** sur le sous-réseau | Moyen, à l'heure | Toute une ferme de VM privées |
| Règle sortante d'un Load Balancer | Inclus dans le Load Balancer | VM déjà derrière un répartiteur |
| VM passerelle (NVA) + table de routage | Le prix d'une petite VM | Filtrer ou journaliser la sortie, comme un pfSense |

## Relier des VNet : le hub & spoke

```
              ┌──────────── vnet-hub 10.20.0.0/16 ─────────────┐
              │ pare-feu ou VM passerelle · VPN · bastion       │
              └───────▲───────────────────────────▲─────────────┘
           appairage  │                           │  appairage
   ┌──────────────────┴──────┐       ┌────────────┴────────────┐
   │ vnet-prod 10.21.0.0/16  │       │ vnet-test 10.22.0.0/16  │
   └─────────────────────────┘       └─────────────────────────┘
```

- Le **hub** porte les services partagés ; chaque **spoke** porte une application ou un environnement.
- L'appairage **n'est pas transitif** : `prod` et `test` ne se voient pas à travers le hub. Il faut soit les appairer entre eux, soit faire passer le trafic par une passerelle du hub avec une table de routage.
- Deux VNet appairés ne doivent **jamais** avoir des plages qui se chevauchent : on prévoit le plan d'adressage avant de créer le premier.

---

## À quoi ça sert concrètement

- **Isoler** un serveur web (sous-réseau exposé) de sa base de données (sous-réseau privé), comme une DMZ.
- **Administrer sans exposer** : une seule machine joignable en SSH, tout le reste en privé, accessible par rebond.
- **Séparer prod, test et dev** dans des VNet distincts, reliés au même hub.
- **Prolonger le réseau de l'entreprise** dans Azure par un VPN site-à-site arrivant sur le hub.

!!! warning "Pièges fréquents"
    - `az vm create` crée par défaut un NSG qui ouvre **SSH (ou RDP) à tout Internet**. Le désactiver (`--nsg-rule NONE`) et écrire ses propres règles limitées à ton IP.
    - Fixer l'IP dans `netplan` ou dans Windows : la VM devient injoignable. Toujours la fixer côté Azure.
    - Choisir un espace d'adresses qui chevauche celui du réseau de l'entreprise ou d'un autre VNet : l'appairage et le VPN deviennent impossibles.
    - Oublier que le NSG du sous-réseau **et** celui de la carte s'appliquent : une règle ouverte sur l'un ne suffit pas.
    - Penser qu'une VM privée peut faire `apt update` : sur un VNet récent, il lui faut une sortie explicite.

## Pour aller plus loin

- 🛠️ **[Monter un VNet avec une VM publique et une VM privée](tuto-vnet-nsg-azure.md)** : la mise en pratique (sous-réseaux, NSG, rebond SSH, NAT Gateway, Network Watcher)
- 🧠 [Adressage IP et masque](../reseau/concept-adressage-ip.md) : préparer son plan d'adressage
- 🧠 [Pare-feu et NAT](../reseau/concept-pare-feu-nat.md) : les mêmes principes sur pfSense
- 🛠️ [SSH par rebond](../reseau/tuto-ssh-rebond.md) : la technique d'accès aux VM privées

??? quote "Sources"
    - [Microsoft Learn : réseau virtuel Azure](https://learn.microsoft.com/fr-fr/azure/virtual-network/virtual-networks-overview)
    - [Microsoft Learn : groupes de sécurité réseau (NSG)](https://learn.microsoft.com/fr-fr/azure/virtual-network/network-security-groups-overview)
    - [Microsoft Learn : appairage de réseaux virtuels](https://learn.microsoft.com/fr-fr/azure/virtual-network/virtual-network-peering-overview)
